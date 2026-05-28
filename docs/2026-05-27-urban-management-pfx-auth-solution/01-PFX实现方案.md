# Urban 客户端 PFX 实现方案

## 1. 方案架构

### 1.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Urban 客户端                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐        ┌──────────────────┐              │
│  │  PFX 证书管理    │        │  许可证号管理    │              │
│  │  - 证书加载      │        │  - BuildLicenseNo│              │
│  │  - 私钥保护      │        │  - FdBuildLicense│              │
│  └────────┬─────────┘        └────────┬─────────┘              │
│           │                           │                          │
│           └───────────┬───────────────┘                          │
│                       │                                          │
│                       ▼                                          │
│           ┌───────────────────────┐                              │
│           │  双重验证控制器        │                              │
│           │  - PFX 验证           │                              │
│           │  - 许可证号验证       │                              │
│           │  - 关联验证           │                              │
│           └───────────┬───────────┘                              │
│                       │                                          │
│                       ▼                                          │
│           ┌───────────────────────┐                              │
│           │  HTTP 客户端层        │                              │
│           │  - Refit API          │                              │
│           │  - 证书处理           │                              │
│           │  - Bearer Token       │                              │
│           └───────────┬───────────┘                              │
└───────────────────────┼──────────────────────────────────────────┘
                        │
                        │ HTTPS + Client Certificate
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    UrbanManagement 服务端                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐        ┌──────────────────┐              │
│  │  证书验证        │        │  许可证号验证    │              │
│  │  - 证书链验证    │        │  - 数据库查询    │              │
│  │  - 证书吊销检查  │        │  - 字符串匹配    │              │
│  └────────┬─────────┘        └────────┬─────────┘              │
│           │                           │                          │
│           └───────────┬───────────────┘                          │
│                       │                                          │
│                       ▼                                          │
│           ┌───────────────────────┐                              │
│           │  授权决策控制器        │                              │
│           │  - 双重验证结果整合   │                              │
│           │  - 风险评估           │                              │
│           │  - 授权决策           │                              │
│           └───────────────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 验证流程

```
客户端请求 ──┐
             │
             ▼
    ┌─────────────────┐
    │ 1. 加载 PFX 证书 │
    │ 2. 验证证书有效性 │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ 3. 构建请求      │
    │  - 附加证书      │
    │  - 许可证号      │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ 4. HTTPS 请求   │
    │  - SSL/TLS 加密 │
    └────────┬────────┘
             │
             ▼
服务端接收 ──┐
             │
             ▼
    ┌─────────────────┐
    │ 5. 证书验证      │
    │  - 证书链验证    │
    │  - 证书吊销检查  │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ 6. 许可证号验证  │
    │  - 数据库查询    │
    │  - 字符串匹配    │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ 7. 关联验证      │
    │  - 证书-项目绑定 │
    │  - 风险评估      │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ 8. 授权决策      │
    │  - 允许/拒绝     │
    └────────┬────────┘
             │
             ▼
        响应返回
```

## 2. 客户端实现

### 2.1 PFX 证书管理

