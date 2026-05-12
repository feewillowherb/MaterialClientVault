# 02 - MaterialClient 现有能力对照

> 目标项目：`repos/MaterialClient/`

## 1. 可直接复用的能力

### 1.1 海康威视车牌识别

**文件**: `MaterialClient.Common/Services/Hikvision/HikvisionLprService.cs`

| 能力 | 状态 | 说明 |
|------|------|------|
| SDK 初始化/清理 | 已实现 | `NET_DVR_Init` / `NET_DVR_Cleanup`，带双重检查锁 |
| 监听启停 | 已实现 | `NET_DVR_StartListen_V30` / `NET_DVR_StopListen_V30`，正确处理 GCHandle |
| 被动识别回调 | 已实现 | `COMM_UPLOAD_PLATE_RESULT` + `COMM_ITS_PLATE_RESULT` |
| 主动抓拍 | 已实现 | `NET_DVR_ContinuousShoot`，支持会话缓存 |
| 多设备管理 | 已实现 | `ConcurrentDictionary<ip, config>` |
| 事件发布 | 已实现 | 通过 `ILocalEventBus` 发布 `LicensePlateRecognizedEventData` |
| 设备在线检查 | 已实现 | `TryLogin` + `NET_DVR_Logout` |
| 错误处理 | 已实现 | 中文错误描述映射 |

**差距**：回调中**未保存图片到本地文件系统**。当前只发布车牌号事件，不处理图片数据。移植时需要：
- 在 `HandlePlateResult` / `HandleItsPlateResult` 中提取图片字节数据
- 保存到本地文件系统（复用 GovClient 的路径格式或 MaterialClient 的存储策略）
- 将图片路径与称重记录关联

### 1.2 海康威视 SDK 集成

**目录**: `MaterialClient.Common/HCNetSDK/`

| 资源 | 状态 |
|------|------|
| HCNetSDK.dll | 已集成 |
| HCNetSDK.lib | 已集成 |
| HCNetSDKCom/ | 已集成 |
| P/Invoke 封装 | 已实现（`HikvisionSdk.cs`） |
| 编码工具 | 已实现（`HikvisionEncodingHelper`） |

### 1.3 重量数据采集

**文件**: `MaterialClient.Common/Services/` 下存在 `TruckScaleWeightService`（由已有调研文档提及）

| 能力 | 状态 | 说明 |
|------|------|------|
| 串口通信 | 已有对应 | MaterialClient 已有地磅相关服务 |
| 稳定性检测 | 已有对应 | 使用时间窗口分析（3000ms），优于 GovClient 的计数器方案 |

### 1.4 附件管理体系（可直接复用）

**核心文件**:
- `MaterialClient.Common/Entities/AttachmentFile.cs` — 附件文件实体
- `MaterialClient.Common/Entities/Enums/AttachType.cs` — 附件类型枚举
- `MaterialClient.Common/Services/AttachmentService.cs` — 附件 CRUD + OSS 同步
- `MaterialClient.Common/Utils/PathManager.cs` — 相对/绝对路径转换

**AttachmentFile 实体字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `Id` | int | 主键 |
| `FileName` | string | 文件名 |
| `LocalPath` | string | 本地相对路径（数据库存储相对路径，操作时通过 PathManager 转绝对路径） |
| `OssFullPath` | string? | OSS 完整路径（上传后填充） |
| `AttachType` | AttachType (short) | 附件类型枚举 |
| `LastSyncTime` | DateTime? | 最后同步到 OSS 的时间 |
| 审计字段 | — | Creator/CreateDate/LastEditor/UpdateDate/IsDeleted 等 |

**AttachType 当前枚举值**:

| 值 | 名称 | 说明 |
|----|------|------|
| 0 | `UnmatchedEntryPhoto` | 未匹配榜单照片 |
| 1 | `EntryPhoto` | 进场照片 |
| 2 | `ExitPhoto` | 出场照片 |
| 3 | `TicketPhoto` | 票据照片 |
| **4（新增）** | **`LprCapturePhoto`** | **LRP 车牌识别抓拍照片** |

**AttachmentService 已有能力**:

