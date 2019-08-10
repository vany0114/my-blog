---
layout: post
title: Simmy and Azure App Configuration
comments: true
excerpt: In the latest post, I introduced you Simmy and we saw all the benefits of chaos engineering and how Simmy helps us making chaos in our systems injecting faults, latency or custom behavior in order to make sure that our resilience strategies are correctly implemented and guarantee that our system is able to withstand turbulence conditions in a production environment. Also, I walked you through an example using Simmy in a distributed architecture, where one of the pieces that we had was a chaos settings microservice which took care of to store and get the chaos settings, however we found that that approach has a downside which could be important to consider, it adds extra latency since it has to retrieve the chaos settings from the API in every request. That’s why in this post we’re going to see how using Azure App Configuration we can manage our chaos settings avoiding to inject extra latency or additional overhead to our system.
keywords: "chaos engineering, resilience, resiliency, resiliency testing, fault injection, polly resilience, fault tolerance, fault based testing, fault tolerant, distributed systems, microservices, simmy, polly simmy, monkey, monkeys, chaos, simian army, inject latency, inject behavior, inject result, inject exception, chaos policies, monkey policies, transient-fault-handling, error-handling, transient fault handling, error handling, retry, circuit-breaker, circuit breaker, timeout, bulkhead isolation, fallback, PolicyWrap, netflix, simian, simian army, netflix simian army, .net, .net core, dotnet, dotnet core, azure app configuration, app configuration, azure"
---