```csharp
// UrbanManagement.Common/Security/PfxCertificateManager.cs
using System.Security.Cryptography.X509Certificates;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace UrbanManagement.Common.Security;

public class PfxCertificateManager : IPfxCertificateManager
{
    private readonly ILogger<PfxCertificateManager> _logger;
    private readonly PfxCertificateOptions _options;
    private X509Certificate2? _certificate;

    public PfxCertificateManager(
        ILogger<PfxCertificateManager> logger,
        IOptions<PfxCertificateOptions> options)
    {
        _logger = logger;
        _options = options.Value;
    }

    public async Task<X509Certificate2> LoadCertificateAsync()
    {
        if (_certificate != null)
            return _certificate;

        try
        {
            var certPath = _options.CertificatePath;
            var certPassword = _options.CertificatePassword;

            if (!File.Exists(certPath))
            {
                throw new FileNotFoundException($"PFX 证书文件未找到: {certPath}");
            }

            _certificate = new X509Certificate2(certPath, certPassword,
                X509KeyStorageFlags.Exportable | X509KeyStorageFlags.MachineKeySet);

            _logger.LogInformation("成功加载 PFX 证书: {Subject}", _certificate.Subject);

            // 验证证书有效期
            ValidateCertificatePeriod(_certificate);

            return _certificate;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "加载 PFX 证书失败");
            throw;
        }
    }

    private void ValidateCertificatePeriod(X509Certificate2 cert)
    {
        var now = DateTime.Now;
        if (now < cert.NotBefore)
        {
            throw new InvalidOperationException($"证书尚未生效，生效时间: {cert.NotBefore}");
        }
        if (now > cert.NotAfter)
        {
            throw new InvalidOperationException($"证书已过期，过期时间: {cert.NotAfter}");
        }
    }

    public async Task<bool> ValidateCertificateAsync(X509Certificate2 certificate)
    {
        try
        {
            // 1. 验证证书有效期
            ValidateCertificatePeriod(certificate);

            // 2. 验证证书用途
            // TODO: 根据实际需求添加增强型密钥用法 (EKU) 验证

            // 3. 验证证书链（如果需要）
            // var chain = new X509Chain();
            // chain.ChainPolicy.RevocationMode = X509RevocationMode.Online;
            // chain.ChainPolicy.VerificationFlags = X509VerificationFlags.NoFlag;

            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "证书验证失败");
            return false;
        }
    }

    public void Dispose()
    {
        _certificate?.Dispose();
    }
}

public interface IPfxCertificateManager : IDisposable
{
    Task<X509Certificate2> LoadCertificateAsync();
    Task<bool> ValidateCertificateAsync(X509Certificate2 certificate);
}

public class PfxCertificateOptions
{
    public const string SectionName = "PfxCertificate";

    public string CertificatePath { get; set; } = string.Empty;
    public string CertificatePassword { get; set; } = string.Empty;
    public bool EnableValidation { get; set; } = true;
    public string? ExpectedIssuer { get; set; }
    public string? ExpectedSubject { get; set; }
}
```

### 2.2 HTTP 客户端配置

```csharp
// UrbanManagement.App/UrbanManagementAppModule.cs
using System.Security.Cryptography.X509Certificates;
using Microsoft.Extensions.DependencyInjection;
using Refit;
using UrbanManagement.Common.Security;
using UrbanManagement.Common.Api;

namespace UrbanManagement.App;

public class UrbanManagementAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var services = context.Services;
        var configuration = services.GetConfiguration();

        // 配置 PFX 证书选项
        services.Configure<PfxCertificateOptions>(
            configuration.GetSection(PfxCertificateOptions.SectionName));

        // 注册 PFX 证书管理器
        services.AddSingleton<IPfxCertificateManager, PfxCertificateManager>();

        // 配置支持 PFX 的 HTTP 客户端
        var urbanApiUrl = configuration["UrbanApi:BaseUrl"] ?? "https://localhost:5001";

        services.AddRefitClient<IUrbanManagementApi>()
            .ConfigureHttpClient(c =>
            {
                c.BaseAddress = new Uri(urbanApiUrl);
                c.Timeout = TimeSpan.FromSeconds(30);
            })
            .ConfigurePrimaryHttpMessageHandler(sp =>
            {
                var certManager = sp.GetRequiredService<IPfxCertificateManager>();
                var certificate = certManager.LoadCertificateAsync().Result;

                return new HttpClientHandler
                {
                    ClientCertificates = { certificate },
                    ServerCertificateCustomValidationCallback = (sender, cert, chain, errors) =>
                    {
                        // 开发环境可以放宽验证，生产环境需要严格验证
                        #if DEBUG
                        return true;
                        #else
                        return errors == SslPolicyErrors.None;
                        #endif
                    },
                    SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13
                };
            })
            .AddHttpMessageHandler<PfxAuthorizationHandler>()
            .AddTransientHttpErrorPolicy(policy =>
                policy.WaitAndRetryAsync(
                    3,
                    retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))
                ));
    }
}
```

### 2.3 授权处理器

