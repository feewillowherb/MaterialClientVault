# 04 - API 兼容性方案

> **更新说明**：本文档已根据最新方案更新。当前主链路为 `UrbanWeighingRecord` 接收后再组装 `GovSyncData` 上行远程服务；旧客户端兼容层（`POST /Api/Post`）仍待实现。

## 1. 当前 API 实现状态

### 1.1 已实现：新客户端 API

| 端点 | 方法 | 状态 | 说明 |
|------|------|------|------|
| `/api/urban/weighing-records` | POST | ✅ 已实现 | 接收称重记录（新客户端） |
| `/api/urban/weighing-records` | GET | ✅ 已实现 | 分页查询称重记录 |
| `/Project/PageList` | POST | ⚠️ Mock | 项目列表（SampleDataProvider） |
| `/Project/Add` | GET/POST | ⚠️ Mock | 项目新增（返回固定成功） |
| `/Project/SetStatus` | POST | ⚠️ Mock | 同步状态切换（返回固定成功） |
| `/Project/Del` | POST | ⚠️ Mock | 项目删除（返回固定成功） |
| `/SyncInfo/PageList` | POST | ⚠️ Mock | 同步数据列表（SampleDataProvider） |
| `/SyncInfo/LogList` | GET | ⚠️ Mock | 同步日志（SampleDataProvider） |
| `/Home/Index` | GET | ✅ 已实现 | 仪表盘首页 |
| `/MainPage/Index` | GET | ✅ 已实现 | 主页面 |

### 1.2 未实现：旧客户端兼容 API

| 端点 | 方法 | 状态 | 说明 |
|------|------|------|------|
| `/Api/Post` | POST | ❌ 未实现 | **旧 GovClient 核心数据接收端点** |

## 2. 已实现的 API 详情

### 2.1 UrbanWeighingRecordController（新客户端）

```csharp
// 实际代码 — Controllers/UrbanWeighingRecordController.cs
[Route("api/urban/weighing-records")]
public class UrbanWeighingRecordController : AbpController
{
    private readonly IUrbanWeighingRecordAppService _appService;

    /// <summary>
    /// 接收称重记录 — POST /api/urban/weighing-records
    /// </summary>
    [HttpPost]
    public async Task<IActionResult> Receive([FromBody] UrbanWeighingRecordDto dto)
    {
        // ModelState 验证 → 调用 AppService → ClientRecordId 幂等去重
        var id = await _appService.ReceiveAsync(
            dto.ClientRecordId, dto.PlateNumber, dto.TotalWeight,
            dto.WeighingTime, dto.SyncType, dto.SnapImages);

        return Json(new { success = true, data = new { id } });
    }

    /// <summary>
    /// 分页查询 — GET /api/urban/weighing-records
    /// </summary>
    [HttpGet]
    public async Task<IActionResult> GetPaged(
        int page = 1, int limit = 20, string? searchText = null,
        DateTime? startTime = null, DateTime? endTime = null)
    {
        var (data, total) = await _appService.GetPagedAsync(
            page, limit, searchText, startTime, endTime);

        return Json(new { success = true, count = total, data });
    }
}
```

### 2.2 DTO 设计（已实现）

```csharp
// 实际代码 — Models/UrbanWeighingRecordDto.cs
public class UrbanWeighingRecordDto
{
    [Required]
    public long ClientRecordId { get; set; }   // 客户端记录 ID（幂等去重）

    public string? PlateNumber { get; set; }    // 车牌号
    public string? VehicleColor { get; set; }   // 车身颜色
    public string? PlateColor { get; set; }     // 车牌颜色
    public string? VehicleType { get; set; }    // 车型

    [Required]
    public decimal TotalWeight { get; set; }    // 总重量

    [Required]
    public DateTime WeighingTime { get; set; }  // 称重时间

    public string? DeviceId { get; set; }       // 设备编号
    public string? BuildLicenseNo { get; set; } // //TODO 授权文件来源待定
    public string? SiteType { get; set; }       // //TODO 授权文件来源待定
    public string? ProId { get; set; }          // //TODO 授权文件来源待定
    public string? ProName { get; set; }        // //TODO 授权文件来源待定

    public int? ClientSyncType { get; set; }    // 客户端同步状态（Client 前缀）
    public DateTime? ClientSyncTime { get; set; } // 客户端同步时间
    public int? ClientRetryCount { get; set; }  // 客户端重试次数
    public DateTime? ClientLastErrorTime { get; set; } // 客户端最近失败时间
    public bool IsAnomaly { get; set; }         // 异常标记
    public string? SnapImages { get; set; }     // 抓拍图片路径
}
```

**与旧客户端 DTO 的差异**：