In the [latest post](http://elvanydev.com/chaos-injection-with-simmy/), I introduced you [Simmy](https://github.com/Polly-Contrib/Simmy) and we saw all the benefits of [chaos engineering](http://principlesofchaos.org/) and how Simmy helps us making chaos in our systems injecting [faults](https://github.com/Polly-Contrib/Simmy#Inject-fault), [latency](https://github.com/Polly-Contrib/Simmy#inject-latency) or custom [behavior](https://github.com/Polly-Contrib/Simmy#inject-behavior) in order to make sure that our resilience strategies are correctly implemented and guarantee that our system is able to withstand turbulence conditions in a production environment. Also, I walked you through an [example](http://elvanydev.com/chaos-injection-with-simmy/#hands-on-lab) using Simmy in a distributed architecture, where one of the pieces that we had was a [chaos settings microservice](http://elvanydev.com/chaos-injection-with-simmy/#chaos-settings-microservice) which took care of to store and get the chaos settings, however we found that that approach has a downside which could be important to consider: it adds extra latency since it has to retrieve the chaos settings from the [API](http://elvanydev.com/chaos-injection-with-simmy/#how-does-it-get-the-chaos-settings) in every request. That's why in this post we're going to see how using [Azure App Configuration](https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview) we can manage our chaos settings avoiding to inject extra latency or additional overhead to our system.

## What is Azure App Configuration?

[Azure App Configuration](https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview) is a fully managed service offered by [Microsoft Azure](https://azure.microsoft.com/en-us/) to centralize the settings of your applications separately from your code, which is very convenient in distributed architectures where you have deployed your services across Clusters/VM's/Containers in the cloud. Azure App Configuration also provides [feature management](https://docs.microsoft.com/en-us/azure/azure-app-configuration/quickstart-feature-flag-aspnet-core) capabilities, but in this example, we're only going to focus on application settings management.

Azure App Configuration provides several clients, in our case we're going to use the [ASP.NET Core](https://docs.microsoft.com/en-us/azure/azure-app-configuration/quickstart-aspnet-core-app) one, which at the end of the day isn't another thing than an implementation of [Configuration Provider](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.2).

> Azure App Configuration is currently in public preview and it's free during the preview period.

## Create an app configuration store

The first thing we need to do is [provisioning](https://docs.microsoft.com/en-us/azure/azure-app-configuration/quickstart-aspnet-core-app#create-an-app-configuration-store) our App Configuration store, for which you're going to need an Azure subscription, if you don't have one, you can create it [for free](https://azure.microsoft.com/en-us/free/).

The second step is to create the settings, the easiest way to do that the first time, I think is using the [import tool](https://docs.microsoft.com/en-us/azure/azure-app-configuration/howto-import-export-data) from the portal which will allow you to import the chaos settings from a `json` file (among other options). After that, you should be able to see the chaos settings from the Azure Portal:

<figure>
  <img src="{{ '/images/azure-app-configuration-settings.png' | prepend: site.baseurl }}" alt=""> 
  <figcaption>Fig1. - Chaos Settings stored on App Configuration (Configuration explorer view)</figcaption>
</figure>

> If you don't want to import the settings, you still can use the [Chaos UI](#the-new-chaos-repository).

## The Refactor

I only updated a couple of things in order to introduce Azure App Configuration to our solution (that's a good sign that our design it's scalable, maintainable and good enough :smile:), so, let's start checking how the new component looks like in our architecture and who interacts with.

<figure>
  <img src="{{ '/images/Simmy-with-AppConfiguration.png' | prepend: site.baseurl }}" alt=""> 
  <figcaption>Fig2. - DUber Architecture using Simmy and Azure App Configuration</figcaption>
</figure>

### Setting up Azure App Configuration

As I mentioned before, Azure App Configuration for ASP.NET Core, implements a *Configuration Provider*, which in this case it will manage the settings using our configuration store created previously in Azure. Let's see how to do so.

#### Web Projects

First of all, we need to reference the `Microsoft.Azure.AppConfiguration.AspNetCore` package in our web projects, which are: `Duber.WebSite`, `Duber.Trip.API` and `Duber.Invoice.API`, then we need to tell them that we want to get our chaos settings not from an `appsettings.json` file but from *Azure App Configuration*, so we need to update our [Program.cs](https://github.com/vany0114/chaos-injection-using-simmy/blob/f7e1304722d83a729660ca8785c9ed0e1d31f3b0/src/Web/Duber.WebSite/Program.cs#L49) like this:

```c#
public static IWebHost BuildWebHost(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        ...
        .ConfigureAppConfiguration((builderContext, config) =>
        {
            var settings = config.Build();
            if (settings.GetValue<bool>("UseAzureAppConfiguration"))
            {
                config.AddAzureAppConfiguration(options =>
                {
                    options.Connect(settings["ConnectionStrings:AppConfig"])
                        .ConfigureRefresh(refresh =>
                        {
                            refresh.Register("GeneralChaosSetting:Sentinel", refreshAll: true);
                            refresh.SetCacheExpiration(TimeSpan.FromSeconds(5));
                        });
                });
            }
        })
        ...
        .Build();
```

We're using a flag called `UseAzureAppConfiguration` which allows the system to identify whether or not to use *Azure App Configuration* through the method `AddAzureAppConfiguration` to get the chaos settings, or use the chaos seettings microservice instead. (That flag is stored in the `appsettings.json` file). 

##### The connection
There are two ways in order to connect to our app configuration store, we can use either [Managed Identity](https://docs.microsoft.com/en-us/azure/azure-app-configuration/howto-integrate-azure-managed-service-identity) or a connection string. In this example we're using the second one, and as you can see we're passing the connection string named  `AppConfig` which is stored into the `appsettings.json` file as well.

##### Refreshing the settings
Another great advantage of Azure App Configuration is the ability to refresh the settings when they have changed. In terms of configuration, we can do that through the method `ConfigureRefresh`. We can tell it which settings we want to refresh once they change using the `Register` method, but what happens when we need to refresh a lot of settings? or even a more complicated scenario, when the settings we need to refresh are dynamic? first, we don't want to hard-code a lot of settings individually because when we have a new one it means we need to register it then we're going to need to deploy again, besides in a [dynamic scenario](https://github.com/vany0114/chaos-injection-using-simmy/blob/5e7c1b36181ee04e887962aa40a88f8773caa35c/src/Infrastructure/Duber.Infrastructure.Chaos/GeneralChaosSetting.cs#L42) you don't have a fixed number of settings. 

So, to achieve that, we need to use a "sentinel" key-value in the app configuration and trigger a reload all of our configuration when that value changes. That's why we set the `refreshAll` parameter to `true`. We need to update the sentinel whenever we want the app to pick up the changes that we made in the app configuration. The sentinel could be anything, just touching that value in app configuration will trigger the refresh for the rest of the settings. The app will see that it got updated and doesn't need to worry about the actual value. (Although, it will get the new value for the sentinel in case we use one who's value you do care about)

##### Caching the settings
Azure App Configuration allows us to cache the settings as well, the default value is 30 seconds if you don't specify it, but you can set the cache expiration as you need it. The minimum value is one second.

##### Azure App Configuration Middleware
The other little change we need to do in our web projects is over the [Startup.cs](https://github.com/vany0114/chaos-injection-using-simmy/blob/f7e1304722d83a729660ca8785c9ed0e1d31f3b0/src/Web/Duber.WebSite/Startup.cs#L96) file. It's quite easy, we only need to use the azure app configuration middleware:

```c#
if (Configuration.GetValue<bool>("UseAzureAppConfiguration"))
    app.UseAzureAppConfiguration();
```

#### Chaos Settings Factory

The second change I made (and the most important one), was over the *Chaos Settings Factory* which takes care of getting the chaos settings, resolving at run-time the *how*. So, in our [original implementation](http://elvanydev.com/chaos-injection-with-simmy/#how-does-it-get-the-chaos-settings), we returned a `Task<GeneralChaosSetting>` which was resolved by the [Chaos API](http://elvanydev.com/chaos-injection-with-simmy/#chaos-settings-microservice).

Now, we merely getting the chaos settings from [App Configuration](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/index?view=aspnetcore-2.2), in this case using *Azure App Configuration* as Configuration Provider. Having said that, it's just matter of using the right [Option pattern](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-2.2), in our case, we want to refresh the configuration every time it changes, so, we will use the [IOptionsSnapshot](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-2.2#reload-configuration-data-with-ioptionssnapshot) approach. 

```c#
public static IServiceCollection AddChaosApiHttpClient(this IServiceCollection services, IConfiguration configuration)
{
    ...
    if (configuration.GetValue<bool>("UseAzureAppConfiguration"))
        services.Configure<GeneralChaosSetting>(configuration.GetSection("GeneralChaosSetting"));

    services.AddScoped<Lazy<Task<GeneralChaosSetting>>>(sp =>
    {
        if (configuration.GetValue<bool>("UseAzureAppConfiguration"))
        {
            var chaosSettings = sp.GetRequiredService<IOptionsSnapshot<GeneralChaosSetting>>();
            return new Lazy<Task<GeneralChaosSetting>>(() => Task.FromResult(chaosSettings.Value), LazyThreadSafetyMode.None);
        }
        ...
    });

    return services;
}
```

As you can see, we're configuring the `GeneralChaosSetting` options from the [GeneralChaosSetting](https://github.com/vany0114/chaos-injection-using-simmy/blob/f7e1304722d83a729660ca8785c9ed0e1d31f3b0/src/Infrastructure/Duber.Infrastructure.Chaos/IoC/ServiceCollectionExtensions.cs#L21) configuration section, then we just resolving the `Task<GeneralChaosSetting>` just returning directly the `IOptionsSnapshot<GeneralChaosSetting>` [object's value](https://github.com/vany0114/chaos-injection-using-simmy/blob/f7e1304722d83a729660ca8785c9ed0e1d31f3b0/src/Infrastructure/Duber.Infrastructure.Chaos/IoC/ServiceCollectionExtensions.cs#L28). It means that in every request we're getting the chaos settings from the app configuration rather than from the *Chaos API*, which will help us to avoid to introduce extra latency.

But, how Azure App Configuration does that magic? I was speaking with [Jimmy Campbell](https://github.com/jimmyca15) who is part of the Azure App Configuration team, and he told me that in the web scenario, one thing that they found is that polling on a timer was not exactly the best solution (when the `WatchAndReloadAll` method existed in the very beginning of Azure App Configuration), often times this can lead to inactive apps putting load on the app configuration instance as well as extraneous Network and CPU usage on the client. Then, they updated the web app scenario to be smart and reload when the app gets a request (if a refresh is scheduled), but here is the thing, it performs the refresh in a [fire-and-forget](https://www.enterpriseintegrationpatterns.com/patterns/conversation/FireAndForget.html) manner, which means we don't need to wait for a response thus it won't add extra latency!

> Azure App Configuration isn't still Open Source.

Up to this point, we've done integrating *Azure App Configuration* to our solution, wasn't a big deal, right? So, you can just hit F5 and try it out, and see how it gets and refreshes the settings from the cloud!

#### The new Chaos Repository

Only the changes mentioned above were required to introduce Azure App Configuration in our solution, however, I decided to make a new [IChaosRepository](https://github.com/vany0114/chaos-injection-using-simmy/blob/master/src/Application/Duber.Chaos.API/Infrastructure/Repository/IChaosRepository.cs) implementation in order to keep our [Chaos API](http://elvanydev.com/chaos-injection-with-simmy/#chaos-settings-microservice) working thus our [Chaos UI](http://elvanydev.com/chaos-injection-with-simmy/#the-chaos-ui) too. So I created the [AzureAppConfigurationRepository](https://github.com/vany0114/chaos-injection-using-simmy/blob/master/src/Application/Duber.Chaos.API/Infrastructure/Repository/AzureAppConfigurationRepository.cs) which basically wraps the [Azure App Configuration SDK](https://github.com/Azure/azure-sdk-for-net/tree/master/sdk/appconfiguration/Azure.ApplicationModel.Configuration).

Notice that in the [GetChaosSettingsAsync](https://github.com/vany0114/chaos-injection-using-simmy/blob/5e7c1b36181ee04e887962aa40a88f8773caa35c/src/Application/Duber.Chaos.API/Infrastructure/Repository/AzureConfigurationAppRepository.cs#L33) we're merely returning the `IOptionsSnapshot<GeneralChaosSetting>` object's value which we're injecting into the [constructor](https://github.com/vany0114/chaos-injection-using-simmy/blob/5e7c1b36181ee04e887962aa40a88f8773caa35c/src/Application/Duber.Chaos.API/Infrastructure/Repository/AzureConfigurationAppRepository.cs#L27).

Also, notice that in the `UpdateChaosSettings` method, the last thing we do is updating the [Sentinel](https://github.com/vany0114/chaos-injection-using-simmy/blob/5e7c1b36181ee04e887962aa40a88f8773caa35c/src/Application/Duber.Chaos.API/Infrastructure/Repository/AzureConfigurationAppRepository.cs#L67) in order to trigger the refresh for all the settings.

> All the dirty code related to [reflection](https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection) into that repository is due to *Azure App Configuration* does not allow you to set/delete in batch or in a *Generic* way just passing an object, that's why we need to set/delete every setting one by one.

## Wrapping up

So far we've seen two different approaches to manage our chaos settings, using [our own chaos settings microservice](http://elvanydev.com/chaos-injection-with-simmy/#chaos-settings-microservice) and the one we described in this post using [Azure App Configuration](https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview), but you might use other sources to manage the chaos settings: a relational/NoSQL database or even the regular way using a configuration file, etc, whatever makes more sense for you in the context of your project/application, the important thing here is what we've seen about how powerful is [Simmy](https://github.com/Polly-Contrib/Simmy) working together with [Polly](https://github.com/App-vNext/Polly) and how you can use a [Context-driven behavior](https://github.com/Polly-Contrib/Simmy#context-driven-behaviour) to control the chaos injected in a dynamic and targeted manner.

Personally I would take advantage of both approaches, taking the best of both worlds, for example, I'd still use the *Chaos Settings API* to automate the chaos injection not only through the [WatchMonkey](http://elvanydev.com/chaos-injection-with-simmy/#watchmonkey) but through the deployments using  [Azure DevOps](https://azure.microsoft.com/es-es/services/devops/), or another DevOps tool like [Octopus](https://octopus.com/), [TeamCity](https://www.jetbrains.com/teamcity/), etc, to enable the automatic chaos after a release as we mentioned in the previous article. Besides to be used for the [Chaos UI](http://elvanydev.com/chaos-injection-with-simmy/#the-chaos-ui) mostly for write operations. On the other hand, I'd use *Azure App Configuration* from the clients interested to get the chaos settings to inject chaos at some point in time, in our case on web site and microservices.

 I encourage you all to start making experiments using Simmy (if you haven't started already) and see what is the approach that works better for you or even to try out your own approaches, we would love to hear about that. We hope Simmy will continue growing with the help and adoption of the community, meanwhile stay tuned because we're working on enhancements and new features! :monkey_face:

> Take a look at the whole implementation on my GitHub repo: https://github.com/vany0114/chaos-injection-using-simmy