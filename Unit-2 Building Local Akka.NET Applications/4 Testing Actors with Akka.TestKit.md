Akka.NET supports all of the popular unit test frameworks: NUnit, xUnit, and MsTest via its array of [Akka.TestKit packages](https://www.nuget.org/packages/Akka.TestKit). In this lesson we’re going to learn the 20% of the TestKit you need for 80% of use cases in order to validate that the `DocumentWordCounter` actor we wrote in the previous lesson is working correctly.

## Testing Actors

“All of our weaknesses are our strengths taken to an extreme” - unknown.

One inherent issue with testing actors is that, by design, actors are designed to encapsulate their internal state and behavior, preventing direct external access. Therefore, using vanilla C# and F# testing tools is going to be problematic.

Enter the Akka.NET TestKit - a set of testing utilites that are testing-framework-agnostic and are designed to do the following:

1. Create a fully isolated `ActorSystem` for every test instance, in order to prevent cross-pollination and contamination;
2. Provide tools that make it easy to test external actor behavior (i.e. replying to messages); and
3. Provide tools that make it possible to access internal actor state, such as the `ActorOfAsTestActorRef<TActor>()` method.

In this unit we’re not going to explore every possible piece of functionality inside the TestKit as those are all actually pretty easy to discover via IDE auto-completion, but we will demonstrate how to use the TestKit to design simple, effective, and easy-to-understand tests.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Amm7jmTifX8" title="Tutorial: Creating Professional, Local Akka.NET Applications (Bootcamp 2.0 - Unit 1)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
_Starts at the appropriate timestamp for this lesson_

### Creating the `AkkaWordCounter2.App.Tests` Project

We need to add a new project to our existing `AkkaWordCounter2.sln` file.

> If you get stuck at any point during these coding exercises, please refer to the [Unit-1 source code on https://github.com/petabridge/akka-bootcamp/](https://github.com/petabridge/akka-bootcamp/tree/master/unit-1)

Add a new “xUnit Test Project” to the solution and call it `AkkaWordCounter2.App.Tests` - your `.csproj` should have all of the same content as the reference code here: [`AkkaWordCounter2.App.Tests.csproj`](https://github.com/petabridge/akka-bootcamp/tree/master/unit-1/completed/AkkaWordCounter2.App.Tests/AkkaWordCounter2.App.Tests.csproj)

You can just copy and paste that XML into your `.csproj` and run `dotnet build` to restore all of the correct NuGet packages.

Ok, does your `.csproj` look like the reference file[^1]? If so, we are ready to move on.

### Writing Your First `TestKit` Test

Inside the `AkkaWordCounter2.App.Tests` add a new file called `DocumentWordCounterSpecs.cs` and type in the following:

```cs
using Akka.Actor;
using Akka.TestKit.Xunit2;
using AkkaWordCounter2.App.Actors;
using Xunit.Abstractions;

namespace AkkaWordCounter2.App.Tests;

public class DocumentWordCounterSpecs : TestKit
{
    public static readonly Akka.Configuration.Config Config = "akka.loglevel=DEBUG";
    
    public DocumentWordCounterSpecs(ITestOutputHelper output) : base(output: output, config: Config)
    {
        
    }
    
    public static readonly AbsoluteUri TestDocumentUri = new(new Uri("http://example.com/test"));

    [Fact]
    public async Task ShouldProcessWordCountsCorrectly()
    {
        // arrange
        var props = Props.Create(() => new DocumentWordCounter(TestDocumentUri));
        var actor = Sys.ActorOf(props);

        IReadOnlyList<IWithDocumentId> messages = [
            new DocumentEvents.WordsFound(TestDocumentUri, ["hello", "world"]),
            new DocumentEvents.WordsFound(TestDocumentUri, ["bar", "foo"]),
            new DocumentEvents.WordsFound(TestDocumentUri, ["HeLlo", "wOrld"]),
            new DocumentEvents.EndOfDocumentReached(TestDocumentUri)
        ];
        
        // have the TestActor subscribe to updates
        actor.Tell(new DocumentQueries.FetchCounts(TestDocumentUri), TestActor);
        
        // act
        foreach (var message in messages)
        {
            actor.Tell(message);
        }
        
        // assert
        var response = await ExpectMsgAsync<DocumentEvents.CountsTabulatedForDocument>();
        Assert.Equal(6, response.WordFrequencies.Count); // words are case sensitive
    }
}
```

The `ShouldProcessWordCountsCorrectly` is going to test the `DocumentWordCounter` actor by feeding it a sequence of `IWithDocumentId` messages which closely match the real-world input this actor is going to receive. At the end of that sequence, the `TestActor` - a special actor that is built-in to the `TestKit` base class, should receive the completed output: a `CountsTabulatedForDocument` output with 6 distinct terms appearing in its contents.

Let’s break down how this test works.

## Using the `TestKit`

There are three different families of TestKit packages to choose from:

1. [Akka.TestKit.xUnit](https://www.nuget.org/packages?q=Akka.TestKit.xUnit) - this is what we use internally and it gets the best support.
2. [Akka.TestKit.NUnit](https://github.com/akkadotnet/Akka.TestKit.NUnit) - NUnit testing support.
3. [Akka.TestKit.MSTest](https://github.com/akkadotnet/Akka.TestKit.MSTest) - rarely used these days.

If you followed the directions earlier in this article, you’ve already installed the `Akka.TestKit.Xunit2` package into `AkkaWordCounter2.App.Tests` otherwise the test we just wrote wouldn’t compile.

The first thing we have to do when using the `TestKit` is make sure all of our test fixtures inherit from the `TestKit` base class! This is what ensures that we get a unique `ActorSystem`, a `TestActor`, and all of the other built-in testing infrastructure we need to test our actors.

### Configuring the `ActorSystem`

Now by default the `TestKit` does not support Akka.Hosting, so you have to configure it using [HOCON](https://getakka.net/articles/configuration/hocon.html) - Akka.NET’s string-based configuration format.

> Later in this Unit of Akka.NET Bootcamp we’re going to learn how to use the Akka.Hosting.TestKit which avoids having to use HOCON at all. Generally speaking, we’re trying to abstract away HOCON as we continue to develop Akka.NET.

Here’s how we do that in `DocumentWordCounterSpecs`:

```cs
public class DocumentWordCounterSpecs : TestKit
{
    public static readonly Akka.Configuration.Config Config = "akka.loglevel=DEBUG";
    
    public DocumentWordCounterSpecs(ITestOutputHelper output) : base(output: output, config: Config)
    {
        
    }
}
```

We pass the `Akka.Configuration.Config` object containing our HOCON into the base class constructor[^2] - in this case all we’re doing is lowering the default `LogLevel` from `Info` to `Debug` so we can see more logs.

> [!TIP] And one additional tip for xUnit users: always make sure your tests accept an `ITestOutputHelper` in their constructors and then pass that down to the base class’ constructor too. This will ensure that all of Akka.NET’s logs are captured by xUnit’s output engine correctly.

### Using the `TestActor`

The `TestActor` is a “fully assertable” actor that is built into the `TestKit` - we can assert what types of messages it receives, whether it not it received anything, and so on.

The `TestActor` is also _always the implicit `Sender`_ when sending messages from inside a unit test to any other actor type - but in order to make the test really obvious for training purposes, we’ve made the `TestActor` the explicit sender on the `FetchCounts` message to the `DocumentWordCounter`.

```cs
actor.Tell(new DocumentQueries.FetchCounts(TestDocumentUri), TestActor);
```

If our `DocumentWordCounter` actor is implemented correctly, we should see the following assertion pass.

```cs
var response = await ExpectMsgAsync<DocumentEvents.CountsTabulatedForDocument>();
```

All `ExpectMsgAsync<T>` calls are designed to fail if they:

1. Don’t receive any message within 3 seconds by default or
2. Receive a message OTHER than the constraints specified by the `T` and the other optional parameters on the method.

You can lengthen the timeout on an individual `ExpectMsgAsync<T>` call by passing in a longer `TimeSpan` on one of the optional parameters, or you can wrap the `ExpectMsgAsync<T>` call inside a `WithinAsync` block:

```cs
await WithinAsync(async () =>{
    // both calls must complete within the same 10 second window, not two separate ones
    await ExpectMsgAsync<DocumentEvents.CountsTabulatedForDocument>();
    await ExpectMsgAsync<DocumentEvents.CountsTabulatedForDocument>();
}, TimeSpan.FromSeconds(10))
```

### Running the Tests

Run `dotnet test` and see what happens:

```powershell
[DEBUG][02/06/2025 22:44:03.644Z][Thread 0015][akka://test/user/$a] Found 2 words in document http://example.com/test
[DEBUG][02/06/2025 22:44:03.644Z][Thread 0015][akka://test/user/$a] Found 2 words in document http://example.com/test
[DEBUG][02/06/2025 22:44:03.644Z][Thread 0015][akka://test/user/$a] Found 2 words in document http://example.com/test
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0015][CoordinatedShutdown (akka://test)] Performing phase [before-service-unbind] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [service-unbind] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [service-requests-done] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [service-stop] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [before-cluster-shutdown] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [cluster-sharding-shutdown-region] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [cluster-leave] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [cluster-exiting] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [cluster-exiting-done] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [cluster-shutdown] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.662Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [before-actor-system-terminate] with [0] tasks.
[DEBUG][02/06/2025 22:44:03.666Z][Thread 0017][CoordinatedShutdown (akka://test)] Performing phase [actor-system-terminate] with [1] tasks: [terminate-system]
[DEBUG][02/06/2025 22:44:03.666Z][Thread 0017][ActorSystem(test)] System shutdown initiated
[DEBUG][02/06/2025 22:44:03.671Z][Thread 0011][EventStream] Shutting down: StandardOutLogger started
[DEBUG][02/06/2025 22:44:03.671Z][Thread 0011][EventStream] All default loggers stopped
```

The test passes and we’re looking good!

## Wrapping Up

That’s it for our quick introduction to the Akka.NET TestKit - in the next lesson we’re going to [revisit the actor hierarchy](https://petabridge.com/bootcamp/lessons/unit-1/actor-hierarchies/) and [learn how to spawn some child actors and how to use the `ReceiveActor` base type built into Akka.NET](https://petabridge.com/bootcamp/lessons/unit-1/child-actors/).

### Further Reading

- [How to Test Akka.NET Applications Pt 1: How to Make Actors Easily Testable](https://www.youtube.com/watch?v=55pms0f9wBY)
- [How to Test Akka.NET Applications Pt 2: Writing Akka.NET Unit & Integration Tests with Akka.TestKit](https://www.youtube.com/watch?v=1Ek07lDCa5Y)
- [How to Unit Test Akka.NET Actors with Akka.TestKit](https://petabridge.com/blog/how-to-unit-test-akkadotnet-actors-akka-testkit/)

[^1]: We’re not embedding the XML from the reference project’s `AkkaWordCounter2.App.Tests.csproj` directly into the written tutorial so we can avoid byte-rot. Dependabot can automatically update the reference tutorial - it can’t update this web page. [↩](https://petabridge.com/bootcamp/lessons/unit-1/unit-testing-akkadotnet/#fnref:1)
    
[^2]: You can look up all of the default HOCON options for all of Akka.NET here: [https://getakka.net/articles/configuration/modules/akka.html](https://getakka.net/articles/configuration/modules/akka.html) [↩](https://petabridge.com/bootcamp/lessons/unit-1/unit-testing-akkadotnet/#fnref:2)

---

- Previous Lesson: [[3 Behavior Switching and Receive Timeouts]]
- Next Lesson: [[5 Working with Child Actors]]