| 对比项 | 旧客户端（mGovRequestWeight） | 新客户端（UrbanWeighingRecordDto） |
|--------|-------------------------------|----------------------------------|
| 字段命名 | camelCase（`carNo`） | PascalCase（`PlateNumber`） |
| 字段数量 | 16 个字段 | 与 GovSyncData 大部分业务字段同构（同步状态不进入 GovSyncData） |
| 车牌号 | `carNo` | `PlateNumber` |
| 重量 | `grossWeight`/`tareWeight`/`goodsWeight` | `TotalWeight`（合并） |
| 图片 | `snapImages`（Base64 数组） | `SnapImages`（路径字符串） |
| 接入码 | `buildLicenseNo`/`fdBuildLicenseNo` | 无（由新客户端自行关联） |
| 去重方式 | 无 | `ClientRecordId` 幂等 |

### 2.3 响应格式（已统一）

所有 API 统一使用以下响应格式（JSON 属性名为 camelCase，由全局 `JsonNamingPolicy.CamelCase` 配置）：

```json
// 成功响应
{
    "success": true,
    "data": { "id": 1 },
    "count": 0
}

// 列表响应
{
    "success": true,
    "count": 100,
    "data": [...]
}

// 错误响应
{
    "success": false,
    "msg": "错误信息"
}
```

此格式与旧系统 `ApiResultDto` 兼容（`success`/`msg`/`code`/`data`/`count`），但当前实现中缺少 `code` 字段。

## 3. JSON 序列化配置（已实现）

### 3.1 全局配置

```csharp
// UrbanManagementAppModule.cs（实际代码）
context.Services.AddControllersWithViews()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        options.JsonSerializerOptions.DefaultIgnoreCondition =
            JsonIgnoreCondition.WhenWritingNull;
    });
```

**与旧系统兼容性**：
- ✅ camelCase 属性名 — 与旧系统一致
- ✅ null 值忽略 — 与旧系统 Newtonsoft.Json `NullValueHandling.Ignore` 一致
- ⚠️ 日期格式 — 当前使用 System.Text.Json 默认 ISO 8601，旧系统使用 `yyyy-MM-dd`（需添加自定义 Converter）

### 3.2 待补充：日期格式兼容

```csharp
// 待添加到 AddJsonOptions 配置中
options.JsonSerializerOptions.Converters.Add(new CustomDateTimeConverter());

public class CustomDateTimeConverter : JsonConverter<DateTime>
{
    public override DateTime Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
        => DateTime.TryParse(reader.GetString(), out var dt) ? dt : default;

    public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions options)
        => writer.WriteStringValue(value.ToString("yyyy-MM-dd"));
}
```

## 4. 旧客户端兼容层（待实现）

### 4.1 兼容性要求

旧 GovClient（.NET Framework WinForms）不可修改，必须保持：

| 兼容项 | 旧格式 | 要求 |
|--------|--------|------|
| 路由 | `/Api/Post` | 必须完全一致 |
| HTTP 方法 | POST | 不变 |
| Content-Type | application/json | 不变 |
| 请求体字段名 | camelCase（`carNo`、`snapImages` 等） | 保持 |
| 响应字段名 | `success`/`msg`/`code`/`data`/`count` | 保持 |
| 请求 DTO | `dynamic` 参数接收 | 保持 |

### 4.2 兼容层设计

```
旧客户端 (GovClient)                    新客户端 (MaterialClient)
       │                                      │
       │ POST /Api/Post                       │ POST /api/urban/weighing-records
       ▼                                      ▼
┌──────────────────┐              ┌───────────────────────────┐
│ LegacyApiController│              │ UrbanWeighingRecordController│
│ (待实现)           │              │ (已实现)                     │
└────────┬─────────┘              └──────────┬────────────────┘
         │                                    │
         │ GovRequestWeightDto                │ UrbanWeighingRecordDto
         │ (camelCase 字段)                    │ (PascalCase 属性)
         │                                    │
         ├─→ IGovProjectManager (待实现)       ├─→ IUrbanWeighingRecordAppService
         ├─→ IFileService (待实现)             │    (已实现)
         └─→ UrbanWeighingRecord 入库          └─→ GovSyncData 组装并上行
```

### 4.3 LegacyApiController 实现（待编码）

```csharp
/// <summary>
/// 旧客户端兼容控制器 — 路由格式与 XiaoShanServe 完全一致
/// </summary>
[Route("Api/[action]")]
public class LegacyApiController : AbpController
{
    private readonly IRepository<GovSyncData, Guid> _syncDataRepo;
    private readonly IRepository<GovProject, Guid> _projectRepo;
    private readonly IFileService _fileService;
    private readonly ILogger<LegacyApiController> _logger;

    [HttpPost]
    public async Task<IActionResult> Post([FromBody] JsonElement model)
    {
        var result = new { success = false, msg = "", code = -1, data = (object?)null };

        try
        {
            // 1. 解析旧客户端 JSON（camelCase 字段）
            var carNo = model.GetProperty("carNo").GetString();
            var buildLicenseNo = model.TryGetProperty("buildLicenseNo", out var bln)
                ? bln.GetString() : null;
            var fdBuildLicenseNo = model.TryGetProperty("fdBuildLicenseNo", out var fbln)
                ? fbln.GetString() : null;

            // 2. 验证接入码（双重验证逻辑）
            var project = await ValidateAccessCodeAsync(buildLicenseNo, fdBuildLicenseNo);
            if (project == null)
            {
                return Json(new { success = false, msg = "接入码未找到", code = -1 });
            }

            // 3. 处理图片 Base64 → 保存
            // 4. 构建 GovSyncData 实体 → 入库
            // 5. 返回兼容格式

            return Json(new { success = true, msg = "成功", code = 200 });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Legacy API Post failed");
            return Json(new { success = false, msg = ex.Message, code = -1 });
        }
    }
}
```

