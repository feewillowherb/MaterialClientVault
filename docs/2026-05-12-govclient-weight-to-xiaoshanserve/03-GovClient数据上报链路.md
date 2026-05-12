# 03 - GovClient 数据上报链路

## 概述

`Fdsoft.Weight.GovClient` 的数据上报采用**本地 SQLite 暂存 + 定时轮询上报**的模式。重量稳定后数据先写入本地 SQLite，然后由 `ServicePushWeightToGovApi` 定时查询并上报到 XiaoShanServe。

## 数据入库流程

### 触发条件

`weightMaxCount == 0` 时（重量稳定 10 秒后），调用 `UpLoadWeightDate()`。

### UpLoadWeightDate() 方法

**文件**: `Form1.cs:589-633`

```
UpLoadWeightDate(carNo, carNoColor, grossWeight, imgList)
    │
    ├── 构建 mRePushWeight 模型
    │       ├── CarNo = carNo[1:]              // 去掉颜色前缀
    │       ├── CarNoColor = licenseColor
    │       ├── BuildLicenseNo = RSA授权码
    │       ├── InOutType = 进场地称重(0)
    │       ├── EquipmentNumber = 配置值
    │       ├── EquipmentType = 配置值
    │       ├── GrossWeight = grossWeight
    │       ├── TareWeight = 0
    │       ├── SnapTime = DateTime.Now
    │       ├── SnapImages = 图片路径逗号拼接
    │       └── SyncStatus = 未同步
    │
    ├── bRePushWeight.Add(model)    // 写入 SQLite
    │
    ├── LED 设备发送（如启用）
    │
    └── return true
```

### 关键设计特点

1. **始终为"进场地称重"**：`InOutType` 固定为 `0`（进场），无出场地称重概念。
2. **皮重固定为 0**：`TareWeight` 始终为 0，不做皮重处理。
3. **图片为本地路径**：`SnapImages` 存储逗号分隔的本地文件路径，上报时再读取转 Base64。
4. **无运单概念**：每次称重独立记录，不做进出场配对。

## 定时上报流程

### ServicePushWeightToGovApi

**文件**: `Services/ServicePushWeightToGovApi.cs`

### 启动配置

```csharp
// 根据 networkCompany 配置选择目标平台
switch (networkCom) {
    case 电信:
        govWightApiUrl = govWightApiUrlDianXin;  // 心跳关闭
        MsgAction = "";
        break;
    case 联通:
        govWightApiUrl = govWightApiUrlLianTong;  // 心跳开启
        MsgAction = "upload";
        break;
    case 萧山1:
        govWightApiUrl = govXiaoShan;             // 萧山模式
        MsgAction = "post";
        break;
}
```

### 主定时器逻辑 (5 秒间隔)

```
agentTmer_Main_Elapsed() 每 5 秒执行
    │
    ├── 检查 GovApiOnline（网络连通性）
    │
    ├── 查询 SQLite: SyncStatus != 不再上传 AND SyncStatus != 成功
    │       最多 100 条，按 Id 升序
    │
    └── foreach (model in list):
            │
            ├── 验证 RSA 授权时间（过期则终止）
            │
            ├── 读取本地图片 → 转 Base64
            │       GetImages(model): 读取 SnapImages 路径，File.ReadAll → Convert.ToBase64
            │
            ├── 构建 mGovRequestWeight（根据运营商区分字段）
            │       萧山1: carType, deviceID, siteType, goodsWeight
            │       联通: equipmentNumber, equipmentType, grossWeight, tareWeight
            │
            ├── HTTP POST → {govWightApiUrl}/{MsgAction}
            │       Post(url, JsonConvert.SerializeObject(requestModel))
            │
            ├── 解析 mGovResponseBase<string>
            │       code == 200 → 成功
            │       其他 → 失败
            │
            ├── 更新 model: SyncStatus, FailReason, FailMsg
            │
            ├── SaveToSqlLite(model)
            │       FailCount += 1
            │       if (FailCount >= 9 && FailReason != 网络) → SyncStatus = 不再上传
            │
            └── 回调 SendGovApiResultEventCallBack → 更新 UI
```

### mGovRequestWeight 模型（上报格式）

| 字段 | 萧山1 模式 | 联通模式 | 说明 |
|------|-----------|----------|------|
| `carNo` | ✅ | ✅ | 车牌号 |
| `carNoColor` | ✅ | ✅ | 车牌颜色 |
| `buildLicenseNo` | ✅ | ✅ | 场地标识 |
| `snapTime` | ✅ | ✅ | 称重时间 |
| `snapImages` | ✅ (Base64[]) | ✅ (Base64[]) | 图片 |
| `carType` | ✅ (>4.5T=大车) | — | 车型 |
| `deviceID` | ✅ (01进/02出) | — | 设备编号 |
| `siteType` | ✅ ("1"=工地) | — | 场地类型 |
| `goodsWeight` | ✅ (字符串) | — | 货物重量 |
| `equipmentNumber` | — | ✅ | 设备编号 |
| `equipmentType` | — | ✅ | 设备型号 |
| `grossWeight` | — | ✅ (int) | 毛重 |
| `tareWeight` | — | ✅ (int) | 皮重 |

## 重试机制

| 参数 | 值 | 说明 |
|------|-----|------|
| `MaxFailCount` | 9 | 最大失败次数 |
| 轮询间隔 | 5 秒 | `agentTmer_Main` |
| 失败后行为 | `FailCount += 1`，下次轮询重试 | 最多重试 9 次 |
| 放弃条件 | `FailCount >= 9` 且 `FailReason != 网络` | 标记为"不再上传" |
| 网络异常 | 不计入 FailCount，下次继续尝试 | 区分网络问题和其他错误 |

## 网络连通性检测

### ServiceTelnet

通过 Telnet 连接目标服务器判断网络是否可用，回调通知 `ServicePushWeightToGovApi`：

```
ServiceTelnet.Start()
    → 检测网络 → 回调 IsOnelineEventCallBack(yes/no)
    → ServicePushWeightToGovApi.GovApiOnline = yes/no
    → GovApiOnline == false 时跳过上报
```
