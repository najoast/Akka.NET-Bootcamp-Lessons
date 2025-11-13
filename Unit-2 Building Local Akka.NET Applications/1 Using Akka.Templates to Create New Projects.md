## Akka.NET Word Counter 2

In Unit-0 we created a new project using the default `console` project type via `dotnet new` - in Unit-1 we’re going to create a new version of our “word counter” application from before with some additional capabilities:

1. Ability to count words off of multiple web pages;
2. Can aggregate word counts across documents; and
3. Can provide incremental progress reports as it works.

This will allow us to continue our Akka.NET education and get exposed to some more real-world questions such as:

1. Handle `Task<T>`s inside actors;
2. Run multiple actor workloads in parallel - _and_ make sure the right message is received by the right actors;
3. Create finite state machines using switchable behaviors;
4. Leverage `IWithTimers` to schedule recurring messages and time-outs to actors; and
5. Use `ReceiveTimeout` to terminate idle actors, a popular technique for bounding memory consumption with actors;
6. How to integrate actors with the Microsoft.Extensions ecosystem for configuration, logging, hosting, dependency injection, and more.
7. Unit testing our actors with the Akka.TestKit and the Akka.Hosting.TestKit.

Let’s get started.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Amm7jmTifX8" title="Tutorial: Creating Professional, Local Akka.NET Applications (Bootcamp 2.0 - Unit 1)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
_Starts at the appropriate timestamp for this lesson_

### Getting Help with Unit-2

If you get stuck at any point during these coding exercises, please refer to the [Unit-1 source code on https://github.com/petabridge/akka-bootcamp/](https://github.com/petabridge/akka-bootcamp/tree/master/unit-1)

You can also [try the `#akkadotnet-bootcamp` channel in the Akka.NET Discord](https://discord.gg/V5KeetTDE2) and ask questions there.

## Installing `Akka.Templates`

The first thing we’re going to do is install [`Akka.Templates`, a set of open source `dotnet new` templates](https://github.com/akkadotnet/akkadotnet-templates) maintained by the Akka.NET project.

> If you’ve never installed a `dotnet new` template before, you can learn more about them here: [https://github.com/dotnet/templating/](https://github.com/dotnet/templating/).

```powershell
dotnet new install "Akka.Templates::*"
```

This will install the [`Akka.Templates` package from NuGet](https://www.nuget.org/packages/Akka.Templates) or update you to using the latest version of the package if you had an older one installed previously.

Any template you install for the `dotnet` CLI will _usually_ also work for IDEs like Visual Studio, VS Code, and JetBrains Rider - so these templates will create either new project / solution / code item options inside the menus for those IDEs as well.

### Create a New `Akka.Templates` Project

We’re going to use [the `akka.console` template](https://github.com/akkadotnet/akkadotnet-templates/blob/dev/docs/ConsoleTemplate.md) as the foundation of our new “word counter 2” project.

`akka.console` is a `project` template, so in order to make it easy for us to add things like unit tests we’re going to create a `.sln` file first.

Execute the following in your shell:

```powershell
dotnet new sln -n "AkkaWordCounter2"
dotnet new akka.console -n "AkkaWordCounter2.App"
dotnet sln add AkkaWordCounter2.App
dotnet build
```

Not the world’s most creative name, but whatever, that’s what we’re rolling with.

You should see the following structure in the directory where you ran your command:

```powershell
.                                                                    
├── AkkaWordCounter2.App                                             
│             ├── AkkaWordCounter2.App.csproj                                  
│             ├── HelloActor.cs                                                
│             ├── Program.cs                                                   
│             ├── README.md                                                    
│             ├── TimerActor.cs                                                
│             ├── Usings.cs                                 
└── AkkaWordCounter2.sln                                                                         
```

## Exploring the Template Output

Open the solution and then look at `AkkaWordCounter2.App\Program.cs`. What did the template make for us?

```cs
using Akka.Hosting;
using AkkaWordCounter2;
using Microsoft.Extensions.Hosting;

var hostBuilder = new HostBuilder();

hostBuilder.ConfigureServices((context, services) => {
    services.AddAkka("MyActorSystem", (builder, sp) => {
        builder
            .WithActors((system, registry, resolver) => {
                var helloActor = system.ActorOf(Props.Create(() => new HelloActor()), "hello-actor");
                registry.Register<HelloActor>(helloActor);
            })
            .WithActors((system, registry, resolver) => {
                var timerActorProps =
                    resolver.Props<TimerActor>(); // uses Msft.Ext.DI to inject reference to helloActor
                var timerActor = system.ActorOf(timerActorProps, "timer-actor");
                registry.Register<TimerActor>(timerActor);
            });
    });
});

var host = hostBuilder.Build();

await host.RunAsync();
```

So this is a very different looking setup than what we saw in Unit-0! Where’s the `ActorSystem.Create` and what are all of these `WithActors` calls?

This is [Akka.Hosting](https://github.com/akkadotnet/Akka.Hosting) - Akka.NET’s “Pit of Success” for making sure customers handle application lifetimes, dependency injection, Microsoft.Extensions integration, and more correctly without too much trouble. We’re going to learn Akka.Hosting by using it throughout this sample.

What this code does:

- Starts the the `HelloActor` and [registers it with the `ActorRegistry`](https://github.com/akkadotnet/Akka.Hosting?tab=readme-ov-file#registering-actors-with-the-actorregistry);
- Starts the `TimerActor` using Akka.DependencyInjection (because it has arguments injected into its constructor); and
- Uses Microsoft.Extensions.Hosting to manage the lifetime of the `ActorSystem` - this is all handled automatically through a background service that gets started as part of the `IServiceCollection.AddAkka` extension method.

We’re going to get into this componentry as we go through the lessons. For now, let’s move onto designing our actor hierarchy and messages for `AkkaWordCounter2`.

### Further Reading

- [Akka.NET, ASP.NET Core, Hosted Services, and Dependency Injection](https://petabridge.com/blog/akkadotnet-hosting-aspnet/)
- [Everything You Wanted to Know about Dependency Injection and Akka.NET](https://www.youtube.com/watch?v=XNIVS5hBTUk)
- [Building Headless Akka.NET Services](https://www.youtube.com/watch?v=EMFALLo0OJ0)

---
- Previous Lesson: [[3 Effective Actor Messaging]]
- Next Lesson: [[2 Actor Hierarchies and Domain Modeling]]