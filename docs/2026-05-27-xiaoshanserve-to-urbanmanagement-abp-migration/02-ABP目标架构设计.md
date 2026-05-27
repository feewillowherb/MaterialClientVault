# 02 - ABP 目标架构设计

> **更新说明**：本文档已根据 UrbanManagement 实际项目结构更新。原规划为 7 项目完整 DDD 分层，实际采用 2 项目简化架构。

## 1. 实际项目结构（已实现）

```
UrbanManagement.sln
├── src/
│   ├── UrbanManagement.Core/              ← 领域层 + 基础设施 + 服务（合并）
│   │   ├── Configuration/
│   │   │   └── AppSettings.cs
│   │   ├── Entities/
│   │   │   ├── Enums/
│   │   │   │   └── SyncStatus.cs
│   │   │   ├── GovLog.cs
│   │   │   ├── GovProject.cs
│   │   │   ├── GovSyncData.cs
│   │   │   └── UrbanWeighingRecord.cs
│   │   ├── EntityFrameworkCore/
│   │   │   └── UrbanManagementDbContext.cs
│   │   ├── Services/
│   │   │   ├── IUrbanWeighingRecordAppService.cs
│   │   │   ├── SampleDataProvider.cs
│   │   │   └── UrbanWeighingRecordAppService.cs
│   │   ├── UrbanManagement.Core.csproj
│   │   └── UrbanManagementCoreModule.cs
│   └── UrbanManagement.App/               ← Web 宿主层（控制器 + 视图 + 静态文件）
│       ├── Controllers/
│       │   ├── HomeController.cs
│       │   ├── MainPageController.cs
│       │   ├── ProjectController.cs
│       │   ├── SyncInfoController.cs
│       │   └── UrbanWeighingRecordController.cs
│       ├── Models/
│       │   ├── ProjectFormModel.cs
│       │   └── UrbanWeighingRecordDto.cs
│       ├── Views/
│       │   ├── Home/Index.cshtml
│       │   ├── MainPage/Index.cshtml
│       │   ├── Project/{Index, Add}.cshtml
│       │   ├── SyncInfo/Index.cshtml
│       │   └── Shared/{_Layout, _ViewImports, _ViewStart}.cshtml
│       ├── wwwroot/
│       │   ├── css/site.css
│       │   ├── js/site.js
│       │   └── public/style/admin.css
│       ├── appsettings.json
│       ├── Program.cs
│       ├── UrbanManagement.App.csproj
│       └── UrbanManagementAppModule.cs
└── tests/
    └── UrbanManagement.Core.Tests/
        └── UrbanManagement.Core.Tests.csproj   ← 暂无实际测试
```

### 1.1 模块依赖关系（实际）

```
UrbanManagement.App (Web 宿主)
    ├── depends on → UrbanManagement.Core
    ├── depends on → AbpAutofacModule
    └── depends on → AbpAspNetCoreMvcModule

UrbanManagement.Core (领域+基础设施)
    ├── depends on → AbpEntityFrameworkCoreModule
    └── depends on → AbpEntityFrameworkCoreSqliteModule
```

### 1.2 与原规划架构的对比

| 维度 | 原规划（7 项目 DDD） | 实际实现（2 项目简化） | 说明 |
|------|---------------------|---------------------|------|
| 领域层 | `UrbanManagement.Domain` | 合并到 `UrbanManagement.Core` | 实体直接在 Core 中 |
| 应用层 | `UrbanManagement.Application` | 合并到 `UrbanManagement.Core/Services` | 服务直接在 Core 中 |
| 契约层 | `UrbanManagement.Application.Contracts` | 无 | 接口直接在 Core/Services 中 |
| HTTP API 层 | `UrbanManagement.HttpApi` | 合并到 `UrbanManagement.App/Controllers` | 控制器在 App 中 |
| HTTP 客户端层 | `UrbanManagement.HttpApi.Client` | **未实现** | Refit 接口未引入 |
| EF Core 层 | `UrbanManagement.EntityFrameworkCore` | 合并到 `UrbanManagement.Core/EntityFrameworkCore` | DbContext 在 Core 中 |
| Web 宿主 | `UrbanManagement.Web` | `UrbanManagement.App` | 名称不同 |
| 测试 | 3 个测试项目 | 1 个测试项目 | 暂无实际测试 |

**简化原因**：项目规模较小（旧系统仅 6 个 Controller、3 个实体），完整 DDD 分层增加复杂度但收益有限。可随业务增长再拆分。

## 2. 实际 ABP 模块配置

