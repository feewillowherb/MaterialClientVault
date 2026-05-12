# 04 - XiaoShanServe 接收与转发链路

## 概述

`FdSoft.MaterialSys.Gov.XiaoShanServe` 是一个 ASP.NET Core Web 服务，扮演**中转层**角色：接收 GovClient 上报的称重数据，存储到本地 SQLite，再通过后台服务转发到真正的政府平台 API。

## 技术栈

| 项 | 值 |
|----|-----|
| 框架 | ASP.NET Core |
| ORM | SqlSugar |
| 数据库 | SQLite (XiaoShan.db) |
| 配置 | appsettings.json |

## 数据接收端点

### ApiController.Post

**路由**: `POST /Api/Post`

### 接收流程

```
POST /Api/Post (JSON body: mGovRequestWeight)
    │
    ├── 验证接入码
    │       查询 buildLicenseNo 或 fdBuildLicenseNo
    │       匹配 Gov_Project 表
    │
    ├── 处理图片
    │       ├── 解析 Base64 图片数组
    │       ├── 保存到本地文件系统 (按 buildLicenseNo 分类)
    │       └── 超过 200KB 自动压缩 (ImageQuality = 60L)
    │
    ├── 构建 GovSyncData 实体
    │       ├── CarNo, goodsWeight, snapTime
    │       ├── BuildLicenseNo
    │       ├── sourceData = JSON 序列化原始请求
    │       ├── snapImages = 本地图片路径
    │       ├── SyncType = 0 (待同步)
    │       └── SyncNumber = 0
    │
    └── 存入 SQLite (SqlSugar)
```

### mGovRequestWeight 接收模型

| 字段 | 类型 | 说明 |
|------|------|------|
| `carNo` | string | 车牌号 |
| `carColor` | string | 车身颜色 |
| `carNoColor` | string | 车牌颜色（黄/蓝） |
| `buildLicenseNo` | string | 场地标识 |
| `fdBuildLicenseNo` | string | 凡东接入码（MD5） |
| `goodsWeight` | string | 货物重量 |
| `snapTime` | string | 称重时间 |
| `snapImages` | string[] | Base64 图片数组 |
| `carType` | string | 车型（大车/小车） |
| `deviceID` | string | 设备编号（01进/02出） |
| `siteType` | string | 场地类型（1=工地） |

## 数据库模型

### Gov_Project（项目表）

| 字段 | 说明 |
|------|------|
| ProId | 项目 ID |
| ProName | 项目名称 |
| BuildLicenseNo | 场地编码 |
| FdBuildLicenseNo | 凡东接入码（MD5 加密） |
| SyncStatus | 同步状态 |
| DeleteStatus | 删除状态 |

### Gov_SyncData（同步数据表）

| 字段 | 说明 |
|------|------|
| CarNo | 车牌号 |
| goodsWeight | 货物重量 |
| snapTime | 抓拍时间 |
| BuildLicenseNo | 场地标识 |
| sourceData | 原始请求 JSON |
| snapImages | 本地图片路径 |
| SyncType | 同步状态 (0=待同步, 1=成功, 2=失败) |
| SyncNumber | 同步次数 |

### Gov_Log（同步日志表）

| 字段 | 说明 |
|------|------|
| SyncId | 同步数据 ID |
| SyncTime | 同步时间 |
| SyncResult | 同步结果 |
| SyncCode | 结果码 |
| SyncMsg | 结果信息 |

## 数据转发到政府平台

### ExplortStatisticBgService

**触发**: 后台定时服务，每 5 秒执行一次。

### 转发流程

```
每 5 秒执行
    │
    ├── 查询待同步数据
    │       SyncType != 1 (成功) AND SyncNumber < 5
    │
    └── foreach (record):
            │
            ├── 读取本地图片 → 转 Base64
            │
            ├── 构建 mGovRequestWeight
            │
            ├── HTTP POST → http://191.12.15.58:8899/sapi/v1/inoutRecord/save
            │
            ├── 成功:
            │       SyncType = 1, SyncNumber += 1
            │       写入 Gov_Log
            │
            └── 失败:
                    SyncType = 2, SyncNumber += 1 (最大 5 次)
                    SyncNumber >= 10 → 标记为彻底失败
                    写入 Gov_Log
```

### 最终政府 API 端点

```
POST http://191.12.15.58:8899/sapi/v1/inoutRecord/save
Content-Type: application/json
Body: mGovRequestWeight (JSON)
Response: mGovResponseBase<string> { code: 200/其他, msg: "...", data: "..." }
```

## 管理功能

### 项目管理 (ProjectController)

| 端点 | 说明 |
|------|------|
| `GET /Project/PageList` | 项目列表 |
| `GET /Project/Add?proId=` | 新增/编辑项目 |
| `POST /Project/Add` | 提交项目信息 |
| `GET /Project/SetStatus?proId=` | 设置同步状态 |
| `GET /Project/Del?proId=` | 删除项目 |

### 同步信息查看 (SyncInfoController)

| 端点 | 说明 |
|------|------|
| `GET /SyncInfo/PageList` | 同步数据列表 |
| `GET /SyncInfo/logList?id=` | 同步日志列表 |

## 数据链路完整图

```
[地磅设备]
    │ 串口
    ▼
[DiBangScaleReceiver] ── DataReceived ──→ [Form1._weight]
                                              │
[摄像头] ── 车牌识别 ──→ [CaptureDevice.result]
                                              │
                                    [weightTimer 2s]
                                              │
                                    weightMaxCount == 0?
                                              │ 是
                                    UpLoadWeightDate()
                                              │
                                    bRePushWeight.Add()
                                              │
                                    ┌─── SQLite (本地) ───┐
                                    │                      │
                                    └──────────────────────┘
                                              │
                              [agentTmer_Main 5s 轮询]
                                              │
                                    HTTP POST /post
                                              │
                                    ┌──────────────────────┐
                                    │                      │
[政府平台API] ←── HTTP POST ── [XiaoShanServe]
  191.12.15.58:8899     ExplortStatisticBgService
                       (5s 转发)
```
