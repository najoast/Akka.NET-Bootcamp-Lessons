We [introduced the Akka.TestKit](https://petabridge.com/bootcamp/lessons/unit-1/unit-testing-akkadotnet/) earlier in Unit 1 of Akka.NET Bootcamp - and one of the problems we mentioned with it is that it still relies on HOCON, Akka.NET’s legacy / internal configuration system.

In this lesson we are going to introduce the Akka.Hosting.TestKit, a fusion of `Akka.Hosting` and the Akka.TestKit that is aimed at making it easier to write integration tests that touch Akka.NET actors and, optionally, the dependencies injected into them.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Amm7jmTifX8" title="Tutorial: Creating Professional, Local Akka.NET Applications (Bootcamp 2.0 - Unit 1)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
_Starts at the appropriate timestamp for this lesson_

## Akka.Hosting.TestKit

Let’s [install the Akka.Hosting.TestKit](https://www.nuget.org/packages/Akka.Hosting.TestKit) into our `AkkaWordCounter2.App.Tests` project, if it’s not installed already.

```sh
dotnet add package Akka.Hosting.TestKit
```

> A small bummer for NUnit and MSTest users: the Akka.Hosting.TestKit is currently xUnit-only.

The Akka.Hosting.TestKit has the following goals:

1. Create an `IHost` for each test method;
2. Create a real `IServiceCollection` and `IServiceProvider` for each test method - NO FAKES OR MOCKS, we’re going to use the real `Microsoft.Extensions.DependencyInjection` system to do the real thing on our code-under-test;
3. Bring all of the same testing functionality of the Akka.TestKit into an environment where we’re using `AkkaConfigurationBuilder` to configure our actors rather than arranging it as part of the test methods themselves.

The Akka.Hosting.TestKit is so convenient for testing Microsoft.Extensions-based code that we often use it for testing non-Akka.NET cases too.

## Writing Our First Integration Test

Inside `AkkaWordCounter2.App.Tests` please create a new file called `ParserActorSpecs.cs` - we will test the `ParserActor` to verify its functionality.

Inside `ParserActorSpecs.cs` please type the following:

```cs
using Akka.Hosting;
using AkkaWordCounter2.App.Actors;
using AkkaWordCounter2.App.Config;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Xunit.Abstractions;

namespace AkkaWordCounter2.App.Tests;

public class ParserActorSpecs : Akka.Hosting.TestKit.TestKit
{
    public ParserActorSpecs(ITestOutputHelper output) : base(output: output)
    {
    }

    protected override void ConfigureServices(HostBuilderContext context, IServiceCollection services)
    {
        services.AddHttpClient();
    }

    protected override void ConfigureAkka(AkkaConfigurationBuilder builder, IServiceProvider provider)
    {
        builder
            .ConfigureLoggers(configBuilder =>
            {
                configBuilder.LogLevel = Akka.Event.LogLevel.DebugLevel;
            })
            .AddParserActors();
    }
    
    public static readonly AbsoluteUri ParserActorUri = new(new Uri("https://getakka.net/"));
    
    [Fact]
    public async Task ShouldParseWords()
    {
        // arrange
        var parserActor = await ActorRegistry.GetAsync<ParserActor>();
        var expectResultsProbe = CreateTestProbe();
        
        // act
        parserActor.Tell(new DocumentCommands.ScanDocument(ParserActorUri), expectResultsProbe);
        
        // assert
        await expectResultsProbe.ExpectMsgAsync<DocumentEvents.WordsFound>(); // should get at least 1 WordsFound
        await expectResultsProbe.FishForMessageAsync(m => m is DocumentEvents.EndOfDocumentReached); // should get EndOfDocumentReached
    }
}
```

Now go ahead and execute this unit test in your IDE or using the `dotnet test` command.

```powershell
{DateTime}:INF:Microsoft.Hosting.Lifetime:0 Application started. Press Ctrl+C to shut down.
{DateTime}:INF:Microsoft.Hosting.Lifetime:0 Hosting environment: Production
{DateTime}:INF:Microsoft.Hosting.Lifetime:0 Content root path: C:\Repositories\Petabridge\akka-bootcamp\unit-1\completed\AkkaWordCounter2.App.Tests\bin\Debug\net9.0\
{DateTime}:INF:System.Net.Http.HttpClient.Default.LogicalHandler:RequestPipelineStart Start processing HTTP request GET https://getakka.net/
{DateTime}:INF:System.Net.Http.HttpClient.Default.ClientHandler:RequestStart Sending HTTP request GET https://getakka.net/
{DateTime}:INF:System.Net.Http.HttpClient.Default.ClientHandler:RequestEnd Received HTTP response headers after 467.1066ms - 200
{DateTime}:INF:System.Net.Http.HttpClient.Default.LogicalHandler:RequestPipelineEnd End processing HTTP request after 487.7466ms - 200
{DateTime}:INF:Microsoft.Hosting.Lifetime:0 Application is shutting down...
```

Our test is passing, excellent.

### Configuring Akka.Hosting.TestKit Tests

Let’s break down what we just did with `ParserActorSpecs`:

1. Just like with the regular Akka.TestKit, our test fixture classes must inherit from Akka.Hosting.TestKit;
2. We passed in xUnit’s `ITestOutputHelper` just like we did before, to ensure that our test output all gets captured appropriately by xUnit;
3. We overrode the `ConfigureServices` method body - this is optional, but this is where we register non-Akka.NET services we want to leverage in our test; and
4. Finally, we have to provide a method body for the `abstract` `ConfigureAkka` method - this is where we used the `AkkaConfigurationBuilder` extension methods we defined in “[Akka.Hosting, Routers, and Dependency Injection](https://petabridge.com/bootcamp/lessons/unit-1/akka-hosting/).”

Inside the `ConfigureAkka` method we call our `AddParsers` configuration method we defined earlier:

```cs
    protected override void ConfigureAkka(AkkaConfigurationBuilder builder, IServiceProvider provider)
    {
        builder
            .ConfigureLoggers(configBuilder =>
            {
                configBuilder.LogLevel = Akka.Event.LogLevel.DebugLevel;
            })
            .AddParserActors();
    }
```

This is how we get our production configuration under test coverage.

### Accessing Our Actors

Now the one other big difference between the regular TestKit and Akka.Hosting.TestKit is how we start our actors - in the former we just start the actors up at the start of each test typically.

But in the Akka.Hosting.TestKit the actors are launched in the background via the implicit `IHost` that is created for us, so the creation of our actors under test might happen before our test even begins to execute[^1].

Therefore, we usually end up accessing our actors via the `ActorRegistry` - just like we might do inside our applications:

```cs
// arrange
var parserActor = await ActorRegistry.GetAsync<ParserActor>();
var expectResultsProbe = CreateTestProbe();
```

### `TestProbe` and `FishForMessageAsync`

Now there are two more things this test is doing that we haven’t seen before:

1. **Creating a new `TestProbe`** - this is the equivalent to creating a second `TestActor`. `TestProbe`s have all of the same functionality as the `TestActor`, but it’s a manual instance of it that you can control.
2. **`FishForMessageAsync`** - we’re using `ExpectAsync` in this test too, but you’ll notice a second call to `FishForMessageAsync<T>`. What this method does is it _filters through_ all of the receive messages the `TestProbe` receives until it reaches the target message type `T`. Or, if 3 seconds passes before `T` is found this method will throw an assertion exception instead.

## Wrapping Up

We are almost done with Unit 1 of Akka.NET Bootcamp. The next thing we need to do is [configure `AkkaWordCounter2` using Microsoft.Extensions.Configuration and the `IOptions` pattern](https://petabridge.com/bootcamp/lessons/unit-1/msft-configuration/).

### Further Reading

- [How to Test Akka.NET Applications Pt 1: How to Make Actors Easily Testable](https://www.youtube.com/watch?v=55pms0f9wBY)
- [How to Test Akka.NET Applications Pt 2: Writing Akka.NET Unit & Integration Tests with Akka.TestKit](https://www.youtube.com/watch?v=1Ek07lDCa5Y)

[^1]: You can still, if you want to, create actors using `Sys.ActorOf` inside an Akka.Hosting.TestKit test - nothing is stopping you from doing that. But since the idea is to test our real configurations our application uses, _usually_ you’re going to end up using the `ActorRegistry` to access actors that were declared as part of the `ConfigureAkka` method [↩](https://petabridge.com/bootcamp/lessons/unit-1/akka-hosting-testkit/#fnref:1)

---

- Previous Lesson: [[7 Akka.Hosting, Routers, and Dependency Injection]]
- Next Lesson: [[9 Using IOptions and Microsoft.Extensions.Configuration]]