### 2.1 Core 模块

```csharp
// UrbanManagementCoreModule.cs（实际代码）
[DependsOn(
    typeof(AbpEntityFrameworkCoreModule),
    typeof(AbpEntityFrameworkCoreSqliteModule)
)]
public class UrbanManagementCoreModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        context.Services.AddAbpDbContext<UrbanManagementDbContext>(options =>
        {
            options.AddDefaultRepositories(true); // 自动注册默认仓储
        });

        Configure<AbpDbContextOptions>(options =>
        {
            options.UseSqlite();
        });
    }
}
```

### 2.2 App 模块

```csharp
// UrbanManagementAppModule.cs（实际代码）
[DependsOn(
    typeof(UrbanManagementCoreModule),
    typeof(AbpAutofacModule),
    typeof(AbpAspNetCoreMvcModule)
)]
public class UrbanManagementAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        // JSON 序列化：camelCase + 忽略 null
        context.Services.AddControllersWithViews()
            .AddJsonOptions(options =>
            {
                options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
                options.JsonSerializerOptions.DefaultIgnoreCondition =
                    JsonIgnoreCondition.WhenWritingNull;
            });

        // 开发环境 Razor 热重载
        if (environment.IsDevelopment())
        {
            context.Services.AddRazorPages().AddRazorRuntimeCompilation();
        }
    }
}
```

### 2.3 启动入口

```csharp
// Program.cs（实际代码）
var builder = WebApplication.CreateBuilder(args);

builder.Host
    .UseAutofac()
    .UseSerilog((context, services, loggerConfig) =>
    {
        loggerConfig
            .ReadFrom.Configuration(context.Configuration)
            .ReadFrom.Services(services);
    });

builder.Services.AddApplication<UrbanManagementAppModule>();

var app = builder.Build();
app.InitializeApplication();
app.Run();
```

## 3. 实际实体设计

### 3.1 GovProject（项目）

```csharp
// 实际代码 — Entities/GovProject.cs
public class GovProject : Entity<Guid>   // 主键为 Guid，非规划的 int
{
    public string ProName { get; set; } = default!;
    public string? BuildLicenseNo { get; set; }
    public string? FdBuildLicenseNo { get; set; }
    public DateTime? AddTime { get; set; }
    public bool? SyncStatus { get; set; }
    public DateTime? LastSyncTime { get; set; }
    public bool? DeleteStatus { get; set; }
}
```

**与规划的差异**：
- 主键 `Guid`（规划为 `int`）— ProId 即 Id，GUID 作为业务标识
- 无 `IHasCreationTime` 审计接口 — 使用 `AddTime` 自定义字段
- 字段名保持与旧系统一致（未重命名 `DeleteStatus` → `IsDeleted`）
- 无领域方法（`EnableSync()`、`DisableSync()` 等）

### 3.2 GovSyncData（同步数据）

```csharp
// 实际代码 — Entities/GovSyncData.cs
public class GovSyncData : Entity<int>   // 主键为 int，非规划的 long
{
    public string? CarNo { get; set; }
    public string? CarColor { get; set; }
    public string? CarNoColor { get; set; }
    public string? CarType { get; set; }
    public string? SnapTime { get; set; }
    public string? DeviceId { get; set; }
    public string? BuildLicenseNo { get; set; }
    public string? SiteType { get; set; }
    public string? GoodsWeight { get; set; }
    public string? SourceData { get; set; }
    public DateTime? AddTime { get; set; }
    public string? ProId { get; set; }
    public string? ProName { get; set; }
    public int? SyncType { get; set; }       // 仍为 int，未改为 enum
    public DateTime? SyncTime { get; set; }
    public int? SyncNumber { get; set; }
    public string? SnapImages { get; set; }
}
```

**与规划的差异**：
- 主键 `int`（规划为 `long`）
- 字段名全部保留旧系统原名（未重命名 `SyncNumber` → `SyncAttempts` 等）
- `SyncType` 仍为 `int?`，未改为 `SyncStatus` 枚举
- 无 `ProjectId`/`ProjectName`，使用 `ProId`/`ProName`（与旧系统一致）
- 无领域方法

### 3.3 GovLog（同步日志）

```csharp
// 实际代码 — Entities/GovLog.cs
public class GovLog : Entity<int>   // 主键为 int，名称为 GovLog（非规划的 GovSyncLog）
{
    public int? SyncId { get; set; }
    public DateTime? SyncTime { get; set; }
    public int? SyncNumber { get; set; }
    public string? SyncSource { get; set; }
    public string? SyncResult { get; set; }
    public string? SyncCode { get; set; }
    public string? SyncMsg { get; set; }
}
```

