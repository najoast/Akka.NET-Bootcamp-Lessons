We introduced `Akka.Hosting` briefly in “[Using Akka.Templates to Create New Projects](https://petabridge.com/bootcamp/lessons/unit-1/akka-templates/)” but in this lesson we’re going to start writing our own configuration with it and we’re going to learn how to leverage [Akka.DependencyInjection](https://getakka.net/articles/actors/dependency-injection.html#managing-lifecycle-dependencies-with-akkadependencyinjection) to inject dependencies into our actors.

We will also spend a little bit of time working with Akka.NET’s `Router` system for scaling up the level of parallelism with the `ParserActor`s.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Amm7jmTifX8" title="Tutorial: Creating Professional, Local Akka.NET Applications (Bootcamp 2.0 - Unit 1)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
_Starts at the appropriate timestamp for this lesson_

## Akka.Hosting: Akka.NET’s Pit of Success Pattern

[Akka.NET’s been in development since 2013](https://petabridge.com/blog/10-years-of-akkadotnet/) and there’s been _a lot_ of evolution in the .NET ecosystem since that time. Probably the biggest change is the introduction of .NET Core, which made .NET a cross-platform runtime, but nearly as big as that change was the introduction of the Microsoft.Extensions ecosystem - which helped bring about some common abstractions for cross-cutting concerns like configuration, logging, dependency injection, hosting, health checks, and more.

[Akka.Hosting](https://github.com/akkadotnet/Akka.Hosting) is Akka.NET’s integration with the Microsoft.Extensions ecosystem and more:

1. Automatically manages the lifecycle of your `ActorSystem` and ties it into the lifecycle of the `Microsoft.Extensions.Hosting` engine - if your app dies the `ActorSystem` terminates gracefully and vice-versa.
2. Ties Akka.NET into the `Microsoft.Extensions.Logging` ecosystem, which among other things, makes it very easy to ship Akka.NET `ILoggingAdapter` logs using [OpenTelemetry](https://opentelemetry.io/).
3. Introduces the `ActorRegistry`, a construct that makes it possible to inject actors via `Microsoft.Extensions.DependencyInjection` into both actors and non-actors via the `IRequiredActor<TActor>` type.
4. Replaces the need to write HOCON and allows users to configure Akka.NET via strongly-typed extension methods instead.

You can learn more about Akka.Hosting in our video: [No Hocon, No Lighthouse, No Problem: Akka.Hosting, Akka.HealthCheck, and Akka.Management](https://www.youtube.com/watch?v=U2IIfLKxprM)

<iframe width="560" height="315" src="https://www.youtube.com/embed/U2IIfLKxprM" title="No Hocon, No Lighthouse, No Problem: Akka.Hosting, Akka.HealthCheck, and Akka.Management" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
### Cleanup `AkkaWordCounter2`

The first thing that we’re going to do is delete some of the items `Akka.Templates` added to our application:

- Delete the `TimerActor.cs` file from `AkkaWordCounter2.App`;
- Delete the `HelloActor.cs` file from `AkkaWordCounter2.App`; and
- Delete the `TimerActor` and `HelloActor` initialization code inside `Program.cs`.

Your `Program.cs` should look like this once you’re finished:

```cs
using Akka.Hosting;
using Microsoft.Extensions.Hosting;

var hostBuilder = new HostBuilder();


hostBuilder
    .ConfigureServices((context, services) => {
        services.AddAkka("MyActorSystem", (builder, sp) => { });
    });

var host = hostBuilder.Build();

await host.RunAsync();
```

### Registering Services

One set of services we know we’re going to need from the start is the `IHttpClientFactory` because the `ParserActor` depends on it.

Open `Program.cs` and inside our `ConfigureServices` method please add the following call:

```cs
services.AddHttpClient(); // needed for IHttpClientFactory
```

Our `Program.cs` should now look like:

```cs
using Akka.Hosting;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection; // added as part of AddHttpClient

var hostBuilder = new HostBuilder();

hostBuilder
    .ConfigureServices((context, services) => {
        services.AddHttpClient();
        services.AddAkka("MyActorSystem", (builder, sp) => { });
    });

var host = hostBuilder.Build();

await host.RunAsync();
```

This is the only non-Akka.NET service our actors require at this stage.

### Configuring Logging

The next thing we’re going to do is leverage [Akka.Hosting’s `Microsoft.Extensions.Logging` capability](https://github.com/akkadotnet/Akka.Hosting?tab=readme-ov-file#microsoftextensionslogging-integration).

Open `Program.cs` and add the following code to the `AddAkka` method:

```cs
builder
    .ConfigureLoggers(logConfig => {
        logConfig.AddLoggerFactory();
    });
```

The entire `AddAkka` method should look like this:

```cs
services.AddAkka("MyActorSystem", (builder, sp) => {
    builder
        .ConfigureLoggers(logConfig => {
            logConfig.AddLoggerFactory();
        });
});
```

We’re going to add more to it here in a moment!

## Starting Actors with Akka.Hosting

Now for an Akka.NET best practice: it’s a _really_ good idea to create Akka.Hosting extension methods for configuring your Akka.NET actors and your `ActorSystem`. It’s the same idea as defining them for configuration your ASP.NET Core applications.

The short reason why this is a best practice: it makes it very easy to get your production configuration under test coverage, something that is extremely tedious to do with string-based configuration by comparison.

### Creating Custom `AkkaConfigurationBuilder` Configuration Extensions

Create a new folder called `Config` inside `AkkaWordCounter2.App`.

Next, create a new file called `ActorConfigurations.cs` inside the `AkkaWordCounter2.App/Config` folder.

Type the following code into `ActorConfigurations.cs`:

```cs
using Akka.Hosting;
using Akka.Routing;
using AkkaWordCounter2.App.Actors;

namespace AkkaWordCounter2.App.Config;

public static class ActorConfigurations {
    public static AkkaConfigurationBuilder AddWordCounterActor(this AkkaConfigurationBuilder builder) {
        return builder.WithActors((system, registry, _) => {
            var props = Props.Create(() => new WordCounterManager());
            var actor = system.ActorOf(props, "wordcounts");
            registry.Register<WordCounterManager>(actor);
        });
    }
}
```

All of our Akka.Hosting-based configuration methods are all extending the `AkkaConfigurationBuilder` type - this is Akka.NET’s equivalent to the `IServiceCollection` from `Microsoft.Extensions.DependencyInjection`.

The `WithActors` method provides us with three parameters we can use to start actors:

1. The `ActorSystem` - obviously, we can’t start top-level actors without that;
2. The `ActorRegistry` - for both retrieving and saving `IActorRef`s using typed-keys for dependency injection later; and
3. The `DependencyResolver` - an Akka.DependencyInjection abstraction wrapped around the `IServiceProvider`. We’re going to see more of it in a moment.

So inside this method we start our actor and then we make the following method call:

```cs
registry.Register<WordCounterManager>(actor);
```

This will allow the `ActorRegistry` to retrieve the `WordCounterManager`’s `IActorRef` later in the future by either calling `ActorRegistry.Get<TActor>` or `ActorRegistry.GetAsync<TActor>`[^1]. Or, if we want to resolve actors via constructor-based dependency injection, the most popular kind, we can use the `IRequiredActor<TActor>` type:

```cs
public WordCountJobActor(
    IRequiredActor<WordCounterManager> wordCounterManager,
    IRequiredActor<ParserActor> parserActor) {
    _wordCounterManager = wordCounterManager;
    _parserActor = parserActor;
}
```

## Working with Akka.DependencyInjection

Next, we need to configure the `ParserActor` - and if you recall, this actor takes a dependency through its constructor: the `IHttpClientFactory`.

Add the following code to the `ActorConfigurations` class we just defined:

```cs
public static AkkaConfigurationBuilder AddParserActors(this AkkaConfigurationBuilder builder) {
    return builder.WithActors((system, registry, resolver) => {
        // ParserActor has DI'd dependencies
        var props = resolver.Props<ParserActor>()
            // create a round-robin pool of 5
            .WithRouter(new RoundRobinPool(5));
        
        var actor = system.ActorOf(props, "parsers");
        registry.Register<ParserActor>(actor);
    });
}
```

The code we’re using to add the `ParserActor` is much the same as the `WordCounterManager` - but with two key differences:

1. `resolver.Props<ParserActor>()` - this call uses the Akka.DependencyInjection’s `DependencyResolver` to create a `Props` instance that uses the `IServiceProvider` to instantiate the actor. You can pass in a combination of dynamic arguments (i.e. an entity id unique per actor) and DI’d arguments into the `resolve.Props<TActor>()` method.
2. We are creating a `RoundRobinPool` of 5 actors - this means that instead of getting a single `ParserActor` instance back, we’re actually getting 5 of them with a parent actor that will perform round-robin load-balancing on top of them.

If an actor created via `DependencyResolver.Props<TActor>` crashes and restarts, all of its previously DI’d arguments will be re-injected back into the new instance that’s created upon restart. See “[What Happens When Akka.NET Actors Restart?](https://petabridge.com/blog/akkadotnet-actors-restart/)” for more details on actor restarts and supervision.

### Routers in Akka.NET

A quick word on [routers in Akka.NET](https://getakka.net/articles/actors/routers.html) - routers are special types of actors built for _quickly_ routing messages to other recipient actors using popular distribution strategies like round-robin, random, broadcast, or consistent hash routing.

Routers do not have a mailbox, so they don’t function the same way as regular actors - but this also means they are capable of achieving much higher throughputs, like 50-60m messages per second versus the usual 7-8m.

We have two families of routers in Akka.NET:

- **`Pool` routers** - these routers create a pool of identical routees from the same `Props` and then apply the given distribution strategy to them. These are primarily used locally (i.e. in-process routing) for increasing parallelism.
- **`Group` routers** - these routers distribute messages to externally-created routees using their `ActorPath`s. They are primarily used in Akka.Cluster and in Akka.Remote applications for facilitating inter-node messaging.

The syntax we used to transform our `ParserActor` into a `RoundRobinPool` of 5 actors is pretty straightforward:

```cs
var props = resolver.Props<ParserActor>()
            // create a round-robin pool of 5
        .WithRouter(new RoundRobinPool(5));
```

`Pool` routers make the most sense when you’re working with “stateless worker” actors, which is exactly what the `ParserActor` is.

### Dependency Lifecycles

One thing we make really, really, really clear in [the Akka.DependencyInjection literature](https://getakka.net/articles/actors/dependency-injection.html#managing-lifecycle-dependencies-with-akkadependencyinjection): we will never, under any circumstances, manage the lifecycle of injected dependencies into actors for you.

We tried that years ago and it was not a success, for one simple reason: actors can live forever!

In .NET-land, dependency injection is almost always taught in the context of ASP.NET / ASP.NET Core. HTTP requests, ASP.NET’s raison d’entre, have very short lifetimes and therefore injecting transient dependencies that are auto-disposed once the request is complete makes a lot of sense.

Applying the same pattern to Akka.NET actors, which can have uptimes measured in _years_, will create problems.

If you need to inject an inherently transient object like `SqlConnection`s into an actor, you do it the following way[^1]:

```cs
internal sealed class CohortActor : UntypedPersistentActor, IWithTimers, IWithStash {
    private readonly ILoggingAdapter _log = Context.GetLogger();
    public CohortDeliveryState State { get; private set; }
    private readonly IActorRef _textSynthesizer;
    private readonly IActorRef _changeFeed;
    private readonly IServiceProvider _serviceProvider;
    private readonly IMaterializer _materializer = ActorMaterializer.Create(Context);

    private CancellationTokenSource? _currentSendTokenSource;

    public CohortActor(CohortId cohortId,
        IRequiredActor<TextSynthesisActorRegistryKeys.TextSynthesizerActorKey> textSynthesizer, IRequiredActor<ChangeFeedActorRegistryKeys.PackageCheckManagerKey> changeFeed,
        IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
        PersistenceId = PersistentIds.CohortPersistenceId(cohortId);
        State = new CohortDeliveryState(cohortId);
        _textSynthesizer = textSynthesizer.ActorRef;
        _changeFeed = changeFeed.ActorRef;
    }

    // rest of class...
```

We inject the `IServiceProvider` as a dependency into the actor and then call `CreateScope` on it whenever we need access to a transient dependency:

```cs
private async Task<SubscriberCohort?> GetCohortFromDb(CohortId cohortId, CancellationToken ct = default) {
    using var scope = _serviceProvider.CreateScope();
    var cohortRepository = scope.ServiceProvider.GetRequiredService<ICohortRepository>();
    return await cohortRepository.GetCohortFromDb(cohortId, ct);
}
```

And if we just add a `using` statement to the `IServiceScope` that gets returned it’ll dispose all of the injected dependencies resolved from the scope automatically. ASP.NET Core does the exact same thing just before it creates the HTTP request handler lifetime.

Actors are long-lived objects - and just like how [Microsoft strongly cautions against injecting transient dependencies into singletons](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection-guidelines) since they present a “life-cycle mismatch,” we should apply that same thinking to actors and their dependencies.

If you want to learn more about DI with Akka.NET, please check out our video “[Everything You Wanted to Know about Dependency Injection and Akka.NET](https://www.youtube.com/watch?v=XNIVS5hBTUk)”

<iframe width="560" height="315" src="https://www.youtube.com/embed/XNIVS5hBTUk" title="Everything You Wanted to Know about Dependency Injection and Akka.NET" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
## Wrapping Up

Now that we’ve defined some `AkkaConfigurationBuilder` extensions for wiring up our actors, it’s time that we learn how to get our production configuration covered with unit tests - so [in the next lesson, we’re going to introduce the Akka.Hosting.TestKit](https://petabridge.com/bootcamp/lessons/unit-1/akka-hosting-testkit/), an enhancement to the Akka.TestKit that makes it very easy to integration-test our actors.

### Further Reading

- [Akka.NET, ASP.NET Core, Hosted Services, and Dependency Injection](https://petabridge.com/blog/akkadotnet-hosting-aspnet/)
- [Akka.NET v1.5: No Hocon, No Lighthouse, No Problem](https://petabridge.com/blog/akkadotnet-1.5-no-hocon-no-lighthouse-no-problem/)
- [Introducing Akka.Hosting - HOCONless Akka.NET Configuration and Runtime](https://petabridge.com/blog/intro-akka-hosting/) - **NOTE**: slightly old; Akka.Hosting’s APIs were not mature at the time we made this.

[^1]: The `ActorRegistry.GetAsync<T>` method is mostly for scenarios where someone might need access to an actor before it’s actually been created - this can happen, for instance, if you have a `BackgroundService` that launches as part of your `IHostBuilder` and it depends on an actor being available at startup. Akka.Hosting manages all of its actors in the background via an `IHostedService` so there might be a 1-2 millisecond delay before all of the actors have been fully instantiated. [↩](https://petabridge.com/bootcamp/lessons/unit-1/akka-hosting/#fnref:1)
    
[^2]: The `CohortActor` is from one of our production Akka.NET applications that is still under development. [↩](https://petabridge.com/bootcamp/lessons/unit-1/akka-hosting/#fnref:2)

---

- Previous Lesson: [[6 Actors and Async and Await]]
- Next Lesson: [[8 Integration Testing with Akka.Hosting.TestKit]]


