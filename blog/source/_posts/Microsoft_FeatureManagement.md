---
title: Microsoft.FeatureManagement Feature Toggle 
date: 2023-01-01 12:20:48
tags:
  - 2023
  - .Net Core
  - FeatureToggle

categories:
  - [FeatureToggle]


keywords: 

  - FeatureToggle

---
# ASP.NET Core Feature Flags #

ASP.NET Core提供了一種動態打開或關閉功能的方法。  
 * Github: <https://github.com/microsoft/FeatureManagement-Dotnet>  
 * Introducing Microsoft.FeatureManagement:<https://andrewlock.net/introducing-the-microsoft-featuremanagement-library-adding-feature-flags-to-an-asp-net-core-app-part-1/>
 * 軟體主廚的程式料理廚房:<https://dotblogs.com.tw/supershowwei/2020/11/23/180548>
## Nuget Package

```PowerShell
     Microsoft.FeatureManagement
     Microsoft.FeatureManagement.AspNetCore
```

---

## Registration

### IServiceCollection 服務註冊
    
```c#
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddFeatureManagement();
              
        services.AddControllersWithViews();
    }
```
### appsetting.json設定
    
```json
{ 
    "FeatureManagement":
    {
        "FeatureToggle": true,
        "ActionFilter": true,
        "MiddlewareFilter": true
    }
}
```

## Consumption

### Use FeatureManager

+ 建構子注入`IFeatureManager`介面，調用`IsEnabledAsync`方法，確認功能是否啟用。
```c#
    public class HomeController : Controller
    {
        private readonly IFeatureManager _featureManager;
        
        public HomeController(IFeatureManager featureManager)
        {
            this._featureManager = featureManager;
        }
        
        public async Task<IActionResult> FeatureToggle()
        {
            bool isEnabled = await this._featureManager.IsEnabledAsync("FeatureToggle");
            return Content(isEnabled);
        }
    }
```
### Use FeatureGateAttribute
 + 針對Controller及Action可以使用`FeatureGateAttribute`，達到控制是否啟用該功能

```c#
        [FeatureGate("ActionFilter")]
        public IActionResult ActionFilter()
        {
            return Content("Home.ActionFilter");
        }
```
### Use MiddlewareForFeature
 + 針對middleware想要做到是否啟用，可以使用`IApplicationBuilder`擴充方法`UseForFeature`或是`UseMiddlewareForFeature`

```c#
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
               
            app.UseForFeature("MiddlewareFilter", appBuilder =>
            {
                appBuilder.UseHealthChecks("/health");
            });       
        }
```
---
## 進階應用

### FeatureFilter
 依照條件啟動功能標記，`Microsoft.FeatureManagement`提供內建三種`IFeatureFilter`實作。
    
#### 1.PercentageFilter 百分比功能標記
 依照百分比的機率啟動功能標記
```json
{
  "FeatureManagement": {
    "FeatureA": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": {
            "Value": 50
          }
        }
      ]
    }
  }
}
```
Startup.cs 服務註冊
```c#
public class Startup
{
  public void ConfigureServices(IServiceCollection services)
  {
      services.AddFeatureManagement()
              .AddFeatureFilter<PercentageFilter>();
  }
}
```

#### TimeWindowFilter 時間功能標記
 在指定時間內啟動功能標記
```json
{
  "FeatureManagement": {
    "FeatureB": {
      "EnabledFor": [
        {
          "Name": "TimeWindow",
          "Parameters": {
            "Start": "Wed, 01 May 2023 13:59:59 GMT",
            "End": "Mon, 01 July 2023 00:00:00 GMT"
          }
        }
      ]
    }
  }
}
```
Startup.cs 服務註冊
```c#
public class Startup
{
  public void ConfigureServices(IServiceCollection services)
  {
      services.AddFeatureManagement()
              .AddFeatureFilter<TimeWindowFilter>();
  }
}
```

#### TargetingFilter 定位受眾功能標記
 在指定目標對象啟動功能標記，在Github有更詳細的說明。<https://github.com/microsoft/FeatureManagement-Dotnet/blob/main/README.md#Targeting>