**与规划的差异**：
- 类名 `GovLog`（规划为 `GovSyncLog`）— 保持与旧系统一致
- 主键 `int`（规划为 `long`）
- 字段名全部保留旧系统原名（`SyncId`/`SyncMsg`，未改为 `SyncDataId`/`Message`）

### 3.4 UrbanWeighingRecord（新增实体）

```csharp
// 实际代码 — Entities/UrbanWeighingRecord.cs
// 这是规划中不存在的全新实体，用于新客户端（MaterialClient 架构）的称重记录上传
public class UrbanWeighingRecord : Entity<long>
{
    public long ClientRecordId { get; set; }   // 客户端记录 ID，唯一索引（幂等去重）
    public string? PlateNumber { get; set; }    // 车牌号
    public decimal TotalWeight { get; set; }    // 总重量
    public DateTime WeighingTime { get; set; }  // 称重时间
    public DateTime AddTime { get; set; }       // 入库时间
    public int? SyncType { get; set; }          // 同步类型
    public string? SnapImages { get; set; }     // 抓拍图片路径
}
```

**设计要点**：
- 以 MaterialClient 本地 WeighingRecord 为蓝本（OQ-4 原则）
- `ClientRecordId` 唯一索引，支持幂等去重
- `TotalWeight` 使用 `decimal`（非 `int`），精度更高
- 与 GovSyncData 不同，这是服务端接收新客户端数据的实体

## 4. 实际服务层设计

### 4.1 已实现的服务

```csharp
// IUrbanWeighingRecordAppService.cs（实际代码）
public interface IUrbanWeighingRecordAppService
{
    Task<long> ReceiveAsync(long clientRecordId, string? plateNumber, decimal totalWeight,
        DateTime weighingTime, int? syncType = null, string? snapImages = null);

    Task<(List<UrbanWeighingRecord> Data, int Total)> GetPagedAsync(
        int page = 1, int limit = 20, string? searchText = null,
        DateTime? startTime = null, DateTime? endTime = null);
}
```

```csharp
// UrbanWeighingRecordAppService.cs（实际代码）
public class UrbanWeighingRecordAppService : IUrbanWeighingRecordAppService, ITransientDependency
{
    private readonly IRepository<UrbanWeighingRecord, long> _repository;

    public async Task<long> ReceiveAsync(...)
    {
        // ClientRecordId 去重检查 → 已存在则返回现有 ID
        // 不存在 → 插入新记录
        await _repository.InsertAsync(record, autoSave: true);
        return record.Id;
    }

    public async Task<(List<UrbanWeighingRecord> Data, int Total)> GetPagedAsync(...)
    {
        // 车牌号模糊查询 + 时间范围筛选
        // 按 AddTime 降序排列
    }
}
```

### 4.2 Mock 数据提供者

```csharp
// SampleDataProvider.cs（实际代码）— 提供模拟数据供前端展示
public interface ISampleDataProvider
{
    Task<(List<GovProject> Data, int Total)> GetPagedProjectsAsync(int page, int limit);
    Task<(List<GovSyncData> Data, int Total)> GetPagedSyncDataAsync(int page, int limit);
    Task<List<GovLog>> GetSyncLogsAsync(int syncDataId);
    Task<DashboardStats> GetDashboardStatsAsync();
}
```

`ProjectController` 和 `SyncInfoController` 当前使用 `ISampleDataProvider`，返回硬编码的模拟数据，尚未接入真实数据库操作。

## 5. 待实现的架构组件

以下组件在原规划中有设计，但实际项目尚未实现：

### 5.1 旧客户端兼容控制器（高优先级）

```csharp
// 待实现 — 保持路由 POST /Api/Post 与旧 GovClient 完全一致
[Route("Api/[action]")]
[ApiController]
public class LegacyApiController : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult> Post([FromBody] dynamic model)
    {
        // 反序列化 → 验证接入码 → 图片处理 → 入库
    }
}
```

### 5.2 领域服务（高优先级）

```csharp
// 待实现 — 接入码双重验证
public interface IGovProjectManager  // 或作为 AppService
{
    Task<GovProject?> ValidateAccessCodeAsync(string? buildLicenseNo, string? fdBuildLicenseNo);
}
```

### 5.3 图片处理服务（高优先级）

```csharp
// 待实现 — Base64 接收 → 保存 → 压缩
public interface IFileService
{
    Task<List<string>> SaveAndCompressImagesAsync(string[] base64Images, string buildLicenseNo);
}
```