### 4.4 兼容 DTO（待创建）

```csharp
/// <summary>
/// 旧客户端请求 DTO — 字段名保持 camelCase 与旧 GovClient 发送的 JSON 一致
/// </summary>
public class GovRequestWeightDto
{
    public string? carNo { get; set; }
    public string? carColor { get; set; }
    public string? carNoColor { get; set; }
    public string? buildLicenseNo { get; set; }
    public string? fdBuildLicenseNo { get; set; }
    public int inOutType { get; set; }
    public string? equipmentNumber { get; set; }
    public string? equipmentType { get; set; }
    public int grossWeight { get; set; }
    public int tareWeight { get; set; }
    public string? snapTime { get; set; }
    public string?[]? snapImages { get; set; }   // Base64 数组
    public string? carType { get; set; }
    public string? deviceID { get; set; }
    public string? siteType { get; set; }
    public string? goodsWeight { get; set; }
}
```

> **注意**：由于全局 `JsonNamingPolicy.CamelCase` 已配置，如果此 DTO 属性名用 camelCase 定义，System.Text.Json 会将属性名转为 camelCase，与旧客户端字段名匹配。但更安全的做法是使用 `[JsonPropertyName("carNo")]` 显式标注。

## 5. 端点兼容对照

| 旧端点 | 旧方法 | 新端点 | 新方法 | 兼容状态 |
|--------|--------|--------|--------|---------|
| `/Api/Post` | POST | `/Api/Post`（LegacyApiController） | POST | ❌ 未实现 |
| `/Project/PageList` | POST | `/Project/PageList`（ProjectController） | POST | ⚠️ Mock |
| `/Project/Add` | GET | `/Project/Add`（ProjectController） | GET | ⚠️ Mock |
| `/Project/Add` | POST | `/Project/Add`（ProjectController） | POST | ⚠️ Mock |
| `/Project/SetStatus` | GET | `/Project/SetStatus`（ProjectController） | POST | ⚠️ Mock（方法改为 POST） |
| `/Project/Del` | GET | `/Project/Del`（ProjectController） | POST | ⚠️ Mock（方法改为 POST） |
| `/SyncInfo/PageList` | POST | `/SyncInfo/PageList`（SyncInfoController） | POST | ⚠️ Mock |
| `/SyncInfo/logList` | GET | `/SyncInfo/LogList`（SyncInfoController） | GET | ⚠️ Mock（大小写变更） |
| `/FileViewer/Index` | GET | — | — | ❌ 未实现 |
| — | — | `/api/urban/weighing-records` | POST | ✅ 新增 |
| — | — | `/api/urban/weighing-records` | GET | ✅ 新增 |

## 6. 测试验证方案

### 6.1 契约测试（待实现）

```csharp
[Fact]
public async Task Legacy_Post_Should_Accept_Old_Format()
{
    var json = @"{
        ""carNo"": ""浙A12345"",
        ""carColor"": ""白"",
        ""carNoColor"": ""黄"",
        ""buildLicenseNo"": ""TEST001"",
        ""inOutType"": 0,
        ""grossWeight"": 5000,
        ""tareWeight"": 0,
        ""snapTime"": ""2026-05-27 10:30:00"",
        ""snapImages"": [],
        ""carType"": ""大车"",
        ""deviceID"": ""01"",
        ""siteType"": ""1"",
        ""goodsWeight"": ""5000""
    }";

    var response = await _client.PostAsync("/Api/Post",
        new StringContent(json, Encoding.UTF8, "application/json"));

    response.EnsureSuccessStatusCode();
    var result = await response.Content.ReadFromJsonAsync<JsonElement>();
    Assert.True(result.GetProperty("success").GetBoolean());
    Assert.Equal(200, result.GetProperty("code").GetInt32());
}

[Fact]
public async Task New_API_Should_Accept_UrbanWeighingRecordDto()
{
    var json = @"{
        ""clientRecordId"": 12345,
        ""plateNumber"": ""浙A12345"",
        ""totalWeight"": 5000.5,
        ""weighingTime"": ""2026-05-27T10:30:00""
    }";

    var response = await _client.PostAsync("/api/urban/weighing-records",
        new StringContent(json, Encoding.UTF8, "application/json"));

    response.EnsureSuccessStatusCode();
}
```

### 6.2 回归测试

1. 实现 LegacyApiController 后，让旧 GovClient 持续运行 48 小时
2. 对比新旧系统的数据接收记录是否完全一致
3. 验证图片存储路径可正确访问
4. 验证后台同步功能正常