```json
{
  "FeatureManagement": {
    "FeatureB": {
      "EnabledFor": [
        {
          "Name": "Microsoft.Targeting",
            "Parameters": {
                "Audience": {
                    "Users": [
                        "Jeff",
                        "Alicia"
                    ],
                    "Groups": [
                        {
                            "Name": "Ring0",
                            "RolloutPercentage": 100
                        },
                        {
                            "Name": "Ring1",
                            "RolloutPercentage": 50
                        }
                    ],
                    "DefaultRolloutPercentage": 20
                }
            }
        }
      ]
    }
  }
}
```
還要額外去實作`ITargetingContextAccessor.cs`Interface
```c#
    /// <summary>
    /// Provides an implementation of <see cref="ITargetingContextAccessor"/> that creates a targeting context using info from the current HTTP request.
    /// </summary>
    public class HttpContextTargetingContextAccessor : ITargetingContextAccessor
    {
        private const string TargetingContextLookup = "HttpContextTargetingContextAccessor.TargetingContext";
        private readonly IHttpContextAccessor _httpContextAccessor;

        public HttpContextTargetingContextAccessor(IHttpContextAccessor httpContextAccessor)
        {
            _httpContextAccessor = httpContextAccessor ?? throw new ArgumentNullException(nameof(httpContextAccessor));
        }

        public ValueTask<TargetingContext> GetContextAsync()
        {
            HttpContext httpContext = _httpContextAccessor.HttpContext;

            //
            // Try cache lookup
            if (httpContext.Items.TryGetValue(TargetingContextLookup, out object value))
            {
                return new ValueTask<TargetingContext>((TargetingContext)value);
            }

            ClaimsPrincipal user = httpContext.User;

            List<string> groups = new List<string>();

            //
            // This application expects groups to be specified in the user's claims
            foreach (Claim claim in user.Claims)
            {
                if (claim.Type == ClaimTypes.GroupName)
                {
                    groups.Add(claim.Value);
                }
            }

            //
            // Build targeting context based off user info
            TargetingContext targetingContext = new TargetingContext
            {
                UserId = user.Identity.Name,
                Groups = groups
            };

            //
            // Cache for subsequent lookup
            httpContext.Items[TargetingContextLookup] = targetingContext;

            return new ValueTask<TargetingContext>(targetingContext);
        }
    }
```

Startup.cs 服務註冊
```c#
public class Startup
{
  public void ConfigureServices(IServiceCollection services)
  {

    services.AddSingleton<ITargetingContextAccessor, HttpContextTargetingContextAccessor>();

    services.AddFeatureManagement();
            .AddFeatureFilter<TargetingFilter>();
  }
}
```
#### 自訂FeatureFilter   
 針對複雜情形可以自己去實作`IFeatureFilter`，達到符合特定條件啟用功能。

  appsetting.json設定
```json
{ 
    "FeatureManagement":
    {
        "RequestFilter": {
            "EnabledFor": [
            {
                "Name": "RestrictRequest",
                "Parameters": {
                    "WhiteListIps": [   
                        ],
                    "RestrictPaths": [                  
                    ]
                }
            }]
        }
    }}
```
需要建立類別實作`IFeatureFilter`介面

```c#
    [FilterAlias("RestrictRequest")]
    public class RequestFilter : IFeatureFilter
    {
        private readonly IHttpContextAccessor _httpContextAccessor;

        public RequestFilter(IHttpContextAccessor httpContextAccessor)
        {
            _httpContextAccessor = httpContextAccessor;
        }
        
        public Task<bool> EvaluateAsync(FeatureFilterEvaluationContext context)
        {
            var httpContext = this._httpContextAccessor.HttpContext;
            var requestFilterOption = context.Parameters.Get<RequestFilterSetting>();
            var request = httpContext.Request;

            var isPathInRestricts = requestFilterOption.RestrictPaths.Any
            (
                restrictPath => request.Path.StartsWithSegments
                (
                    $"/{restrictPath}",
                    StringComparison.OrdinalIgnoreCase
                )
            );
            
            if (isPathInRestricts.Equals(true))
            {
                var clientIp = // your ip;

                var ipAddress = httpContext.Connection.RemoteIpAddress.MapToIPv4().ToString();

                ipAddress = string.IsNullOrWhiteSpace(clientIp)
                    ? ipAddress
                    : clientIp;

                var isIpInWhiteList = requestFilterOption.WhiteListIps.Any
                (
                    whiteListIp => ipAddress.StartsWith(whiteListIp)
                );
                
                return Task.FromResult(isIpInWhiteList);
            }

            return Task.FromResult(true);
        }
    }
```
 Startup.cs 服務註冊
```c#
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddHttpContextAccessor();
            
            services.AddFeatureManagement()                 
                    .AddFeatureFilter<RequestFilter>();
                     
            services.AddControllersWithViews();
        }
```

使用上可用`IFeatureManager`或是`FeatureGateAttribute`

```c#
        [FeatureGate("RequestFilter")]
        public IActionResult RequestFilter()
        {
            return Content("Home.RequestFilter");
        }
```