```csharp
// UrbanManagement.Common/Security/PfxAuthorizationHandler.cs
using System.Security.Cryptography.X509Certificates;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace UrbanManagement.Common.Security;

public class PfxAuthorizationHandler : DelegatingHandler
{
    private readonly ILogger<PfxAuthorizationHandler> _logger;
    private readonly PfxCertificateOptions _options;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public PfxAuthorizationHandler(
        ILogger<PfxAuthorizationHandler> logger,
        IOptions<PfxCertificateOptions> options,
        IHttpContextAccessor httpContextAccessor)
    {
        _logger = logger;
        _options = options.Value;
        _httpContextAccessor = httpContextAccessor;
    }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        try
        {
            // 添加许可证号头部（保留现有验证机制）
            var httpContext = _httpContextAccessor.HttpContext;
            if (httpContext != null)
            {
                var buildLicenseNo = httpContext.Items["BuildLicenseNo"]?.ToString();
                var fdBuildLicenseNo = httpContext.Items["FdBuildLicenseNo"]?.ToString();

                if (!string.IsNullOrEmpty(buildLicenseNo))
                {
                    request.Headers.Add("X-Build-License-No", buildLicenseNo);
                }
                if (!string.IsNullOrEmpty(fdBuildLicenseNo))
                {
                    request.Headers.Add("X-Fd-Build-License-No", fdBuildLicenseNo);
                }
            }

            // 添加客户端证书信息
            var certManager = httpContext?.RequestServices
                .GetService<IPfxCertificateManager>();
            if (certManager != null)
            {
                var certificate = await certManager.LoadCertificateAsync();
                var thumbprint = certificate.Thumbprint;
                request.Headers.Add("X-Certificate-Thumbprint", thumbprint);
            }

            return await base.SendAsync(request, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "PFX 授权处理失败");
            throw;
        }
    }
}
```

### 2.4 API 接口定义

```csharp
// UrbanManagement.Common/Api/IUrbanManagementApi.cs
using Refit;

namespace UrbanManagement.Common.Api;

public interface IUrbanManagementApi
{
    [Post("/api/urban/weighing-records")]
    Task<ApiResultDto> SubmitWeighingRecordAsync(
        [Body] UrbanWeighingRecordDto record,
        CancellationToken cancellationToken = default);

    [Get("/api/projects/{projectId}")]
    Task<ProjectDto> GetProjectAsync(
        string projectId,
        CancellationToken cancellationToken = default);

    [Post("/api/projects/validate-license")]
    Task<LicenseValidationResultDto> ValidateLicenseAsync(
        [Body] LicenseValidationRequestDto request,
        CancellationToken cancellationToken = default);
}
```

## 3. 服务端实现

### 3.1 证书验证中间件

