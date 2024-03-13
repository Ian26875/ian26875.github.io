---
title: MockServer 整合測試
date: 2024-01-14 21:19:48
tags:
  - 2024


categories:
  - [MockServer,Test]


keywords: 
  - Test


---

# Mock Server

由於要針對API 服務進行壓力測試，並假設其他服務有回應延遲，需要使用一套簡單使用的MockServer，原本想使用Postman，但礙於環境限制，於是找一套可以運行在Docker上的MockServer。


## Docker Install

首先執行 Docker 指令下載 Docker Image，並執行。

```shell
docker run -d --name mockserver -p 1080:1080 mockserver/mockserver
```


## MockServer UI

GET `http://localhost:1080/mockserver/dashboard`，在瀏覽器輸入這個網址便能看到 MockServer 的畫面。

從這邊可以看到以下的畫面。
![Dashboard](../static/images/MockServerUI.png)

## Create or Update Expectation

```csharp
void Main()
{
	
	var mockServerHost = "http://localhost:1080";

	using (var client = new HttpClient())
	{
		
        var mockServerUrl = $"{mockServerHost}/mockserver/expectation";
            
        var expectation = new MockServerExpectation
        {
			Id = "e5805f53-340b-4c9e-9399-aa0ac75266ee",
			Priority=0,
            HttpRequest = new HttpRequest
            {
                Method = "GET",
                Path = "/get/yourpath"
            },
            // 模擬回應
            HttpResponse = new HttpResponse
            {
                StatusCode = 200,
                Body = JsonConvert.SerializeObject
				(
					new 
                    {
                        Success = true
					}
                ),
                // 模擬延遲10秒回應
				Delay = new Delay
				{
					TimeUnit = "SECONDS",
					Value = 10
				}
			}
		};

		var jsonSetting = new JsonSerializerSettings 
		{
			NullValueHandling = NullValueHandling.Ignore
		};

		var json = JsonConvert.SerializeObject(expectation, jsonSetting);
        
		var content = new StringContent(json, Encoding.UTF8, "application/json");

		var response = client.PutAsync(mockServerUrl, content).GetAwaiter().GetResult();

		Console.WriteLine($"Response Status Code: {response.StatusCode}");
	}
}

public class MockServerExpectation
{
	[JsonProperty("id")]
	public string Id {get;set;}

	[JsonProperty("priority")]
	public int Priority {get;set;} = 0;
	
	[JsonProperty("httpRequest")]
	public HttpRequest HttpRequest { get; set; }
	
	[JsonProperty("httpResponse")]
	public HttpResponse HttpResponse { get; set; }
}

public class HttpRequest
{
	[JsonProperty("method")]
	public string Method { get; set; }
    
	[JsonProperty("path")]
	public string Path { get; set; }
}

public class HttpResponse
{
	[JsonProperty("statusCode")]
	public int StatusCode { get; set; }
    
	[JsonProperty("body")]
	public object Body { get; set; }
    
	[JsonProperty("delay")]
	public Delay Delay { get; set; }
}

public class Delay
{
	[JsonProperty("timeUnit")]
	public string TimeUnit { get; set; }
    
	[JsonProperty("value")]
	public int Value { get; set; }
}

```

Request 以下的 Content 給 MockServer，非常簡單使用。


```json
{
  "id": "e5805f53-340b-4c9e-9399-aa0ac75266ee",
  "priority": 0,
  "httpRequest": {
    "method": "GET",
    "path": "/get/yourpath"
  },
  "httpResponse": {
    "statusCode": 200,
    "body": "{\"Success\":true}",
    "delay": {
      "timeUnit": "SECONDS",
      "value": 10
    }
  }
}
```


使用上面的API 進行 建立修改 Expectation，之後將要Web服務指向這個Url即可進行使用。可以進行一些API異常測試等等。


## 使用情境

* **API 模擬**

有時間我們需要測試串接第三方API，尤其對方還尚未提供測試環境。

* **錯誤處理模擬**

測試應用程序如何處理各種錯誤 Response，如 4xx 和 5xx 狀態碼。

* **延時和性能測試**

模擬回應延遲和服務器處理時間，這對於測試應用程序在不同條件下的性能非常有用。

---

MockServer 非常適合於在開發階段進行測試，尤其當你需要與外部系統整合但該系統尚未準備。也可以用於自動化測試，提供一個可控且可預測的外部系統環境。

原本的應用程式因為要測試情境，採用appsettings.json去控制測試情境。千萬不要把這種程式碼寫在Production Code Commit。


## 參考資料

* [MockServer入門：輕鬆整合前後端接口](https://www.tpisoftware.com/tpu/articleDetails/1895)
* [MockServer 官方文件](https://www.mock-server.com/)