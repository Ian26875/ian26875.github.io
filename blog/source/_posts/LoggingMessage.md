---
title: 更高效 Logging - LoggerMessage 
date: 2024-02-16 20:19:48
tags:
  - 2024
  - .Net Core
  - WebAPI

categories:
  - [WebAPI,Log]


keywords: 
  - Logging


---
# Logging - LoggerMessage 

原先在記錄Log都會使用微軟提供LoggingExtensions方法，最近朋友聊天分享給我`LoggerMessage.cs`與`LoggerMessageAttribute.cs`。於是研究一下並記錄一下。

這方式把寫Log的方式更好的管理及使用強型別方式。

下面提供兩種方式去改寫 Logging 也可以提升效能，官方文件提到以下原因。

> 相較於記錄器擴充方法，LoggerMessage 提供下列效能優勢：
> 記錄器擴充方法需要 "boxing" (轉換) 實值型別，例如將 int 轉換為 object。 LoggerMessage 模式可使用靜態 Action 欄位和擴充方法搭配強型別參數來避免 boxing。
> 記錄器擴充方法在每次寫入記錄訊息時，都必須剖析訊息範本 (具名格式字串)。 LoggerMessage 只需在定義訊息時剖析範本一次。


這是原本使用 `LoggingExtensions.cs`去寫 Logging，這邊要注意的是 EventId 是自定的數字。

```csharp
    [HttpGet(Name = "GetWeatherForecast")]
    public IEnumerable<WeatherForecast> Get()
    {
        _logger.LogInformation
		(
			new EventId(13,"CreateOrderFailure"),
			"create order failed !! order id : {orderId}",100
		);
        
        // 略
    }
```

<br>

---

## 第一種方式 - LoggerMessage 類別

LoggerMessage.Define 是回傳委派共提供最多六個參數。
這邊簡單建立類別。

* LogHelper.cs
```csharp
public static class LogHelper
{
    private static Action<ILogger, int, Exception> FailedToCreateOrder => LoggerMessage.Define<int>
    (
        LogLevel.Information,
        new EventId(13, nameof(CreateOrderFailure)),
        "create order failed !! order id : {order.Id}"
    );
    
    public static void CreateOrderFailure(this ILogger logger,Order order)
    {
        if (logger.IsEnabled(LogLevel.Information))
        {
            FailedToCreateOrder(logger, order.Id, default!);
        }
    }
}

```




* 調用方

```csharp
    [HttpGet(Name = "GetWeatherForecast")]
    public IEnumerable<WeatherForecast> Get()
    {
        // 使用擴充方法
        _logger.CreateOrderFailure(new Order
        {
            Id=100
        });
        
        // 略
    }
```

* 執行結果 : 

```
info: LogDefinition.Controllers.WeatherForecastController[13]
      create order failed !! order id : 100
```


## 第二種方式 - LoggerMessageAttribute 類別

這邊要注意的是 `LoggerMessageAttribute.cs` 類別是在 **.Net 6** 以後才引入的。

使用 LoggerMessageAttribute 時，必須遵守限制：

* 記錄方法必須為 partial 並傳回 void。
* 記錄方法名稱不可以底線開頭。
* 記錄方法的參數名稱不可以底線開頭。
* 記錄方法不可在巢狀型別中定義。
* 記錄方法不可為泛型。
* 如果記錄方法為 static，則須以 ILogger 執行個體作為參數。

這邊也建立 `LogHelperV2.cs` 類別

```csharp
public static partial class LogHelperV2
{
    [LoggerMessage(13, LogLevel.Information, "create order failed !! order id : {OrderId}", EventName = "CreateOrderFailureV2")]
    public static partial void CreateOrderFailureV2(this ILogger logger, int orderId);
}

```
* 調用方:

```csharp
	[HttpGet(Name = "GetWeatherForecast")]
    public IEnumerable<WeatherForecast> Get()
    {
    
        // 加上這一行
        _logger.CreateOrderFailureV2(100);
        
        // 略
    }
```


執行結果 : 

```
info: LogDefinition.Controllers.WeatherForecastController[13]
      create order failed !! order id : 100
```

這邊也提供官方範例作為範例。

* 官方範例 : [HttpsLoggingExtensions.cs](https://github.com/dotnet/aspnetcore/blob/9db62024cbe3c3cb28efe372541fc1bdfcdb375e/src/Middleware/HttpsPolicy/src/HttpsLoggingExtensions.cs)


```csharp
using Microsoft.Extensions.Logging;

namespace Microsoft.AspNetCore.HttpsPolicy;

internal static partial class HttpsLoggingExtensions
{
    [LoggerMessage(1, LogLevel.Debug, "Redirecting to '{redirect}'.", EventName = "RedirectingToHttps")]
    public static partial void RedirectingToHttps(this ILogger logger, string redirect);

    [LoggerMessage(2, LogLevel.Debug, "Https port '{port}' loaded from configuration.", EventName = "PortLoadedFromConfig")]
    public static partial void PortLoadedFromConfig(this ILogger logger, int port);

    [LoggerMessage(3, LogLevel.Warning, "Failed to determine the https port for redirect.", EventName = "FailedToDeterminePort")]
    public static partial void FailedToDeterminePort(this ILogger logger);

    [LoggerMessage(5, LogLevel.Debug, "Https port '{httpsPort}' discovered from server endpoints.", EventName = "PortFromServer")]
    public static partial void PortFromServer(this ILogger logger, int httpsPort);
}

```



## 資料來源

* https://learn.microsoft.com/zh-tw/dotnet/core/extensions/high-performance-logging
* https://learn.microsoft.com/zh-tw/dotnet/core/extensions/logger-message-generator