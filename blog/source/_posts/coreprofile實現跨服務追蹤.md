---
title: CoreProfile 實現跨應用程式性能調整及監控
date: 2022-12-16 22:53:48
tags:
  - 2022
  - .Net Core
  - Monitor

categories:
  - [Monitor]


keywords: 
  - CoreProfile

---

## 適用情境:
- 當橫跨API服務應用程式之間請求與回應效能低下，需要API服務性能調教及追蹤效能。

## NuGet套件:
```PowerShell
Install-Package CoreProfiler
Install-Package CoreProfiler.Web
```

## 環境要求:
- WebService應用程式須安裝CoreProfile或NanoProfile，且DrillDown功能須設定為開啟，允許從外部應用程序向下鑽取子請求。

- Net.Core專案 Startup.cs 必須加上 app.UseCoreProfiler(true)。 (參數為true表示開啟跨應用程式，從外部應用程序向下取子請求drillDown功能)


```C#
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            app.UseCoreProfiler(true);

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseHttpsRedirection();

            app.UseRouting();

            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
```

## 實作內容:

------------
### 方法一 :  ProfilingSession.Current.WebTimingAsync

```C#
            // WebTimingAsync() profiles the wrapped action as a web request timing
            var url = this.Request.Scheme + "://" + this.Request.Host + "/home/child";
            await ProfilingSession.Current.WebTimingAsync(url, async (correlationId) =>
            {
                using (var httpClient = new HttpClient())
                {
                    httpClient.DefaultRequestHeaders.Add(CoreProfilerMiddleware.XCorrelationId, correlationId);

                    var uri = new Uri(url);
                    var result = await httpClient.GetStringAsync(uri);
                }
            });
```


* HttpClient發送前，必需要在HttpHeader加上CorrelationId，才能取得外部應用程式CoreProfiler或NanoProfiler的監控紀錄。

* HttpClient都需要加上以上的程式碼，會過於繁重，個人比較推薦使用第二種方法，亦可使用設定的方式進行移除或裝載功能。

------------
### 方法二 : HttpClientFactory 實作 DelegatingHandler

CoreProfilerDelegatingHandler.cs
```C#

    public class CoreProfilerDelegatingHandler : DelegatingHandler
    {   
        protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        {
            if (ProfilingSession.Current == null)
            {
                return await base.SendAsync(request, cancellationToken);
            }

            string url = request.RequestUri.AbsoluteUri;

            var webTiming = new WebTiming
            (
                profiler: ProfilingSession.Current.Profiler, 
                url: url
            );
            
            try
            {
                request.Headers.Add
                (
                    CoreProfilerMiddleware.XCorrelationId,
                    webTiming.CorrelationId
                );

                return await base.SendAsync(request, cancellationToken);
            }
            finally
            {
                webTiming.Stop();
            }
        }
    }
```


   ~/Startup.cs
```C#
  public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            //HttpClient 註冊可以用給予名稱進行設定
           services.AddTransient<CoreProfilerDelegatingHandler>()
                    .AddHttpClient("Tracing")
                    .AddHttpMessageHandler<CoreProfilerDelegatingHandler>();

            services.AddControllers();
        }

       
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            // 記得一定是ture
            app.UseCoreProfiler(true);

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

           
        }
    }
```

* 使用HttpClientFactory已註冊過使用的name，已經註冊的HttpClient已加入CoreProfilerDelegatingHandler。

```C#
    //必須要跟註冊的HttpClient的名稱一樣
    var httpClient = this._httpClientFactory.CreateClient("Tracing");

    var response = await httpClient.GetAsync(url).ConfigureAwait(false);
 
```




##  參考資料:

HttpClientFactory:
* https://docs.microsoft.com/zh-tw/aspnet/core/fundamentals/http-requests?view=aspnetcore-3.1

CoreProfiler:
* https://github.com/teddymacn/CoreProfiler
* https://github.com/teddymacn/cross-app-profiling-demo
* https://www.itread01.com/articles/1475204636.html