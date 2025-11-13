In “[Actor Hierarchies and Domain Modeling](https://petabridge.com/bootcamp/lessons/unit-1/actor-hierarchies/)” we learned what an actor hierarchy is and why it exists. In this lesson, we will build one. Additionally, we will explore the other common actor base type we haven’t been using: the `ReceiveActor`.

## `ReceiveActor`s and `UntypedActor`s

We covered this in “[Our First Akka.NET Application](https://petabridge.com/bootcamp/lessons/unit-0/first-akkadotnet-app/)”, but there are two different common actor base types in the core Akka library:

- [**`UntypedActor`**](https://getakka.net/articles/actors/untyped-actor-api.html) - the simplest possible actor type and
- [**`ReceiveActor`**](https://getakka.net/articles/actors/receive-actor-api.html)[1](https://petabridge.com/bootcamp/lessons/unit-1/child-actors/#fn:1) - an actor that uses strongly-typed `Receive<T>` and `ReceiveAsync<T>` delegates to define message-handlers for our actors.

Both actor types are equally capable[1](https://petabridge.com/bootcamp/lessons/unit-1/child-actors/#fn:1), so the choice between them is largely a matter of preference and style.

In this particular lesson we’re going to create the `WordCounterManager` actor using the `ReceiveActor` syntax just so you can get exposed to both flavors.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Amm7jmTifX8" title="Tutorial: Creating Professional, Local Akka.NET Applications (Bootcamp 2.0 - Unit 1)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
_Starts at the appropriate timestamp for this lesson_

### Creating the `WordCounterManager`

Inside the `AkkaWordCounter2.App/Actors/` folder, create a new file: `WordCounterManager.cs`.

Then, enter the following code:

```cs
using System.Web;

namespace AkkaWordCounter2.App.Actors;

// Parent actor that manages the lifecycle of DocumentWordCounter actors
public sealed class WordCounterManager : ReceiveActor
{
    public WordCounterManager()
    {
        Receive<IWithDocumentId>(s =>
        {
            string childName = $"word-counter-{HttpUtility.UrlEncode(s.DocumentId.ToString())}";
            IActorRef child = Context.Child(childName);
            if (child.IsNobody())
            {
                // start the child if it doesn't exist
                child = Context.ActorOf(Props.Create(() => new DocumentWordCounter(s.DocumentId)), childName);
            }
            child.Forward(s);
        });
    }
}
```

`ReceiveActor`s use strongly typed `Receive<T>` and `ReceiveAsync<T>` methods in order to process messages, rather than relying on `switch` statements and expressions like the `UntypedActor`.

In this particular case we’re making good on the “generic message routing” promise we made in “[Behavior-Switching and Receive Timeouts](https://petabridge.com/bootcamp/lessons/unit-1/behavior-switching/)” - the `WordCounterManager` doesn’t need to receive every imaginable type of `IWithDocumentId` message and handle those separately; it can just pattern match on the `IWithDocumentId` interface directly.

### Working with Child Actors

We’re using [the “child-per-entity” pattern](https://petabridge.com/blog/top-akkadotnet-design-patterns/) here to allocate a unique `DocumentWordCounter` per `AbsoluteUri` - this is an extremely common and good pattern to help make sure all data for a unique entity accumulates inside a single actor who represents it.

The key to making this work is naming all child actors after their respective entities - [Akka.Cluster.Sharding does this automatically at large scale](https://petabridge.com/blog/distributing-state-with-cluster-sharding/), for instance.

```cs
string childName = $"word-counter-{HttpUtility.UrlEncode(s.DocumentId.ToString())}";
IActorRef child = Context.Child(childName);
```

**All actor names must be URI-friendly, or Akka.NET will throw an `InvalidActorNameException` when calling `ActorOf`**!

For most simple strings they’re already URI-friendly but for things like URI’s themselves we’re going to call `System.Web.HttpUtility.UrlEncode` to guarantee that this won’t be a problem.

Next, we’re going to use the `WordCounterManager`’s `Context.Child` method to see if we already have a child actor named `word-counter-{Uri}`.

If a child actor exists, the returned `IActorRef` is valid; otherwise, it will be `ActorRefs.Nobody`. We use the `IsNobody()` extension method to check for this, as it also handles things like `null` checks for us:

```cs
if (child.IsNobody())
{
    // start the child if it doesn't exist
    child = Context.ActorOf(Props.Create(() => new DocumentWordCounter(s.DocumentId)), childName);
}		
```

If `IsNobody()` returns `true`, no child exists with this name, so we create one using `Context.ActorOf`.

This will create a new child actor named `childName` - and if that actor is ever terminated at any point in the future its entry will be removed from the `WordCounterManager`’s collection of children. Akka.NET guarantees that all ActorPaths within an ActorSystem are unique, ensuring no two child actors with the same name exist simultaneously under the same parent.

## Wrapping Up

Now that we’ve implemented half of the actors we need to power `AkkaWordCounter2`, it’s time we work on the remaining two actors and implement them. In the next lesson we’re going to learn [how to work with `async` and `await` inside actor message processing routines](https://petabridge.com/bootcamp/lessons/unit-1/async-await-actors/).

### Further Reading

- [Meet the Top Akka.NET Design Patterns](https://petabridge.com/blog/top-akkadotnet-design-patterns/)

[^1]: As of Akka.NET v1.5, [there is a slight performance advantage for `UntypedActor`s due to their ability to better leverage modern JIT techniques in .NET 7 and higher](https://petabridge.com/blog/dotnet7-pgo-performance-improvements/). However, we are working on addressing this in [v1.6 as part of our AOT support initiative](https://petabridge.com/blog/akkadotnet-v1.6-roadmap/). [↩](https://petabridge.com/bootcamp/lessons/unit-1/child-actors/#fnref:1) [↩2](https://petabridge.com/bootcamp/lessons/unit-1/child-actors/#fnref:1:1)

---

- Previous Lesson: [[4 Testing Actors with Akka.TestKit]]
- Next Lesson: [[6 Actors and Async and Await]]