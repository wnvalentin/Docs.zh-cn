---
title: 在 ASP.NET Core 依赖注入
author: ardalis
description: 了解 ASP.NET Core 如何实现依赖注入和如何使用它。
ms.author: riande
ms.custom: H1Hack27Feb2017
ms.date: 10/14/2016
uid: fundamentals/dependency-injection
ms.openlocfilehash: 04c52bd47d34cd2135753c469077b6a75ee02f86
ms.sourcegitcommit: a1afd04758e663d7062a5bfa8a0d4dca38f42afc
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 06/20/2018
ms.locfileid: "36278448"
---
# <a name="dependency-injection-in-aspnet-core"></a>ASP.NET Core 中的依赖注入

<a name="fundamentals-dependency-injection"></a>

作者：[Steve Smith](https://ardalis.com/) 和 [Scott Addie](https://scottaddie.com)

ASP.NET Core 的设计从头至尾以支持和利用依赖注入为目标。 ASP.NET Core 应用程序可以通过将内置框架服务注入 Startup 类的方法中来对其加以利用，应用程序服务也可以进行注入配置。 由 ASP.NET Core 提供的默认服务容器提供了一个最小功能集合，并非用于替换其他容器。

[查看或下载示例代码](https://github.com/aspnet/Docs/tree/master/aspnetcore/fundamentals/dependency-injection/sample)（[如何下载](xref:tutorials/index#how-to-download-a-sample)）

## <a name="what-is-dependency-injection"></a>什么是依赖注入？

依赖注入 (DI) 是一种用于在对象与其协作者或依赖项之间实现松散耦合的技术。 该技术不是直接实例化协作者或使用静态引用，而是以某种方式向类提供该类执行其操作所需的对象。 大多数情况下，类通过其构造函数声明它们的依赖项，从而允许它们遵循[显式依赖关系原则](http://deviq.com/explicit-dependencies-principle/)。 此方法称为"构造函数注入"。

在遵循 DI 原则来设计类时，由于它们与其协作者没有直接的，硬编码的依赖关系，所以它们之间的耦合更为松散。 这遵循[依赖倒置原则](http://deviq.com/dependency-inversion-principle/)，其中指出 *"高级模块不应依赖于低级模块; 同时两者都应依赖于抽象"。* 类请求在该类被构造时提供给它们的抽象（通常是 `interfaces`），而不是引用特定的实现。 将依赖关系提取到接口中并将这些接口的实现作为参数提供也是[策略设计模式](http://deviq.com/strategy-design-pattern/) 的一个例子。

当系统被设计为使用 DI 时，许多类通过其构造函数 （或属性）来请求它们的依赖项，所以有一个专门用来创建这些类以及它们相关的依赖项的类是很有帮助的。 这些类统称为*容器*，或更具体地说，[控制反转 (IoC)](http://deviq.com/inversion-of-control/) 容器或依赖注入 (DI) 容器。 容器本质上是一个工厂，它负责提供从中请求的类型的实例。 如果给定的类型已声明它具有依赖项，并且容器已经被配置为提供这些依赖类型，那么它将在创建请求实例时创建依赖项。 通过这种方式，可以为类提供复杂的依赖关系图，而无需任何硬编码对象构造。 除了创建包含依赖项的对象外，容器通常还可以管理应用程序中的对象生命周期。

ASP.NET Core 包含一个简单的内置容器 (表示为 `IServiceProvider` 接口)，默认情况下，该容器支持构造函数注入，ASP.NET 可通过 DI 提供某些服务。 ASP.NET 的容器指其作为 *服务* 管理的类型。 本文其余部分中，*服务* 指的是由 ASP.NET Core 的 IoC 容器管理的类型。 您可以在应用程序的 `Startup` 类中的 `ConfigureServices` 方法中配置内置容器的服务。

> [!NOTE]
> Martin Fowler 写了另一篇全面介绍 [控制反转容器和依赖注入模式](https://www.martinfowler.com/articles/injection.html) 的文章。 Microsoft 模式和实践也对 [依赖关系注入](https://msdn.microsoft.com/library/hh323705.aspx) 进行了很好的描述。

> [!NOTE]
> 本文介绍了适用于所有 ASP.NET 应用程序的依赖注入。 [依赖关系注入和控制器](../mvc/controllers/dependency-injection.md)中介绍了 MVC 控制器内的依赖关系注入。

### <a name="constructor-injection-behavior"></a>构造函数注入行为

构造函数注入要求相关构造函数是公共的。 否则，应用程序会引发 `InvalidOperationException` 异常:

> 找不到适合类型 YourType 的构造函数。 请确保该类型是具体的，并为公共构造函数的所有参数注册服务。

构造函数注入要求只存在一个适用的构造函数。 支持构造函数重载，但其参数可以全部通过依赖注入来实现的重载只能存在一个。 如果存在多个，应用程序将引发 `InvalidOperationException` 异常:

> 已在类型 YourType 中找到多个接受所有给定的参数类型的构造函数。 应该只有一个适用的构造函数。

构造函数可以接受依赖注入没有提供的参数，但这些参数必须支持默认值。 例如:

```csharp
// throws InvalidOperationException: Unable to resolve service for type 'System.String'...
public CharactersController(ICharacterRepository characterRepository, string title)
{
    _characterRepository = characterRepository;
    _title = title;
}

// runs without error
public CharactersController(ICharacterRepository characterRepository, string title = "Characters")
{
    _characterRepository = characterRepository;
    _title = title;
}
```

## <a name="using-framework-provided-services"></a>使用框架提供的服务

`Startup` 类中的 `ConfigureServices` 方法负责定义应用程序将使用的服务，包括 Entity Framework Core 和 ASP.NET Core MVC 之类的平台功能。 最初，提供给 `ConfigureServices` 的 `IServiceCollection` 定义了以下服务（具体取决于[配置主机的方式](xref:fundamentals/host/index)）：

| 服务类型 | 生存期 |
| ----- | ------- |
| [Microsoft.AspNetCore.Hosting.IHostingEnvironment](/dotnet/api/microsoft.aspnetcore.hosting.ihostingenvironment) | 单一实例 |
| [Microsoft.Extensions.Logging.ILoggerFactory](/dotnet/api/microsoft.extensions.logging.iloggerfactory) | 单一实例 |
| [Microsoft.Extensions.Logging.ILogger&lt;T&gt;](/dotnet/api/microsoft.extensions.logging.ilogger) | 单一实例 |
| [Microsoft.AspNetCore.Hosting.Builder.IApplicationBuilderFactory](/dotnet/api/microsoft.aspnetcore.hosting.builder.iapplicationbuilderfactory) | 暂时 |
| [Microsoft.AspNetCore.Http.IHttpContextFactory](/dotnet/api/microsoft.aspnetcore.http.ihttpcontextfactory) | 暂时 |
| [Microsoft.Extensions.Options.IOptions&lt;T&gt;](/dotnet/api/microsoft.extensions.options.ioptions-1) | 单一实例 |
| [System.Diagnostics.DiagnosticSource](https://docs.microsoft.com/dotnet/core/api/system.diagnostics.diagnosticsource) | 单一实例 |
| [System.Diagnostics.DiagnosticListener](https://docs.microsoft.com/dotnet/core/api/system.diagnostics.diagnosticlistener) | 单一实例 |
| [Microsoft.AspNetCore.Hosting.IStartupFilter](/dotnet/api/microsoft.aspnetcore.hosting.istartupfilter) | 暂时 |
| [Microsoft.Extensions.ObjectPool.ObjectPoolProvider](/dotnet/api/microsoft.extensions.objectpool.objectpoolprovider) | 单一实例 |
| [Microsoft.Extensions.Options.IConfigureOptions&lt;T&gt;](/dotnet/api/microsoft.extensions.options.iconfigureoptions-1) | 暂时 |
| [Microsoft.AspNetCore.Hosting.Server.IServer](/dotnet/api/microsoft.aspnetcore.hosting.server.iserver) | 单一实例 |
| [Microsoft.AspNetCore.Hosting.IStartup](/dotnet/api/microsoft.aspnetcore.hosting.istartup) | 单一实例 |
| [Microsoft.AspNetCore.Hosting.IApplicationLifetime](/dotnet/api/microsoft.aspnetcore.hosting.iapplicationlifetime) | 单一实例 |

下面的示例介绍了如何使用多个扩展方法（如 `AddDbContext`、`AddIdentity` 和 `AddMvc`）将附加服务添加到容器中。

[!code-csharp[](../common/samples/WebApplication1/Startup.cs?highlight=5-6,8-10,12&range=39-56)]

ASP.NET 提供的功能和中间件（如 MVC），遵循使用单个 AddServiceName 扩展方法注册该功能请求的所有服务的约定。

> [!TIP]
> 你可以通过其参数列表在 `Startup` 方法内请求某些框架提供的服务，请参阅[应用程序启动](startup.md)了解详细信息。

## <a name="registering-services"></a>注册服务

可以按如下方式注册自己的应用程序服务。 第一个泛型类型表示将从容器请求的类型（通常是一个接口）。 第二个泛型类型表示将由容器实例化并用于满足此类请求的具体类型。

[!code-csharp[](../common/samples/WebApplication1/Startup.cs?range=53-54)]

> [!NOTE]
> 每个 `services.Add<ServiceName>` 扩展方法添加（并可能配置）服务。 例如，`services.AddMvc()` 添加 MVC 需要的服务。 建议遵循此约定，将扩展方法放置在 `Microsoft.Extensions.DependencyInjection` 命名空间中，以封装服务注册的组。

`AddTransient` 方法用于将抽象类型映射到为每个需要它的对象分别实例化的具体服务。 这称为服务的 *生存期*，其他生存期选项如下所述。 为注册的每个服务选择适当的生存期非常重要。 是否应该向每个请求它的类提供一个新的服务实例？ 是否在整个给定的 Web 请求中使用一个实例？ 或是否应该在应用程序生存期内使用单例？

在本文示例中，有一个显示字符名称的简单控制器，名为 `CharactersController`。 其 `Index` 方法会显示存储在应用程序中的当前字符列表，如果不存在，则使用少量字符初始化该集合。 请注意，尽管此应用程序为其持久化使用Entity Framework Core和 `ApplicationDbContext`，但在控制器中没有任何明显的表现。 相反，特定的数据访问机制已经被抽象为一个接口，即 `ICharacterRepository`，它遵循 [仓储模式](http://deviq.com/repository-pattern/)。 通过构造函数请求 `ICharacterRepository` 实例，并将其分配给私有字段，然后根据需要使用它访问字符。

[!code-csharp[](../fundamentals/dependency-injection/sample/DependencyInjectionSample/Controllers/CharactersController.cs?highlight=3,5,6,7,8,14,21-27&range=8-36)]

`ICharacterRepository` 定义了控制器使用 `Character` 实例时需要的两种方法。

[!code-csharp[](../fundamentals/dependency-injection/sample/DependencyInjectionSample/Interfaces/ICharacterRepository.cs?highlight=8,9)]

此接口又由一个具体类型实现，即在运行时使用的 `CharacterRepository`。

> [!NOTE]
> DI 与 `CharacterRepository` 类一起使用的方式是一个通用模型，您可以为所有应用程序服务遵循此模型，而不仅仅在“仓储库”或数据访问类中。

[!code-csharp[](../fundamentals/dependency-injection/sample/DependencyInjectionSample/Models/CharacterRepository.cs?highlight=9,11,12,13,14)]

请注意，`CharacterRepository` 在其构造函数中请求一个 `ApplicationDbContext`。 依赖注入以这种链式方式被使用并不罕见，（每个被请求的依赖进而请求它自己的依赖）。 容器将负责解决图中所有的依赖关系，并返回完全解析后的服务。

> [!NOTE]
> 创建请求的对象，和它需要的所有对象，以及这些对象需要的所有对象，这有时被称为 *对象图*。 同样，必须被解析的依赖关系的集合通常被称为*依赖关系树*或*依赖项关系图*。

在这种情况下，`Startup` 和 `ApplicationDbContext` 必须在 `ICharacterRepository` 类的 `ConfigureServices` 中的服务容器中注册。 `ApplicationDbContext` 通过调用扩展方法 `AddDbContext<T>` 进行配置。 下面的代码演示了 `CharacterRepository` 类型的注册。

[!code-csharp[](dependency-injection/sample/DependencyInjectionSample/Startup.cs?highlight=3-5,11&range=16-32)]

应使用 `Scoped` 生命周期将实体框架上下文添加到服务容器中。 如果您使用上面所示的帮助方法，则会自动执行此操作。 使用实体框架的仓储库应使用相同的生命周期。

> [!WARNING]
> 需要注意的主要危险是从单例解析 `Scoped` 服务。 在此类情况下，当处理后续请求时，服务可能会处于不正确的状态。

具有依赖关系的服务应在容器中对它们进行注册。 如果服务构造函数需要一个基元（如 `string`），可使用[配置](xref:fundamentals/configuration/index)和[选项模式](xref:fundamentals/configuration/options)将其注入。

## <a name="service-lifetimes-and-registration-options"></a>服务生存期和注册选项

可使用以下生存期配置 ASP.NET 服务：

**暂时**

暂时生存期服务是每次请求时创建的。 这种生存期适合轻量级、 无状态的服务。

**作用域（Scoped）**

作用域生存期服务以每个请求一次的方式创建。

> [!WARNING]
> 如果在中间件内使用有作用域的服务，请将该服务注入至 `Invoke` 或 `InvokeAsync` 方法。 请不要通过构造函数注入进行注入，因为它会强制服务的行为与单一实例类似。

**单例**

单例生存期服务在第一次被请求时或者 `ConfigureServices` 运行时（如果在其中指定实例）创建，然后每个后续请求使用同一实例。 如果应用程序需要单例行为，建议允许服务容器管理服务的生存期，而不是实现单例设计模式并在类中自行管理对象的生存期。

服务可以使用多种方式注册到容器中。 我们已经看到了如何通过指定要使用的具体类型来注册一个给定类型的服务实现。 此外，可以指定一个工厂，然后将其用于按需创建实例。 第三种方法是直接指定要使用的类型的实例，在这种情况下，容器将永远不会尝试创建实例（也不会释放实例）。

为了演示这些生存期和注册选项之间的差异，请考虑一个简单的接口，将一个或多个任务表示为具有唯一标识符 `OperationId` 的*操作*。 根据此服务的生存期配置方式，容器将向请求的类提供相同或不同的实例。 为了明确正在请求哪个生存期，我们将为每个生命周期选项创建一个类型：

[!code-csharp[](../fundamentals/dependency-injection/sample/DependencyInjectionSample/Interfaces/IOperation.cs?highlight=5-8)]

我们使用单个类 `Operation` 来实现这些接口，它在构造函数中接受一个 `Guid`，如果没有提供，则使用一个新的 `Guid`。

接下来，在 `ConfigureServices` 中，根据其命名的生存期，将每个类型添加到容器中：

[!code-csharp[](dependency-injection/sample/DependencyInjectionSample/Startup.cs?range=26-32)]

请注意，`IOperationSingletonInstance` 服务正在使用一个具有已知 ID `Guid.Empty` 的特定实例，因此可以明确何时在使用此类型（其 Guid 将全部为零）。 我们已注册了依赖于每个其他 `Operation` 类型的 `OperationService`，因此对每个 Operation 类型来说，可以在请求中明确该服务是会获得与控制器相同的实例，还是获得一个新实例。 此服务的全部作用就是将其依赖项作为属性公开，以便它们可以显示在视图中。

[!code-csharp[](dependency-injection/sample/DependencyInjectionSample/Services/OperationService.cs)]

为了演示对应用程序的各个独立请求之内和之间的对象生存期，该示例包含一个`OperationsController`，它请求每种 `IOperation` 类型以及一个 `OperationService`。 `Index` 操作随后显示所有控制器和服务的 `OperationId` 值。

[!code-csharp[](dependency-injection/sample/DependencyInjectionSample/Controllers/OperationsController.cs)]

现在对该控制器操作发起了两个单独的请求：

![Microsoft Edge 中运行的依赖关系注入示例 Web 应用程序的操作视图对第一个请求显示暂时、限定、单一实例、实例控制器和操作服务操作的操作 ID 值 (GUID)。](dependency-injection/_static/lifetimes_request1.png)

![操作视图对第二个请求显示操作 ID 值。](dependency-injection/_static/lifetimes_request2.png)

观察请求内和请求之间哪些 `OperationId` 值不同。

* 暂时对象始终不同；向每个控制器和每项服务提供一个新的实例。

* 限定对象在请求内是相同的，但在不同的请求中是不同的

* *单例* 对象对每个对象和每个请求都是相同的 (不管 `ConfigureServices` 是否提供了一个实例)

## <a name="resolve-a-scoped-service-within-the-application-scope"></a>解析应用程序作用域中有作用域的服务

采用 [IServiceScopeFactory.CreateScope](/dotnet/api/microsoft.extensions.dependencyinjection.iservicescopefactory.createscope) 创建 [IServiceScope ](/dotnet/api/microsoft.extensions.dependencyinjection.iservicescope)，以解析应用作用域中有作用域的服务。 此方法可以用于在启动时访问有作用域的服务以便运行初始化任务。 以下示例演示如何在 `Program.Main` 中获取 `MyScopedService` 的上下文：

```csharp
public static void Main(string[] args)
{
    var host = BuildWebHost(args);

    using (var serviceScope = host.Services.CreateScope())
    {
        var services = serviceScope.ServiceProvider;

        try
        {
            var serviceContext = services.GetRequiredService<MyScopedService>();
            // Use the context here
        }
        catch (Exception ex)
        {
            var logger = services.GetRequiredService<ILogger<Program>>();
            logger.LogError(ex, "An error occurred.");
        }
    }

    host.Run();
}
```

## <a name="scope-validation"></a>作用域验证

如果在 ASP.NET Core 2.0 或更高版本上的开发环境中运行应用，默认的服务提供程序会执行检查，从而确认以下内容：

* 没有从根服务提供程序直接或间接解析到有作用域的服务。
* 未将有作用域的服务直接或间接注入到单一实例。

调用 [BuildServiceProvider](/dotnet/api/microsoft.extensions.dependencyinjection.servicecollectioncontainerbuilderextensions.buildserviceprovider) 时，会创建根服务提供程序。 在启动提供程序和应用时，根服务提供程序的生存期对应于应用/服务的生存期，并在关闭应用时释放。

有作用域的服务由创建它们的容器释放。 如果作用域创建于根容器，则该服务的生存会有效地提升至单一实例，因为根容器只会在应用/服务关闭时将其释放。 验证服务作用域，将在调用 `BuildServiceProvider` 时收集这类情况。

有关详细信息，请参阅 [Web 托管主题中的作用域验证](xref:fundamentals/host/web-host#scope-validation)。

## <a name="request-services"></a>请求服务

在 `HttpContext` 中的 ASP.NET 请求内可用的服务通过 `RequestServices` 集合进行公开。

![HttpContext 请求服务 Intellisense 上下文对话框指出请求服务可获取或设置对请求的服务容器提供访问权限的 IServiceProvider。](dependency-injection/_static/request-services.png)

请求服务表示你作为应用程序的一部分配置的服务和请求。 当对象指定依赖关系时，`RequestServices`（而不是 `ApplicationServices`）中的类型将满足这些要求。

通常情况下，不应直接使用这些属性，而应该通过类的构造函数来请求所需类的类型，并让框架注入这些依赖关系。 这将生成更易于测试且耦合更松散的类（请参阅[测试和调试](xref:test/index)）。

> [!NOTE]
> 与访问 `RequestServices` 集合相比，以构造函数参数的形式请求依赖项是更优先的选择。

## <a name="designing-services-for-dependency-injection"></a>设计能够进行依赖注入的服务

你应将自己的服务设计为可以使用依赖注入来获取其协作对象。 这意味着避免在服务中使用有状态的静态方法调用（这会导致称为[静态粘贴](http://deviq.com/static-cling/) 的代码异味）和依赖类的直接实例化。 在选择是实例化一个类型还是通过依赖注入来请求它时，记住这个短语可能会有所帮助：[新增即粘附](https://ardalis.com/new-is-glue)。 遵循[面向对象设计的SOLID原则](http://deviq.com/solid/)，你往往会获得小型、良好分解和易于测试的类。

如果发现你的类往往具有太多的依赖项需要注入怎么办？ 这通常表明你的类正试图做太多的事情，而且可能违反了 SRP-[单一责任原则](http://deviq.com/single-responsibility-principle/)。 看看是否可以通过将某些职责移动到一个新类来重构类。 请记住，你的 `Controller` 类应关注用户界面问题，因此业务规则和数据访问实现细节应保留在适用于这些[分离的关注点](http://deviq.com/separation-of-concerns/) 的类中。

具体就数据访问而言，可以将 `DbContext` 注入到你的控制器（假设你已向 `ConfigureServices` 中的服务容器中添加了 EF）。 一些开发人员更愿意使用数据库的存储库接口而不是直接注入 `DbContext`。 使用接口将数据访问逻辑封装于一处可以最大程序减少数据更改时需要更改的位置数量。

### <a name="disposing-of-services"></a>释放服务

容器将为其创建的 `IDisposable` 类型调用 `Dispose`。 但是，如果您自己将一个实例添加到容器，它将不会被释放。

示例:

```csharp
// Services implement IDisposable:
public class Service1 : IDisposable {}
public class Service2 : IDisposable {}
public class Service3 : IDisposable {}

public interface ISomeService {}
public class SomeServiceImplementation : ISomeService, IDisposable {}


public void ConfigureServices(IServiceCollection services)
{
    // container will create the instance(s) of these types and will dispose them
    services.AddScoped<Service1>();
    services.AddSingleton<Service2>();
    services.AddSingleton<ISomeService>(sp => new SomeServiceImplementation());

    // container didn't create instance so it will NOT dispose it
    services.AddSingleton<Service3>(new Service3());
    services.AddSingleton(new Service3());
}
```

> [!NOTE]
> 在 1.0 版中，容器将对*所有* `IDisposable` 对象调用 dispose，包括那些并非由它创建的对象。

## <a name="replacing-the-default-services-container"></a>替换默认服务容器

内置服务容器专门服务框架和基于其上生成的大多数应用程序的基本需求。 但是，开发人员可以使用自己喜欢的容器替换内置容器。 `ConfigureServices` 方法通常返回 `void`，但是，如果更改其签名以返回 `IServiceProvider`，则可以配置和返回不同的容器。 有许多可用于 .NET 的 IOC 容器。 在此示例中，将使用 [Autofac](https://autofac.org/) 包。

首先，安装适当的容器包：

* `Autofac`
* `Autofac.Extensions.DependencyInjection`

接下来，在 `ConfigureServices` 中配置容器并返回 `IServiceProvider`：

```csharp
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    // Add other framework services

    // Add Autofac
    var containerBuilder = new ContainerBuilder();
    containerBuilder.RegisterModule<DefaultModule>();
    containerBuilder.Populate(services);
    var container = containerBuilder.Build();
    return new AutofacServiceProvider(container);
}
```

> [!NOTE]
> 使用第三方 DI 容器时，必须更改 `ConfigureServices`，以便其返回 `IServiceProvider`，而不是 `void`。

最后，将 Autofac 在 `DefaultModule` 中配置为正常：

```csharp
public class DefaultModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        builder.RegisterType<CharacterRepository>().As<ICharacterRepository>();
    }
}
```

在运行时，将使用 Autofac 来解析类型，并注入依赖。 [了解有关使用 Autofac 和 ASP.NET Core的更多信息](http://docs.autofac.org/en/latest/integration/aspnetcore.html)。

### <a name="thread-safety"></a>线程安全

单例服务需要是线程安全的。 如果单例服务依赖于一个瞬时服务，那么瞬时服务可能也需要是线程安全的，具体取决于单例使用它的方式。

## <a name="recommendations"></a>建议

使用依赖注入时，请记住以下建议：

* DI 适用于具有复杂的依赖关系的对象。 控制器、 服务、 适配器和仓储都是可能添加到 DI 中的对象示例。

* 避免在 DI 中直接存储数据和配置。 例如，用户的购物车通常不应添加到服务容器中。 配置应使用 [选项模型](xref:fundamentals/configuration/options)。 同样，避免"数据持有者"对象，也就是仅仅为实现对某些其他对象的访问而存在的对象。 如果可能，最好通过 DI 请求所需的实际项目。

* 避免静态访问服务。

* 应用程序代码中避免服务位置。

* 避免静态访问 `HttpContext`。

如所有建议组合一样，你可能会遇到必须忽略一个的情况。 我们发现例外情况很罕见 - 大多数是框架本身内部非常特殊的情况。

依赖注入是静态/全局对象访问模式的替代方法。 如果将其与静态对象访问混合使用，则可能无法实现 DI 的优点。

## <a name="additional-resources"></a>其他资源

* [视图中的依赖关系注入](xref:mvc/views/dependency-injection)
* [控制器中的依赖关系注入](xref:mvc/controllers/dependency-injection)
* [要求处理程序中的依赖关系注入](xref:security/authorization/dependencyinjection)
* [应用程序启动](xref:fundamentals/startup)
* [测试和调试](xref:test/index)
* [基于工厂的中间件激活](xref:fundamentals/middleware/extensibility)
* [在 ASP.NET Core 中使用依赖关系注入编写干净代码 (MSDN) ](https://msdn.microsoft.com/magazine/mt703433.aspx)
* [Container-Managed Application Design, Prelude: Where does the Container Belong?](https://blogs.msdn.microsoft.com/nblumhardt/2008/12/26/container-managed-application-design-prelude-where-does-the-container-belong/)（容器托管的应用程序设计，序言：容器属于何处？）
* [Explicit Dependencies Principle](http://deviq.com/explicit-dependencies-principle/)（显式依赖关系原则）
* [Inversion of Control Containers and the Dependency Injection Pattern](https://www.martinfowler.com/articles/injection.html)（控制反转容器和依赖关系注入模式）
