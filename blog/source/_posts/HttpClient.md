---
title: 關於 HttpClient 使用方式
date: 2023-11-27 21:32:48
tags:
  - 2023
  - .Net Core


categories:
  - [WebAPI]
  - [HTTP]

keywords: 
  - HTTP
  - WebAPI


---

# 關於 HttpClient


因為許多企業大量採用微服務，導致使用 HTTP 連線非常頻繁。在**CSharp**中最常使用的類別庫就是`HttpClient.cs`，但是`HttpClient.cs`使用方式也是需要注意，一不小心就會造成Connection資源耗盡的問題。


## 官方不建議使用方式

<br>

### 直接使用```using``` HttpClient



```csharp!

static async Task Main()
{
	var url = "http://localhost:5259/WeatherForecast";
    
	int requestCount = 10000; // 設定請求的次數

	for (int i = 0; i < requestCount; i++)
	{
		using (var client = new HttpClient())
		{
			try
			{
				var response = await client.GetAsync(url);
				
				response.EnsureSuccessStatusCode();
			}
			catch (Exception ex)
			{
				Console.WriteLine($"An error occurred: {ex.Message}");
			}
		}
	}

	Console.WriteLine("Finish!!");
}

```

建立一支API 讓這段程式碼可以去Connection，很有可能會發生以下的錯誤。

```csharp!

An error occurred: System.Net.Http.HttpRequestException: 一次只能用一個通訊端位址 (通訊協定/網路位址/連接埠)。 (localhost:5259)
 ---> System.Net.Sockets.SocketException (10048): 一次只能用一個通訊端位址 (通訊協定/網路位址/連接埠)。
   at System.Net.Sockets.Socket.AwaitableSocketAsyncEventArgs.ThrowException(SocketError error, CancellationToken cancellationToken)
   at System.Net.Sockets.Socket.AwaitableSocketAsyncEventArgs.System.Threading.Tasks.Sources.IValueTaskSource.GetResult(Int16 token)
   at System.Net.Sockets.Socket.<ConnectAsync>g__WaitForConnectWithCancellation|281_0(AwaitableSocketAsyncEventArgs saea, ValueTask connectTask, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.ConnectToTcpHostAsync(String host, Int32 port, HttpRequestMessage initialRequest, Boolean async, CancellationToken cancellationToken)
   --- End of inner exception stack trace ---
    // 略
    
```

在官方文件有提到這一點

[.NET 中可用的原始 HttpClient 類別問題](https://learn.microsoft.com/zh-tw/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests#issues-with-the-original-httpclient-class-available-in-net)


> 雖然這個類別會實作 IDisposable，但不建議在 using 陳述式內加以宣告及具現化，因為處置 `HttpClient.cs` 物件時，底層通訊端並不會立即釋放，而這可能會導致「通訊端耗盡」問題。 


<br>



### 在Dependency Injection 中使用Singleton或是宣告靜態物件


> 開發人員遇到的另一個問題是在長時間執行的處理序中使用 HttpClient 的共用執行個體時。 在 HttpClient 具現化為 singleton 或靜態物件的情況下，其並無法處理 DNS 變更


例如:

```csharp!
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.

builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// 註冊成Singleton
builder.Services.AddSingleton<HttpClient>();

```


---

<br>
<br>


## 解決方式之一 使用`IHttpClientFactory.cs`

在 .Net Core引入`IHttpClientFactory.cs`介面去建立`HttpClient.cs`。
特點是可以由`IHttpClientFactory.cs`去管理HttpMessageHandler的生命週期，避免自行管理`HttpClient.cs`的生命週期。

### 基本使用方式

需要在Dependency Injection裡面加上了``` services.AddHttpClient();``` 

```csharp!

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.

builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// 加上這一行
builder.Services.AddHttpClient();

```


### 具名HttpClient

* 應用程式需要不同的HttpClient用法或是設定。


```csharp!

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.

builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// 加上這一行
builder.Services.AddHttpClient("YourName");

```

#### 調用方

```csharp!

public class Repository
{
    private readonly IHttpClientFactory _httpClientFactory;
    
    public Repository(IHttpClientFactory httpClientFactory)  
    {        
        this._httpClientFactory = httpClientFactory;        
    }
    
    public async Task<Todo[]> GetToDos()
    {
        string url = "your service url";
        
        using var client = _httpClientFactory.CreateClient("YourName");
        
        var response = await client.GetFromJsonAsync<Todo[]>(url,new JsonSerializerOptions(JsonSerializerDefaults.Web));  
		
		return response;
    }
}

```