| 能力 | 方法 | 说明 |
|------|------|------|
| 创建附件 | `CreateOrReplaceBillPhotoAsync()` | 保存文件 + 创建 AttachmentFile 实体 + 关联 junction table |
| 路径管理 | `PathManager.ToRelativePath()` / `ToAbsolutePath()` | 数据库存相对路径保证可移植性 |
| OSS 上传 | `SyncWaybillAttachmentsToOssAsync()` / `SyncPendingAttachmentsToOssAsync()` | 自动上传到阿里云 OSS |
| 服务端同步 | `UploadAttachmentInfoToServerAsync()` | 通过 `IMaterialPlatformApi.UpdateAttachesAsync()` 同步附件信息 |
| 删除/替换 | `CreateOrReplaceBillPhotoAsync()` 内置 | 先删旧附件再创建新附件 |

**关联表**（junction table）:
- `WeighingRecordAttachment` — 称重记录 ↔ AttachmentFile（多对多）
- `WaybillAttachment` — 运单 ↔ AttachmentFile（多对多）

**对 LRP 图片的适用性分析**:

LRP 图片与现有附件（进场/出场/票据照片）在数据结构上完全一致：都需要保存本地文件 + 记录路径 + 后续可能同步。因此：

- **可直接复用** `AttachmentFile` 实体存储 LRP 图片
- **仅需新增** `LprCapturePhoto = 4` 枚举值
- **无需新建** junction table — LRP 图片通过 `WeightSyncRecord` 中的 `AttachmentFileId` 列表关联（或复用 `WeighingRecordAttachment`）
- **自动获得** OSS 同步、审计跟踪、路径管理等已有能力

### 1.5 配置管理

**文件**: `MaterialClient.Common/Configuration/`

| 能力 | 状态 | 说明 |
|------|------|------|
| 海康威视配置 | 已实现 | `HikvisionLprDefaults.cs`、`LicensePlateRecognitionConfig` |
| 系统设置服务 | 已实现 | `ISettingsService`，支持热更新 |

## 2. 需要新增的能力

### 2.1 政府平台同步服务（新增）

MaterialClient 中**不存在**任何政府平台同步功能。需要新建：

| 组件 | 说明 | 参考 |
|------|------|------|
| 同步服务 | 定时扫描未同步记录，POST 到萧山 API | `ServicePushWeightToGovApi.cs` |
| 重推服务 | 失败记录的重试逻辑 | `ServiceRePushWeightToGovApi.cs` |
| 网络检测 | 检测政府 API 是否可达 | `ServiceTelnet.cs` |

### 2.2 称重记录持久化（新增/扩展）

| 组件 | 说明 |
|------|------|
| 称重记录实体 | 对应 `mRePushWeight`，需包含同步状态字段 |
| 数据库表 | 在现有 EF Core 上下文中添加 |
| 数据访问层 | 查询未同步记录、更新同步状态 |

### 2.3 图片保存机制（复用 AttachmentFile + 新增 AttachType）

| 组件 | 说明 | 复用/新增 |
|------|------|-----------|
| `AttachType.LprCapturePhoto = 4` | 新增枚举值，标识 LRP 抓拍照片 | **新增** |
| LRP 回调图片保存 | 在 `HikvisionLprService` 回调中提取图片 → 保存到本地 → 创建 `AttachmentFile` 实体 | **修改**（复用 AttachmentService 模式） |
| 图片压缩 | 从 `CaptureDevice.CompressImage` 移植压缩逻辑（可选） | **新增** |
| 图片路径关联 | 通过 `AttachmentFile.LocalPath` 存储相对路径，替代 `SnapImages` 逗号分隔方案 | **复用** |
| 同步时图片读取 | `PathManager.ToAbsolutePath()` + `File.ReadAllBytes()` + Base64 | **复用** |

**与原 GovClient 方案的对比**：

| 维度 | GovClient（原方案） | MaterialClient（新方案） |
|------|---------------------|-------------------------|
| 图片路径存储 | `SnapImages` 逗号分隔字符串字段 | `AttachmentFile.LocalPath` 相对路径 + 关联表 |
| 路径格式 | 绝对路径（`Environment.CurrentDirectory + "\\" + item`） | 相对路径（`PathManager.ToRelativePath()`），操作时转绝对 |
| 数据完整性 | 无外键约束 | 通过 `AttachmentFile` 实体 + 关联表保证 |
| OSS 同步 | 不支持 | 自动获得（`OssFullPath` + `LastSyncTime`） |
| 审计跟踪 | 无 | 自动获得（Creator/CreateDate 等） |

### 2.4 地磅串口通信（需确认）

