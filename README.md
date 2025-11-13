This is an open-source mirror of the [Akka.NET Bootcamp Lessons](https://petabridge.com/bootcamp/lessons/).

## Table of Contents
- [Akka.NET Bootcamp Lesson Overview](./Akka.NET%20Bootcamp%20Lesson%20Overview.md)
- Unit 1: Basics of Akka.NET
	- [1 Why Learn Akka.NET](./Unit-1%20Basics%20of%20Akka.NET/1%20Why%20Learn%20Akka.NET.md)
	- [2 Our First Akka.NET Application](./Unit-1%20Basics%20of%20Akka.NET/2%20Our%20First%20Akka.NET%20Application.md)
	- [3 Effective Actor Messaging](./Unit-1%20Basics%20of%20Akka.NET/3%20Effective%20Actor%20Messaging.md)
- Unit 2: Building Local Akka.NET Applications
	- [1 Using Akka.Templates to Create New Projects](./Unit-2%20Building%20Local%20Akka.NET%20Applications/1%20Using%20Akka.Templates%20to%20Create%20New%20Projects.md)
	- [2 Actor Hierarchies and Domain Modeling](./Unit-2%20Building%20Local%20Akka.NET%20Applications/2%20Actor%20Hierarchies%20and%20Domain%20Modeling.md)
	- [3 Behavior Switching and Receive Timeouts.md](./Unit-2%20Building%20Local%20Akka.NET%20Applications/3%20Behavior%20Switching%20and%20Receive%20Timeouts.md)
	- [4 Testing Actors with Akka.TestKit.md](./Unit-2%20Building%20Local%20Akka.NET%20Applications/4%20Testing%20Actors%20with%20Akka.TestKit.md)
	- [5 Working with Child Actors.md](./Unit-2%20Building%20Local%20Akka.NET%20Applications/5%20Working%20with%20Child%20Actors.md)
	- [6 Actors and Async and Await.md](./Unit-2%20Building%20Local%20Akka.NET%20Applications/6%20Actors%20and%20Async%20and%20Await.md)
	- [7 Akka.Hosting, Routers, and Dependency Injection.md](./Unit-2%20Building%20Local%20Akka.NET%20Applications/7%20Akka.Hosting,%20Routers,%20and%20Dependency%20Injection.md)
	- [8 Integration Testing with Akka.Hosting.TestKit.md](./Unit-2%20Building%20Local%20Akka.NET%20Applications/8%20Integration%20Testing%20with%20Akka.Hosting.TestKit.md)
	- [9 Using IOptions and Microsoft.Extensions.Configuration.md](./Unit-2%20Building%20Local%20Akka.NET%20Applications/9%20Using%20IOptions%20and%20Microsoft.Extensions.Configuration.md)
	- [10 Writing Sagas with Actors, Message Stashing, and IWithTimers.md](./Unit-2%20Building%20Local%20Akka.NET%20Applications/10%20Writing%20Sagas%20with%20Actors,%20Message%20Stashing,%20and%20IWithTimers.md)
- [Future Bootcamp Units and Lessons](./Future%20Bootcamp%20Units%20and%20Lessons.md)

## Why This Mirror Exists
The reason for this mirror is that I always struggled with the lack of a night mode when watching this course. It was fine during the day, but at night it was as blinding as a flashbang.

Another major pain point was the unintuitive navigation:

1. The opened page wasn't highlighted in the left-hand navigation, making it difficult to know where I was.
2. The courses weren't numbered; they were only linked by Previous & Next Lessons.

These problems made watching this course quite a painful experience.

So, I organized it into Markdown format and stored it in Obsidian, numbering each course.

This largely solves the above problems.

By the way, during the organization process, I also discovered some issues with the Mermaid graph rendering position.

## Code Style

To make the code look more compact, I changed the code style in the tutorial to [K&R style](https://en.wikipedia.org/wiki/Indentation_style#K&R). However, if the statement before the `{` is split into multiple lines, I will revert to [Allman style](https://en.wikipedia.org/wiki/Indentation_style#Allman_style) to ensure readability.

> [!TIP] The difference between K&R Style and Allman Style:
> - K&R Style: The opening `{` is placed at the end of the line preceding the block.
> - Allman Style: The opening `{` is placed on a new line, aligned with the block's indentation level.

For example:

```cs
void Hello(int paramA,
	int paramB,
	int paramC)
{
	// code
}
```

The reason for this is that because the `{` in K&R style is at the end of the line, it is not visually obvious. Therefore, this style degenerates into something like Python where code hierarchy is identified by indentation. However, the above code has two types of indentation, which disrupts this indentation hierarchy, resulting in very low readability. Using Allman style in this case maintains excellent readability.

This is what it looks like if K&R style is still used in this situation; you can see how bad the readability is:

```cs
void Hello(int paramA,
	int paramB,
	int paramC) {
	// code
}
```

Although this style is somewhat inconsistent, it's actually the only exception, and I believe it represents a balance between code compactness and readability. This is also the style I use in actual projects.

## Acknowledgments

Since this Markdown version was very helpful to me, I thought it would be helpful to others as well, so I'm open-sourcing it.

Finally, I would like to express my sincere gratitude to Petabridge for developing such a wonderful Actor framework; you guys have done a great job. Thank you for your great contributions to the .NET community!