```csharp
// UrbanManagement.App/Middleware/ClientCertificateMiddleware.cs
using System.Security.Cryptography.X509Certificates;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

namespace UrbanManagement.App.Middleware;

public class ClientCertificateMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ClientCertificateMiddleware> _logger;
    private readonly ClientCertificateOptions _options;

    public ClientCertificateMiddleware(
        RequestDelegate next,
        ILogger<ClientCertificateMiddleware> logger,
        IOptions<ClientCertificateOptions> options)
    {
        _next = next;
        _logger = logger;
        _options = options.Value;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (_options.RequireCertificate)
        {
            var clientCertificate = await context.Connection.GetClientCertificateAsync();
            if (clientCertificate == null)
            {
                _logger.LogWarning("客户端未提供证书");
                context.Response.StatusCode = StatusCodes.Status403Forbidden;
                await context.Response.WriteAsync("需要客户端证书");
                return;
            }

            // 验证证书
            var validationResult = await ValidateCertificateAsync(clientCertificate);
            if (!validationResult.IsValid)
            {
                _logger.LogWarning("客户端证书验证失败: {Reason}", validationResult.Reason);
                context.Response.StatusCode = StatusCodes.Status403Forbidden;
                await context.Response.WriteAsync($"证书验证失败: {validationResult.Reason}");
                return;
            }

            // 将证书存储到 HttpContext 中供后续使用
            context.Items["ClientCertificate"] = clientCertificate;
        }

        await _next(context);
    }

    private async Task<CertificateValidationResult> ValidateCertificateAsync(
        X509Certificate2 certificate)
    {
        try
        {
            // 1. 验证证书有效期
            var now = DateTime.Now;
            if (now < certificate.NotBefore || now > certificate.NotAfter)
            {
                return CertificateValidationResult.Fail("证书不在有效期内");
            }

            // 2. 验证证书颁发者
            if (!string.IsNullOrEmpty(_options.ExpectedIssuer))
            {
                if (!certificate.I.Contains(_options.ExpectedIssuer))
                {
                    return CertificateValidationResult.Fail("证书颁发者不匹配");
                }
            }

            // 3. 验证证书主题
            if (!string.IsNullOrEmpty(_options.ExpectedSubject))
            {
                if (!certificate.Subject.Contains(_options.ExpectedSubject))
                {
                    return CertificateValidationResult.Fail("证书主题不匹配");
                }
            }

            // 4. 验证证书链
            var chain = new X509Chain();
            chain.ChainPolicy.RevocationMode = X509RevocationMode.Online;
            chain.ChainPolicy.RevocationFlag = X509RevocationFlag.ExcludeRoot;
            chain.ChainPolicy.VerificationFlags = X509VerificationFlags.NoFlag;
            chain.ChainPolicy.VerificationTime = DateTime.Now;
            chain.ChainPolicy.UrlRetrievalTimeout = new TimeSpan(0, 0, 30);

            var chainValid = chain.Build(certificate);
            if (!chainValid)
            {
                var status = chain.ChainStatus.FirstOrDefault();
                var reason = status?.StatusInformation ?? "证书链验证失败";
                return CertificateValidationResult.Fail(reason);
            }

            return CertificateValidationResult.Success();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "证书验证过程中发生异常");
            return CertificateValidationResult.Fail($"证书验证异常: {ex.Message}");
        }
    }
}

public class ClientCertificateOptions
{
    public const string SectionName = "ClientCertificate";

    public bool RequireCertificate { get; set; } = true;
    public string? ExpectedIssuer { get; set; }
    public string? ExpectedSubject { get; set; }
}

public record CertificateValidationResult(
    bool IsValid,
    string? Reason = null)
{
    public static CertificateValidationResult Success() =>
        new CertificateValidationResult(true);
    public static CertificateValidationResult Fail(string reason) =>
        new CertificateValidationResult(false, reason);
}
```

### 3.2 双重验证服务