MaterialClient 是否已支持 GovClient 使用的特定协议（耀华TF0、嘉兴丹柯等），需要确认。如果不支持，需要从 `WeightDevice.cs` 或 `DiBangScaleReceiver.cs` 移植协议解析逻辑。

## 3. 在线状态能力（扩展需求相关）

### 3.1 GovClient 的在线检测机制

| 组件 | 文件 | 机制 |
|------|------|------|
| 网络连通性 | `Services/ServiceTelnet.cs` | 1 秒定时器，Telnet TCP 连接测试政府 API 端口 |
| 设备在线状态 | `Form1.cs` | 3 秒定时器维护 `videoOnline`/`weightOnline`/`ledOnline` 三个状态变量 |
| 网络状态回调 | `ServicePushWeightToGovApi.ServiceClass_IsOnelineEventHandler` | 同步服务订阅网络状态，离线时暂停上报 |

**关键特征**：纯本地检测，**不向服务端上报**在线状态。服务端无法感知客户端是否在线。

### 3.2 MaterialClient 的现有在线检测能力

| 组件 | 文件 | 机制 |
|------|------|------|
| 统一设备在线查询 | `Services/LprDeviceOnlineStatusService.cs` | `IsOnline(deviceType, config)` 统一入口 |
| 海康威视在线检查 | `Services/Hikvision/HikvisionLprService.cs` | `TryLogin` + `NET_DVR_Logout` |
| 华夏智信心跳记录 | `Services/Huaxiazhixin/HuaxiazhixinLprService.cs` | `RecordLastSeen(ip)` + 超时判断（默认 30 秒） |
| 臻识在线检查 | `Services/Vzvision/VzvisionLprService.cs` | `VzLPRClient_IsConnected` SDK API |
| Web 回调端点 | `Services/MinimalWebHostService.cs` | 提供 `/api/CarLicense/CallDeviceMessageHuaXiaZhiXing` 接收设备 heartbeat 回调 |

**关键特征**：MaterialClient 已有完善的**本地**设备在线检测，但同样**不向服务端上报**客户端整体在线状态和设备状态汇总。

### 3.3 差距分析

服务端要知道"客户端在线 + 设备在线"，当前两端都缺失以下能力：

| 缺失能力 | 说明 |
|---------|------|
| 客户端→服务端心跳上报 | 定期向服务端发送自身在线状态 |
| 设备状态汇总上报 | 将各设备（地磅、相机）的在线状态一并发送 |
| 服务端在线状态存储 | 服务端记录每个客户端的最后心跳时间 |
| 服务端状态查询 API | 供管理后台查询客户端/设备在线情况 |
| XiaoShanServe 端点 | 当前 XiaoShanServe 没有接收在线状态的端点 |

## 4. 不需要移植的部分

| 功能 | 原因 |
|------|------|
| 电信/联通同步模式 | 仅移植萧山1 |
| 心跳包机制 | 萧山1 不需要心跳 |
| RSA 授权检查 | 授权逻辑与 MaterialClient 的授权机制不同，需另行设计 |
| 臻识科技支持 | MaterialClient 当前仅集成海康威视 |
| WinForms UI | MaterialClient 使用 Avalonia |
| LED 设备控制 | 不在本次移植范围 |
| 实时视频预览 | 已有调研文档明确标记为范围外 |

## 5. 架构差异对照

| 维度 | GovClient | MaterialClient |
|------|-----------|----------------|
| **UI 框架** | WinForms | Avalonia |
| **DI 容器** | 无（手动 new） | ABP / Microsoft DI |
| **事件机制** | C# event / delegate | ILocalEventBus |
| **配置** | App.config / ConfigurationManager | ISettingsService / appsettings.json |
| **数据库** | SQLite + CodeSmith 生成 DAL | EF Core + SQLite |
| **日志** | log4net | Microsoft.Extensions.Logging |
| **异步模式** | Timer + Monitor 锁 | async/await + Rx.NET |
| **HTTP 客户端** | HttpWebRequest | IHttpClientFactory / Refit |
| **SDK 生命周期** | 静态类，手动管理 | 单例服务，DI 管理，IAsyncDisposable |
| **图片存储** | 绝对路径 + 逗号分隔字符串 | `AttachmentFile` 实体 + 相对路径 + 关联表 |
| **文件同步** | 无 | `AttachmentService` + OSS + 服务端 API |
