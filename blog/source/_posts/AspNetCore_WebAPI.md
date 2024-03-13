---
title: Asp.Net Core API Controller繼承
date: 2023-04-16 20:40:48
tags:
  - 2023
  - .Net Core
  - WebAPI

categories:
  - [WebAPI]


keywords: 
  - WebAPI

---
# .Net Core Web API – API Controller 的繼承

##  API Controller 繼承

建立新的API Controller類別時，繼承到底要繼承Controller還是ControllerBase? 這兩個類別有什麼差異?
當翻開原始碼時候可以發現兩個類別的繼承關係。

* 完整程式碼 : [ControllerBase](https://github.com/dotnet/aspnetcore/blob/main/src/Mvc/Mvc.Core/src/ControllerBase.cs)


```csharp!
[Controller]
public abstract class ControllerBase
{
    
    private ControllerContext? _controllerContext;
    private IModelMetadataProvider? _metadataProvider;
    private IModelBinderFactory? _modelBinderFactory;
    private IObjectModelValidator? _objectValidator;
    private IUrlHelper? _url;
    private ProblemDetailsFactory? _problemDetailsFactory;
    // 略
}
```



* 完整程式碼 : [Controller](https://github.com/dotnet/aspnetcore/blob/main/src/Mvc/Mvc.ViewFeatures/src/Controller.cs)

```csharp!

/// <summary>
/// A base class for an MVC controller with view support.
/// </summary>
public abstract class Controller : ControllerBase, IActionFilter, IAsyncActionFilter, IDisposable
{
    private ITempDataDictionary? _tempData;
    private DynamicViewData? _viewBag;
    private ViewDataDictionary? _viewData;
    // 略
}

```

這發現了Controller.cs只是比ControllerBase.cs多了許多View上類別。所以當需要開發WebAPI，繼承了ControllerBase，會多了不必要處理View的方法。所以開發WebAPI的時候，應繼承 ControllerBase.cs，當開發MVC的時候則繼承 Controller.cs


