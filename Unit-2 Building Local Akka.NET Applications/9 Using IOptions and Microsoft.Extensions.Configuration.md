In this lesson we’re going to configure `AkkaWordCounter2` using Microsoft.Extensions.Configuration and the `IOptions` pattern.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Amm7jmTifX8" title="Tutorial: Creating Professional, Local Akka.NET Applications (Bootcamp 2.0 - Unit 1)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
_Starts at the appropriate timestamp for this lesson_

## Configuring `AkkaWordCounter2`

One thing we have not resolved yet: how are we going to tell `AkkaWordCounter2` which documents to count words from?

In this instance, we are going to use Microsoft.Extensions.Configuration to do it.

First, inside our `AkkaWordCounter2.App/Config` folder, please add a new file called `WordCounterSettings.cs` and then type the following:

```cs
public class WordCounterSettings {
    public string[] DocumentUris { get; set; } = [];
}
```

This is a very simple strongly typed settings class that we’re going to use to configure our application - but the next thing we’re going to do is add a new `appSettings.json` file to `AkkaWordCounter2.App`:

```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Debug",
            "System": "Information",
            "Microsoft": "Information"
        }
    },
    "WordCounter": {
        "DocumentUris": [
            "https://raw.githubusercontent.com/akkadotnet/akka.net/dev/README.md",
            "https://getakka.net/"
        ]
    }
}
```

We’re going to add some code to our `IHostBuilder` to parse this `appSettings.json` file into our `WordCounterSettings`class in a minute - that’s where we’re headed.

### Making Sure `appSettings.json` Gets Copied

However, one other thing we need to do is edit the `AkkaWordCounter2.App.csproj` file and make sure that we always copy the `appSettings.json` to our output folder when we run the application. Otherwise none of our settings will be effective.

Add the following XML to `AkkaWordCounter2.App.csproj` or set these values via your preferred IDE dialog window:

```xml
<ItemGroup>
	<Content Include="appsettings.json">
	  <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
	</Content>
</ItemGroup>
```

### Leveraging the `IOptions` Pattern

We’re going to do this [using the `IOptions` pattern from Microsoft.Extensions.Configuration](https://learn.microsoft.com/en-us/dotnet/core/extensions/options), since that’s considered to be a generally robust practice for application configuration.

We already have our `WordCounterSettings` class defined inside `AkkaWordCounter2.App/Config/WordCounterSettings.cs` - that is our base options class. Now we’re going to add some configuration _validation_ code, designed to ensure that our application fails fast at startup if it’s misconfigured - another good habit to get into.

Inside `AkkaWordCounter2.App/Config/WordCounterSettings.cs` please type the following:

```cs
public sealed class WordCounterSettingsValidator : IValidateOptions<WordCounterSettings> {
    public ValidateOptionsResult Validate(string? name, WordCounterSettings options) {
        var errors = new List<string>();
        
        if (options.DocumentUris.Length == 0) {
            errors.Add("DocumentUris must contain at least one URI");
        }
        
        if(options.DocumentUris.Any(uri => !Uri.IsWellFormedUriString(uri, UriKind.Absolute))) {
            errors.Add("DocumentUris must contain only absolute URIs");
        }
        
        return errors.Count == 0
            ? ValidateOptionsResult.Success
            : ValidateOptionsResult.Fail(errors);
    }
}

public static class WordCounterSettingsExtensions {
    public static IServiceCollection AddWordCounterSettings(this IServiceCollection services) {
        services.AddSingleton<IValidateOptions<WordCounterSettings>, WordCounterSettingsValidator>();
        services.AddOptionsWithValidateOnStart<WordCounterSettings>()
            .BindConfiguration("WordCounter");
        
        return services;
    }
}
```

If your IDE doesn’t auto-suggest the relevant namespaces for you to include, here’s what the top of the `AkkaWordCounter2.App/Config/WordCounterSettings.cs` file should look like after you add this:

```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
```

What this code does:

1. Declares a `WordCounterSettingsValidator` that will be used to validate the fully-parsed content of our `WordCounterSettings` at startup.
2. Creates an `IServiceCollection` extension method that registers the `WordCounterSettingsValidator` as a singleton; binds the `WordCounterSettings` to the `WordCounter` section of our `appSettings.json` file[1](https://petabridge.com/bootcamp/lessons/unit-1/msft-configuration/#fn:1); and then instructs the `IHost` to validate our `WordCounterSettings` immediately upon host start.

### Wiring Configuration to `IHost`

We’re almost done configuring `AkkaWordCounter2` - the last things we need to do are to add the following to `AkkaWordCounter2.App/Program.cs`:

First, we need to add our configuration sources to the host builder:

```cs
hostBuilder
    .ConfigureAppConfiguration((context, builder) => {
        builder
            .AddJsonFile("appsettings.json", optional: true)
            .AddJsonFile($"appsettings.{context.HostingEnvironment.EnvironmentName}.json", 
                    optional: true)
            .AddEnvironmentVariables();
    })
```

Next, we need to add our `IOptions` to the `IServiceCollection` so they can be accessed by our application:

```cs
 services.AddWordCounterSettings();
```

Your `Program.cs` should look like this:

```cs
using Akka.Hosting;
using AkkaWordCounter2.App;
using AkkaWordCounter2.App.Actors;
using AkkaWordCounter2.App.Config;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Options;

var hostBuilder = new HostBuilder();

hostBuilder
    .ConfigureAppConfiguration((context, builder) => {
        builder
            .AddJsonFile("appsettings.json", optional: true)
            .AddJsonFile($"appsettings.{context.HostingEnvironment.EnvironmentName}.json", 
                    optional: true)
            .AddEnvironmentVariables();
    })
    .ConfigureServices((context, services) => {
        services.AddWordCounterSettings();
        services.AddHttpClient(); // needed for IHttpClientFactory
        services.AddAkka("MyActorSystem", (builder, sp) => {
            builder
                .ConfigureLoggers(logConfig => {
                    logConfig.AddLoggerFactory();
                });
        });
    });

var host = hostBuilder.Build();

await host.RunAsync();
```

We’re all finished with configuration.

## Wrapping Up

With configuration out the way, we are onto our final lesson of Unit 1: building the `WordCountJobActor`, a small “saga” actor that uses message stashing to orchestrate work between multiple other actors.

### Further Reading

- [Akka.NET Application Design: Don’t Create Bespoke Frameworks; Use Repeatable Patterns](https://youtu.be/X1Tg4R2JFMQ?si=d4HGboPPLER6ZQKH)
- [No Hocon, No Lighthouse, No Problem: Akka.Hosting, Akka.HealthCheck, and Akka.Management](https://www.youtube.com/watch?v=U2IIfLKxprM)

[^1]: This will also work with any other configuration sources we might specify later, such as environment variables or `appSettings.{ENVIRONMENT}.json` - so long as those [follow the Microsoft.Extensions.Configuration conventions](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration-providers). [↩](https://petabridge.com/bootcamp/lessons/unit-1/msft-configuration/#fnref:1)

---

- Previous Lesson: [[8 Integration Testing with Akka.Hosting.TestKit]]
- Next Lesson: [[10 Writing Sagas with Actors, Message Stashing, and IWithTimers]]