```csharp
// UrbanManagement.Core/Security/IDualValidationService.cs
using System.Security.Cryptography.X509Certificates;
using Volo.Abp.Domain.Repositories;

namespace UrbanManagement.Core.Security;

public interface IDualValidationService
{
    Task<DualValidationResult> ValidateAsync(
        X509Certificate2? certificate,
        string? buildLicenseNo,
        string? fdBuildLicenseNo);
}

public class DualValidationService : IDualValidationService
{
    private readonly IRepository<GovProject, Guid> _projectRepository;
    private readonly ILogger<DualValidationService> _logger;

    public DualValidationService(
        IRepository<GovProject, Guid> projectRepository,
        ILogger<DualValidationService> logger)
    {
        _projectRepository = projectRepository;
        _logger = logger;
    }

    public async Task<DualValidationResult> ValidateAsync(
        X509Certificate2? certificate,
        string? buildLicenseNo,
        string? fdBuildLicenseNo)
    {
        var result = new DualValidationResult();

        try
        {
            // 1. 证书验证
            if (certificate != null)
            {
                result.CertificateValid = await ValidateCertificateAsync(certificate);
            }
            else
            {
                result.CertificateValid = false;
                _logger.LogWarning("未提供客户端证书");
            }

            // 2. 许可证号验证（保留原有逻辑）
            result.LicenseValid = await ValidateLicenseAsync(buildLicenseNo, fdBuildLicenseNo);

            // 3. 关联验证
            if (result.CertificateValid && result.LicenseValid)
            {
                result.AssociationValid = await ValidateAssociationAsync(certificate!, buildLicenseNo!, fdBuildLicenseNo!);
            }

            // 4. 综合决策
            result.IsValid = result.CertificateValid && result.LicenseValid && result.AssociationValid;

            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "双重验证过程中发生异常");
            result.IsValid = false;
            result.ErrorMessage = ex.Message;
            return result;
        }
    }

    private async Task<bool> ValidateCertificateAsync(X509Certificate2 certificate)
    {
        try
        {
            // 基本验证
            var now = DateTime.Now;
            if (now < certificate.NotBefore || now > certificate.NotAfter)
            {
                _logger.LogWarning("证书不在有效期内");
                return false;
            }

            // TODO: 添加更多验证逻辑，如 CRL 检查、OCSP 验证等

            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "证书验证失败");
            return false;
        }
    }

    private async Task<bool> ValidateLicenseAsync(string? buildLicenseNo, string? fdBuildLicenseNo)
    {
        try
        {
            // 如果是对接码，需要验证
            if (!string.IsNullOrEmpty(fdBuildLicenseNo))
            {
                var project = await _projectRepository.FirstOrDefaultAsync(
                    p => p.FdBuildLicenseNo == fdBuildLicenseNo);
                if (project == null)
                {
                    _logger.LogWarning("对接码 [{FdBuildLicenseNo}] 未找到对应项目", fdBuildLicenseNo);
                    return false;
                }
            }

            // 如果是建设许可证号，需要验证
            if (!string.IsNullOrEmpty(buildLicenseNo))
            {
                var project = await _projectRepository.FirstOrDefaultAsync(
                    p => p.BuildLicenseNo == buildLicenseNo);
                if (project == null)
                {
                    _logger.LogWarning("建设许可证号 [{BuildLicenseNo}] 未找到对应项目", buildLicenseNo);
                    return false;
                }
            }

            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "许可证号验证失败");
            return false;
        }
    }

    private async Task<bool> ValidateAssociationAsync(
        X509Certificate2 certificate,
        string buildLicenseNo,
        string fdBuildLicenseNo)
    {
        try
        {
            // 验证证书与许可证号的关联关系
            // 这里可以根据实际业务逻辑实现，例如：
            // 1. 检查证书中的主题或扩展属性是否包含项目信息
            // 2. 检查证书指纹与数据库中的记录是否匹配

            var certThumbprint = certificate.Thumbprint;
            var project = await _projectRepository.FirstOrDefaultAsync(
                p => p.BuildLicenseNo == buildLicenseNo || p.FdBuildLicenseNo == fdBuildLicenseNo);

            if (project == null)
            {
                _logger.LogWarning("未找到与许可证号关联的项目");
                return false;
            }

            // TODO: 添加证书-项目关联验证逻辑
            // 例如：project.CertificateThumbprint == certThumbprint

            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "关联验证失败");
            return false;
        }
    }
}

public class DualValidationResult
{
    public bool IsValid { get; set; }
    public bool CertificateValid { get; set; }
    public bool LicenseValid { get; set; }
    public bool AssociationValid { get; set; }
    public string? ErrorMessage { get; set; }
    public GovProject? Project { get; set; }
}
```

## 4. 配置文件

### 4.1 客户端配置

```json
{
  "PfxCertificate": {
    "CertificatePath": "Certificates/client.pfx",
    "CertificatePassword": "YourPasswordHere",
    "EnableValidation": true,
    "ExpectedIssuer": "CN=UrbanManagement CA",
    "ExpectedSubject": "CN=UrbanClient"
  },
  "UrbanApi": {
    "BaseUrl": "https://api.urbanmanagement.com"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "UrbanManagement.Common.Security": "Debug"
    }
  }
}
```

### 4.2 服务端配置

```json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://*:5001",
        "Certificate": {
          "Path": "Certificates/server.pfx",
          "Password": "YourPasswordHere",
          "AllowInvalid": "false"
        }
      }
    }
  },
  "ClientCertificate": {
    "RequireCertificate": true,
    "ExpectedIssuer": "CN=UrbanManagement CA",
    "ExpectedSubject": "CN=UrbanClient"
  }
}
```

## 5. 迁移策略

### 5.1 分阶段实施

**阶段一：基础设施搭建（1-2 周）**
1. 生成测试证书
2. 实现 PFX 证书管理器
3. 配置 HTTP 客户端支持证书
4. 编写单元测试