### 5.4 后台同步 Worker（中优先级）

```csharp
// 待实现 — 替代旧系统 ExplortStatisticBgService
public class GovSyncBackgroundWorker : AsyncPeriodicBackgroundWorkerBase
{
    // Timer.Period = 5000 (5秒间隔)
    // 查询待同步数据 → 读取图片 → HTTP 转发到政府 API → 更新状态
}
```

### 5.5 HTTP 客户端（中优先级）

```csharp
// 待实现 — Refit + Polly 政府平台转发
public interface IGovSyncHttpClient
{
    [Post("")]
    Task<GovResponseBase<string>> PostWeightAsync([Body] GovForwardRequest request);
}
```

## 6. 实际 DbContext 配置

```csharp
// UrbanManagementDbContext.cs（实际代码）
[ConnectionStringName("Default")]
public class UrbanManagementDbContext : AbpDbContext<UrbanManagementDbContext>
{
    public DbSet<GovProject> GovProjects => Set<GovProject>();
    public DbSet<GovSyncData> GovSyncData => Set<GovSyncData>();
    public DbSet<GovLog> GovLogs => Set<GovLog>();
    public DbSet<UrbanWeighingRecord> UrbanWeighingRecords => Set<UrbanWeighingRecord>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        builder.Entity<GovProject>(b =>
        {
            b.ToTable("Gov_Project");
            b.HasKey(e => e.Id);                         // Guid 主键
            b.Property(e => e.ProName).IsRequired().HasMaxLength(200);
            b.Property(e => e.BuildLicenseNo).HasMaxLength(200);
            b.Property(e => e.FdBuildLicenseNo).HasMaxLength(200);
        });

        builder.Entity<GovSyncData>(b =>
        {
            b.ToTable("Gov_SyncData");
            b.HasKey(e => e.Id);                         // int 主键
            b.Property(e => e.CarNo).HasMaxLength(50);
            b.Property(e => e.GoodsWeight).HasMaxLength(50);
            b.Property(e => e.SnapTime).HasMaxLength(100);
            b.Property(e => e.SnapImages).HasMaxLength(2000);
        });

        builder.Entity<GovLog>(b =>
        {
            b.ToTable("Gov_Log");
            b.HasKey(e => e.Id);                         // int 主键
            b.Property(e => e.SyncMsg).HasMaxLength(2000);
            b.Property(e => e.SyncResult).HasMaxLength(2000);
        });

        builder.Entity<UrbanWeighingRecord>(b =>
        {
            b.ToTable("Urban_WeighingRecord");
            b.HasKey(e => e.Id);                         // long 主键
            b.Property(e => e.PlateNumber).HasMaxLength(50);
            b.Property(e => e.SnapImages).HasMaxLength(2000);
            b.HasIndex(e => e.ClientRecordId).IsUnique(); // 幂等去重索引
        });
    }
}
```

## 7. 技术栈汇总（实际）

| 项 | 值 |
|----|-----|
| 框架 | ASP.NET Core 10 + ABP 10.0.1 |
| ORM | EF Core 10.0.1 + SQLite |
| DI 容器 | Autofac（ABP 集成） |
| 日志 | Serilog（Console + File） |
| JSON 序列化 | System.Text.Json（camelCase + 忽略 null） |
| 前端 | LayUI 2.9.21 + Bootstrap 5 + ECharts 5.5.1 + jQuery |
| 包管理 | Central Package Management（Directory.Packages.props） |
| 目标框架 | .NET 10 |

## 8. 与 MaterialClient 架构的对照

| 维度 | MaterialClient | UrbanManagement（实际） | 说明 |
|------|---------------|------------------------|------|
| 客户端/服务端 | 客户端（Avalonia） | 服务端（Web API + MVC） | 角色不同 |
| ABP 模块 | `MaterialClientCommonModule` | `UrbanManagementCoreModule` | 同样继承 `AbpModule` |
| 项目数量 | 4 个项目 | 3 个项目 | 都采用简化架构 |
| EF Core | SQLite | SQLite | 同样使用 `AbpDbContext` |
| 后台服务 | `PollingBackgroundService` | **未实现**（计划用 Worker） | 同样模式 |
| HTTP 客户端 | Refit + Polly | **未实现** | 同样模式 |
| 实体基类 | `Entity<T>` | `Entity<T>` | 同样模式 |
| DI 模式 | `ITransientDependency` | `ITransientDependency` | 同样模式 |
| JSON 配置 | camelCase | camelCase | 完全一致 |
