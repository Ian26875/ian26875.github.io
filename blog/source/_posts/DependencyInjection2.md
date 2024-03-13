---
title: Dependency Injection (二) Mircosoft.Extensions.DependencyInjection 介紹
date: 2023-11-27 21:12:48
tags:
  - 2023
  - .Net Core


categories:
  - [OOP,DI]


keywords: 
  - DI


---

# Microsoft.Extensions.DependencyInjection

在```ASP.NET Core```中，Dependency Injection Container 是一個核心功能，用於管理應用程序中的依賴關係。```ASP.NET Core``` 的 Dependency Injection Container系統提供了一個建立基於依賴注入的應用程序的機制，並支持以下功能：

* 提供應用程序組件之間的依賴注入。
* 可以在Controller、Filter、View等多個組件中使用。
* 提供了生命週期管理功能，可以在需要時建立、和釋放物件。
* 支持對服務的多個實現進行注入，並可根據需要在運行時選擇實現。

    >.NetCore DI 預設只支援建構子注入，但是Middleware支援方法注入。


<br>

## IServiceCollection 是什麼?

當查詢原始碼時，會看到下列的介面

```csharp!
public interface IServiceCollection : IList<ServiceDescriptor>
{


}
```

 ServiceDescriptor.cs
 
 
 ```csharp!
public class ServiceDescriptor 
{

    /// <inheritdoc />
    public ServiceLifetime Lifetime { get; }

    /// <inheritdoc />
    public Type ServiceType { get; }

    /// <inheritdoc />
    public Type ImplementationType { get; }

    /// <inheritdoc />
    public object ImplementationInstance { get; }

    /// <inheritdoc />
    public Func<IServiceProvider, object> ImplementationFactory { get; }
    
    /// ... 略


}

```

其實```IServiceCollection.cs```，其實只是```ServiceDescriptor.cs```的集合。

---

<br>

## Dependency Injection –生命週期

在```ServiceDescriptor.cs```其中的```ServiceLifetime.cs ```是Enum，主要負責進行依賴注入的物件生命週期。


```csharp!
    /// <summary>
    /// Specifies the lifetime of a service in an <see cref="IServiceCollection"/>.
    /// </summary>
    public enum ServiceLifetime
    {
        /// <summary>
        /// Specifies that a single instance of the service will be created.
        /// </summary>
        Singleton,
        
        /// <summary>
        /// Specifies that a new instance of the service will be created for each scope.
        /// </summary>
        /// <remarks>
        /// In ASP.NET Core applications a scope is created around each server request.
        /// </remarks>
        Scoped,
        
        /// <summary>
        /// Specifies that a new instance of the service will be created every time it is requested.
        /// </summary>
        Transient
    }
```

每種方式的生命週期管理方式都不同，需要根據具體的應用場景來選擇。
* Singleton：對象在整個應用程序生命週期內只會被創建一次，之後重複使用。
* Scoped：對象在範圍內只會被建立一次，同一個Request內所有需要使用該對象的Class都共享同一個對象。
* Transient：對象每次被注入時都會重新創建一個新的Instance。


<br>

## 使用 Dependency Injection 註冊方式



這邊介紹一下在開發上蠻常用的 Dependency Injection 註冊方式


1. `AddXXX<TService,TImplementation>()` 加入DI容器並直接決定生命週期。

```csharp!

var builder = WebApplication.CreateBuilder(args);

//建立註冊 生命週期是 Scoped
builder.Services.AddScoped<ISubjectRepository, SubjectRepository>();

```

2. `AddXXX<TService>(Func<IServiceProvider, object> implementationFactory)`


```csharp!

var builder = WebApplication.CreateBuilder(args);

//建立註冊 生命週期是 Scoped
builder.Services.AddScoped<ISubjectRepository>(serviceProvider=>
{
    return new SubjectRepository();
});

```


3. `TryAddXXX<TService,TImplementation>()` 加入DI容器，當發現`TService`已經有註冊了就不再進行註冊。


```csharp!

var builder = WebApplication.CreateBuilder(args);

//建立註冊 生命週期是 Scoped
builder.Services.AddScoped<ISubjectRepository>(serviceProvider=>
{
    return new SubjectRepository();
});

```


---

<br>

## 多個實作註冊相同的介面

單一介面使用單一實作這邊就不做介紹。多種實作應該是屬常見的情境。
當介面有多個實作時，則會採取最後一個實作。對於 `IServiceCollection.cs` 是後進先出。但可以取得該介面的所有實作。

```csharp!
public interface ISubjectRepository
{
    List<Subject> GetAll();
}
```

在實作部分可能會是DB或是API。

```csharp!

public class SubjectDbRepository : ISubjectRepository
{
    public List<Subject> GetAll()
    {
        //略
    }
}


public class SubjectApiRepository : ISubjectRepository
{
    public List<Subject> GetAll()
    {
        //略
    }
}

```

在DI註冊則是以下程式碼。

```csharp!

builder.Services.AddScoped<ISubjectRepository,SubjectDbRepository>();

builder.Services.AddScoped<ISubjectRepository,SubjectApiRepository>();
```
在這段程式碼中，當使用了 ```ISubjectRepository.cs``` ，DI容器會給予 ```SubjectApiRepository.cs``` 的實作。但是如果我們需要取得全部可以在建構子中使用 ```IEnumerable<ISubjectRepository>``` 的建構子注入。



```csharp!
public class SubjectRepositoryFactory : ISubjectRepositoryFactory
{
    private readonly IEnumerable<ISubjectRepository> _subjectRepositories;

    public SubjectRepositoryFactory(IEnumerable<ISubjectRepository> subjectRepositories)
    {
        _subjectRepositories = subjectRepositories;
    }
    
    public ISubjectRepository Create()
    {
        return CreateInstance<SubjectDbRepository>();
    }

    private ISubjectRepository CreateInstance<T>() where T : ISubjectRepository
    {
        return this._subjectRepositories.Single(sp => sp is T);
    }
}

```

在這個 `SubjectRepositoryFactory.cs` 建構子中可以取得 `ISubjectRepository.cs` 的所有實作。

---

<br>
<br>

## 結論


雖然微軟提供的Dependency Injection 容器很簡單，但是可以應付大部分場景。個人並不喜歡把DI容器使用的深入。不只後續抓錯誤麻煩。也會遇到執行錯誤不知道從哪邊抓。