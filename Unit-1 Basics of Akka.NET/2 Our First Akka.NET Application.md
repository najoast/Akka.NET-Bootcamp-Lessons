## A Basic Akka.NET Console Application

Let’s get into the basics of [Akka.NET](https://getakka.net/) with a simple console application.

<iframe width="560" height="315" src="https://www.youtube.com/embed/NnxC6ySwrRs" title="Tutorial: Creating Your First Akka.NET Application (Bootcamp 2.0 Unit-0 - Lesson 1 of 2)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

The first thing we’re going to implement is a super basic “Hello World” with actors so we can get a feel for actor syntax and some of the basic tools.

> We’re going to gradually upgrade this sample so it does something less trivial so feel free to skip ahead at any time.

Also, don’t forget - you can find the completed source code for this unit here: [https://github.com/petabridge/akka-bootcamp/tree/master/unit-0](https://github.com/petabridge/akka-bootcamp/tree/master/unit-0)

### Getting Help with Unit-1

If you get stuck at any point during these coding exercises, please refer to the [Unit-0 source code on https://github.com/petabridge/akka-bootcamp/](https://github.com/petabridge/akka-bootcamp/tree/master/unit-0)

You can also [try the `#akkadotnet-bootcamp` channel in the Akka.NET Discord](https://discord.gg/V5KeetTDE2) and ask questions there.

## Creating Your First Akka.NET Application

We’re going to _eventually_ use the [Akka.Templates package](https://github.com/akkadotnet/akkadotnet-templates) to power our Akka.NET Bootcamp samples, but we’re going to start off without it here.

So for now, let’s create a new console application using the `dotnet` CLI:

```sh
dotnet new console -n AkkaWordCounter
```

Then let’s enter the directory of the new project we just made:

```sh
cd AkkaWordCounter
```

And then install the latest version of the [Akka NuGet package](https://www.nuget.org/packages/Akka).

```sh
dotnet add package Akka
```

That’s all the work we need to do on the CLI. Now open up the `Program.cs` inside your favorite text editor or IDE.

### Creating the ActorSystem

The first thing we need to do is to create an [`ActorSystem`](https://getakka.net/api/Akka.Actor.ActorSystem.html); these are the constructs used to create actors, configure them, resolve their `ActorPath`s, and more.

So first things first - this is a minimal .NET application and typically your `Program.cs` will look like this:

```cs
// See https://aka.ms/new-console-template for more information
Console.WriteLine("Hello, World!");
```

The first thing we’re going to do is _delete this code_ and replace it with the following:

```cs
using Akka.Actor;
using Akka.Event;

ActorSystem myActorSystem = ActorSystem.Create("LocalSystem");
myActorSystem.Log.Info("Hello from the ActorSystem");
await myActorSystem.Terminate();
```

This is going to create an `ActorSystem`, it’ll use the [built-in Akka.NET `ILoggingAdapter`](https://getakka.net/articles/utilities/logging.html) to write out to the console, and then it will wait for the `ActorSystem` to terminate before exiting.

Let’s run the sample and make sure we get some output.

```powershell
PS> dotnet run
[INFO][{DateTime}][Thread 0001][ActorSystem(LocalSystem)] Hello from the ActorSystem
```

Ok, we’re in business.

### Creating Your First Actor

For this next step we’re going to create a basic actor - there’s several different actor base classes in Akka.NET but there’s really two fundamental ones we can use:

- [**`UntypedActor`**](https://getakka.net/articles/actors/untyped-actor-api.html) - the simplest possible actor type and
- [**`ReceiveActor`**](https://getakka.net/articles/actors/receive-actor-api.html)[1](https://petabridge.com/bootcamp/lessons/unit-0/first-akkadotnet-app/#fn:1) - an actor that uses strongly-typed `Receive<T>` and `ReceiveAsync<T>` delegates to define message-handlers for our actors.

For this simple we’re going to roll with an `UntypedActor` aimed at doing some basic ‘Hello World’ type work.

In a new file called `HelloActor.cs` please type or copy-and-paste the following:

```cs
using Akka.Actor;
using Akka.Event;

namespace AkkaWordCounter;

// basic actor that just logs whatever it receives and replies back
public class HelloActor : UntypedActor
{
    private readonly ILoggingAdapter _log = Context.GetLogger();
    
    protected override void OnReceive(object message)
    {
        switch (message)
        {
            case string msg:
                _log.Info("Received message: {0}", msg);
                Sender.Tell($"{msg} reply");
                break;
            default:
                Unhandled(message);
                break;
        }
    }
}
```

This is a very simple actor implementation:

- It looks for a `string` message;
- Logs it using Akka.NET’s built-in logging infrastructure[2](https://petabridge.com/bootcamp/lessons/unit-0/first-akkadotnet-app/#fn:2); and
- Replies back to the `Sender` - the `IActorRef` who sent us this message.

But we’re not done yet: we still need to start this actor and make sure the `ActorSystem` gives it an address!

### Spawning and Messaging Our Actor

Let’s go back to `Program.cs` and add the following lines **before** we call `myActorSystem.Terminate()`:

```cs
// Props == formula for creating an actor
Props myProps = Props.Create<HelloActor>();

// IActorRef == handle for messaging an actor
// Survives actor restarts, is serializable
IActorRef myActor = myActorSystem.ActorOf(myProps, "MyActor");

// tell my actor to display a message via fire-and-forget messaging
myActor.Tell("Hello, World!");

// use Ask<T> to do request-response messaging
string whatsUp = await myActor.Ask<string>("What's up?");
Console.WriteLine(whatsUp);
```

This code is going to do the following:

1. Define `Props` for the `HelloActor` - `Props` is the formula we can use to configure an actor. It’s immutable, serializable, and can be used multiple times to start many actors with the same configuration.
2. Create an instance of our `HelloActor` via the `ActorOf` method, which returns an `IActorRef` we can use to message our actor.
3. Messages the actor in two different ways: one via `IActorRef.Tell`, which leverages Akka.NET’s default “fire-and-forget” messaging style - and another that uses `Ask<T>` to message the actor using request-response.

If you run the application you should see the following output:

```powershell
What's up? reply
[INFO][{DateTime}][Thread 0001][ActorSystem(LocalSystem)] Hello from the ActorSystem
[INFO][{DateTime}][Thread 0011][akka://LocalSystem/user/MyActor] Received message: Hello, World!
[INFO][{DateTime}][Thread 0011][akka://LocalSystem/user/MyActor] Received message: What's up?
[INFO][{DateTime}][Thread 0010][akka://LocalSystem/deadLetters] Message [String] from [akka://LocalSystem/user/MyActor#259497744] to [akka://LocalSystem/deadLetters] was not delivered. [1] dead letters encountered. If this is not an expected behavior then [akka://LocalSystem/deadLetters] may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'. Message content: Hello, World! reply
```

There’s some interesting details at work here:

1. The result of the `Ask<string>` operation will often, _but not always_, get logged first. This is because Akka.NET’s own logging infrastructure runs in the background using a dedicated set of built-in actors and the .NET `ThreadPool` has to schedule those actors to run as they start receiving messages. More on this in a moment.
2. All of the calls made to `ActorSystem.Log` or the `ILoggingAdapter` inside the `HelloActor` have their output rendered in their original invocation order - this is a consequence of the Akka.NET logging system running on top of a dedicated actor.
3. The final message indicates a [`DeadLetter` log](https://getakka.net/api/Akka.Event.DeadLetter.html) - what does that mean?

## How Actor Messaging Works

> We’re going to dig into some theory here - if you want to skip it and move onto the next section, feel free. But knowing how these details work will make you a better programmer and will help you use Akka.NET more effectively.

So why did `What's up? reply` get written out via the `Console.WriteLine` first, even though the logging statements written to Akka.NET’s `ILoggingAdapter` happened earlier?

The answer is two parts:

- **Non-determinism**: no one can predict the order in which the OS is going to schedule threads for execution and
- **Asynchrony**: all actor messaging, including `IActorRef.Tell`, is always asynchronous even though it returns `void`. Many internal calls inside Akka.NET such as `ILoggingAdapter.Log` use `IActorRef.Tell` internally.

> That’s right: `IActorRef.Tell` is asynchronous even though it’s a `void` method. We have an entire blog post explaining [why this is a better default than returning `Task`s](https://petabridge.com/blog/actorref-tell-ask/).

When we call `ILoggingAdapter.Log` (or any of the extension methods that call `Log` internally, such as `.Info`) - we are actually performing an `IActorRef.Tell` call under the covers to the internal actors who power Akka.NET’s logging system.

### Execution Order: Only Guaranteed Per-Actor

Conceptually we have at least two different concurrent threads of execution at work here:

1. The console application’s foreground thread, which gets `await`-ed on the `Ask` call and
2. The background thread(s) that service the actors’ message processing.

The `HelloActor` we’ve created is going to process messages in the following order:

1. `myActor.Tell("Hello, World!")`
2. `myActor.Ask<string>("What's up?")`

You can see this clearly in the log output - this is Akka.NET doing it’s job: ensuring that all messages are processed in the original order in which they were received.

You can also see that the internal logging actors process _their messages in-order too_:

1. `[ActorSystem(LocalSystem)] Hello from the ActorSystem`
2. `[akka://LocalSystem/user/MyActor] Received message: Hello, World!`
3. `[akka://LocalSystem/user/MyActor] Received message: What's up?`

These three logging statements are all recorded in the exact sequence the `ILoggingAdapter` calls were made both by the `ActorSystem` and by the `HelloWorld` actor.

What **is not guaranteed is the _global_ execution order** - Akka.NET isn’t going to enforce whether or not the logging actor gets called first or whether the `await`’s continuation on the main foreground thread gets called first _because that’s not even controllable_.

This is why the results of the `await` might be displayed first prior to the logging statements being written out to the console.

A combination of the OS’ thread scheduler, the .NET `ThreadPool`, and the default `TaskScheduler` are responsible for execution order - and even _they_ won’t guarantee global execution order either because doing so is extremely non-performant, complicated, and fragile.

What Akka.NET guarantees is that, for a single actor, all messages are processed one-at-a-time in their originally received order.

## Dead Letters and Unprocessable Messages

A final question you may have about the output so far: what’s the deal with this “dead letter?”

```powershell
[INFO][{DateTime}][Thread 0010][akka://LocalSystem/deadLetters] Message [String] from [akka://LocalSystem/user/MyActor#259497744] to [akka://LocalSystem/deadLetters] was not delivered. [1] dead letters encountered. If this is not an expected behavior then [akka://LocalSystem/deadLetters] may have terminated unexpectedly. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'. Message content: Hello, World! reply
```

Dead letters are a class of undeliverable message in Akka.NET - in this case it means that an `IActorRef` or `ActorSelection` we called `.Tell` on either never existed or was terminated prior to us messaging it.

In this particular instance it’s this line inside the `HelloActor` that is causing the dead letter:

```cs
case string msg:
    _log.Info($"Received message: {msg}");
    Sender.Tell($"{msg} reply"); // DeadLetter triggered here when the `Sender` is not set
    break;
```

The reason being is that when we message the `HelloActor` from outside the `ActorSystem` in our `Program.cs`:

```cs
IActorRef myActor = myActorSystem.ActorOf(myProps, "MyActor");

// tell my actor to display a message via Fire-and-Forget messaging
myActor.Tell("Hello, World!"); 
```

**There is no sender to reply back to** - therefore Akka.NET will automatically substitute that `null` / `ActorRefs.NoSender` value with a reference to our `DeadLetterListener` - the actor that wrote the log message we can see on the console.

You’ll notice we _do not_ get a dead letter during the `Ask<T>` call to the `HelloActor` - this is because the `Ask` operation creates a tempory actor and passes _that_ along as the `Sender` in this case.

### Do I Need to Worry About Dead Letters?

No, not typically. Dead letters _usually_ occur when actors are shutting down inside Akka.NET. And in this case, this dead letter is no big deal because we’re not expecting a reply back.

_However_, it’s still important to keep an eye on dead letters because they might indicate a programming error. There are two classes of dead letters:

1. **[`DeadLetter`](https://getakka.net/api/Akka.Event.DeadLetter.html)** - this means that the message was sent to an `IActorRef` that doesn’t exist. This means either the actor shut down or we have an addressing problem.
2. **[`UnhandledMessage`](https://getakka.net/api/Akka.Event.UnhandledMessage.html)** - when this happens, it means the `Unhandled` message handler inside a live, running actor was called. This is _almost always_ evidence of a programming bug. It means a living actor received the message but was not programmed to handle it. Always investigate these! They’ll appear with the log message: `Message [{messageStr}] was unhandled.`

## Wrapping Up

So we’ve made our first actor program and learned a bit about Akka.NET’s message-processing behavior. In the next lesson we’re going to learn about actors messaging each other to do more interesting things.

### Further Reading

More articles, blog posts, and videos relevant to this lesson:

1. [Why `IActorRef.Tell` Doesn’t Return a `Task`](https://petabridge.com/blog/actorref-tell-ask/)
2. [How Akka.NET Actors Process Messages](https://petabridge.com/blog/how-akkadotnet-actors-process-messages/)

3. In the early days of Akka.NET, `switch`-based pattern matching wasn’t available in the C# language specification yet - thus we created the `ReceiveActor` to help make it easier to build pattern matching behavior in an idiomatic C# fashion. These days, it’s much less necessary and the `UntypedActor` is just as easy to use in most cases. [↩](https://petabridge.com/bootcamp/lessons/unit-0/first-akkadotnet-app/#fnref:1)
    
4. Why not use [.NET string interpolation](https://learn.microsoft.com/en-us/dotnet/csharp/tutorials/string-interpolation) here? _Performance_. Generally with all logging calls, whether it’s Akka.NET’s `ILogger` or Microsoft.Extensions.Logging you want to defer string formatting until the log hits the log export pipeline. This is an extremely useful performance optimization technique known as “[Deferred Allocation](https://www.youtube.com/watch?v=Hk_jvttYb2c).” Roslyn Analyzers will often flag string interpolation usage inside logging statements for this reason. [↩](https://petabridge.com/bootcamp/lessons/unit-0/first-akkadotnet-app/#fnref:2)

- Previous Lesson: [[1 Why Learn Akka.NET]]
- Next Lesson: 