**阶段二：双重验证机制（2-3 周）**
1. 实现证书验证中间件
2. 实现双重验证服务
3. 保留现有许可证号验证逻辑
4. 编写集成测试

**阶段三：现有系统迁移（2-3 周）**
1. 更新 UrbanManagement API
2. 更新 UrbanManagement 客户端
3. 向后兼容处理
4. 性能测试和优化

**阶段四：监控和优化（1-2 周）**
1. 添加监控和日志
2. 性能优化
3. 安全审计
4. 文档完善

### 5.2 兼容性保证

```csharp
// 向后兼容的验证逻辑
public async Task<DualValidationResult> ValidateWithFallbackAsync(
    X509Certificate2? certificate,
    string? buildLicenseNo,
    string? fdBuildLicenseNo)
{
    // 1. 如果有证书，使用双重验证
    if (certificate != null)
    {
        return await ValidateAsync(certificate, buildLicenseNo, fdBuildLicenseNo);
    }

    // 2. 如果没有证书，只使用许可证号验证（向后兼容）
    _logger.LogWarning("未提供证书，仅使用许可证号验证");
    var licenseValid = await ValidateLicenseAsync(buildLicenseNo, fdBuildLicenseNo);

    return new DualValidationResult
    {
        IsValid = licenseValid,
        CertificateValid = false, // 证书无效，但许可证号有效
        LicenseValid = licenseValid,
        AssociationValid = false
    };
}
```

## 6. 安全考虑

### 6.1 证书存储安全

```csharp
// 使用 Windows 证书存储（生产环境推荐）
public class WindowsCertificateStoreManager : ICertificateManager
{
    public X509Certificate2? LoadCertificateFromStore(string thumbprint)
    {
        using var store = new X509Store(StoreName.My, StoreLocation.LocalMachine);
        store.Open(OpenFlags.ReadOnly);

        var certificates = store.Certificates.Find(
            X509FindType.FindByThumbprint,
            thumbprint,
            validOnly: true);

        return certificates.Count > 0 ? certificates[0] : null;
    }
}
```

### 6.2 证书更新机制

```csharp
// 支持证书自动更新
public class CertificateRenewalService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var cert = await _certificateManager.LoadCertificateAsync();
                var daysUntilExpiry = (cert.NotAfter - DateTime.Now).Days;

                if (daysUntilExpiry < 30)
                {
                    _logger.LogWarning("证书即将过期: {NotAfter}, 剩余天数: {Days}",
                        cert.NotAfter, daysUntilExpiry);

                    // 触发证书更新流程
                    await RenewCertificateAsync();
                }

                await Task.Delay(TimeSpan.FromDays(1), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "证书续期检查失败");
                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
            }
        }
    }
}
```

## 7. 测试策略

### 7.1 单元测试

```csharp
[Fact]
public async Task ValidateAsync_WithValidCertificateAndLicense_ShouldReturnValid()
{
    // Arrange
    var mockCert = new X509Certificate2(TestCertificates.ValidCert);
    var service = new DualValidationService(/* dependencies */);

    // Act
    var result = await service.ValidateAsync(mockCert, "TEST001", null);

    // Assert
    Assert.True(result.IsValid);
    Assert.True(result.CertificateValid);
    Assert.True(result.LicenseValid);
}
```

### 7.2 集成测试

```csharp
[Fact]
public async Task SubmitWeighingRecord_WithPfxCertificate_ShouldSucceed()
{
    // Arrange
    var client = _factory.WithWebHostBuilder(builder =>
    {
        builder.ConfigureServices(services =>
        {
            services.AddSingleton<IPfxCertificateManager,
                TestPfxCertificateManager>();
        });
    }).CreateClient();

    // Act
    var response = await client.PostAsync("/api/urban/weighing-records",
        new StringContent(jsonContent, Encoding.UTF8, "application/json"));

    // Assert
    response.EnsureSuccessStatusCode();
}
```

---

**实现优先级**：高
**预估工期**：6-8 周
**风险等级**：中等（需要证书管理和分发机制）