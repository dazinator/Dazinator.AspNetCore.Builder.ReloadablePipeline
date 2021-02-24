## Features

`ResponsiveCore` provides features to enable your .net core applications to be more responsive to changes at runtime, so you don't have to restart the application in order for changes to take effect. 
It does this whilst trying to be as minimally invasive in your code base as possible.

Features:

- Middleware: Define middleware pipelines that can be dynamically reloaded / rebuilt from latest config at runtime, without dropping requests.
- Background Services: Allow your existing `IHostedService`s to be dynamically started and stopped based on your own logic to determine whether they should be currently enabled or not.

[![Build Status](https://darrelltunnell.visualstudio.com/Public%20Projects/_apis/build/status/dazinator.Dazinator.ResponsiveCore?branchName=develop)](https://darrelltunnell.visualstudio.com/Public%20Projects/_build/latest?definitionId=16&branchName=develop)

Thanks:

- [Changify](https://github.com/dazinator/Changify) - easily build composite change token producers.

## Reloadable Middleware Pipelines

1. Add the `Dazinator.ResponsiveCore.ReloadablePipeline` nuget package to your project.
2. Configure an `Options` class, and then build a middleware pipeline that from it that will be rebuilt whenever `IOptionsMonitor` detects a change:
 
```csharp

    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        // For more information on how to configure your application, visit https://go.microsoft.com/fwlink/?LinkID=398940
        public void ConfigureServices(IServiceCollection services)
        {
            var configSection = Configuration.GetSection("Pipeline");
            services.Configure<PipelineOptions>(configSection);
        }


        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            // Note: Use vs Run (latter is terminal, former is not)
            // make a change to appsettings.json "Pipelines" section and watch log output in console on furture requests.
             app.UseReloadablePipeline((options) =>
            {
                var monitor = app.ApplicationServices.GetRequiredService<IOptionsMonitor<PipelineOptions>>();
                var tokenProducer = new ChangeTokenProducerBuilder() // I am choosing to use the Changify library to build a Func<IChangeToken> as is easier - you don't have to.
                                  .IncludeSubscribingHandlerTrigger((trigger) => monitor.OnChange((o, n) => trigger()))
                                  .Build(out var disposable);

                options.SetChangeTokenProducer(tokenProducer, disposable)
                       .ConfigureMiddlewarePipeline((app) =>
                       {
                           ConfigureReloadablePipeline(app, env, monitor.CurrentValue);
                       });
            });      

            app.UseWelcomePage();
        }

        private void ConfigureReloadablePipeline(IApplicationBuilder appBuilder, IWebHostEnvironment environment, PipelineOptions options)
        {
            var logger = appBuilder.ApplicationServices.GetRequiredService<ILogger<Startup>>();
            logger.LogInformation("Building reloadable pipeline from current options!");

            if (options.UseDeveloperExceptionPage)
            {
                appBuilder.Use(async (context, onNext) =>
                {
                    var logger = context.RequestServices.GetRequiredService<ILogger<Startup>>();
                    logger.LogInformation("Using dev middleware!");
                    await onNext();
                });
            }
            else
            {
                appBuilder.Use(async (context, onNext) =>
                {
                    var logger = context.RequestServices.GetRequiredService<ILogger<Startup>>();
                    logger.LogInformation("Not using dev middleware..");
                    await onNext();
                });
            }
        }
    }


```

### How do I signal the pipeline to rebuild for other sorts of changes - for example a button click?

You can supply a `Func<IChangeToken>` (change token producer) that will trigger a reload whenever an `IChangeToken` is signalled.

### Rebuild Strategies

The default strategy for pipeline rebuilds, is to rebuild it within a lock at the point its invalidated. 
Once the new pipeline has been build, the new instance is swapped in for the old instance (no locking), and new requests are now pushed through the new instance of the pipeline instead of the old.
This strategy means there is no downtime for requests - requests continue to be processed through the old pipeline until the new one is swapped in without any lock delaying requests.
Another optional strategy that is provided out of the box, allows instead for the pipeline to be lazily re-built in line with the next request. This means other requests that occur whilst a rebuild is in progress may queue behind the lock until the rebuild completes, and for this reason - its not the default strategy - however it might prove useful if for example you'd like access to a current httpcontext (request) during the pipeline build itself - which is ordinarly not achievable.

To override the default strategy, set the `RebuildStrategy` on the `options` to an instance of `IRebuildStrategy`.


```csharp
  app.UseReloadablePipeline((options) =>
            {
              options.RebuildStragety = new RebuildOnDemandStrategy();
              ...
```

## Responsive Hosted Services

Suppose you have an `IHostedService` that you want to be able to stop and start at runtime without restarting the application.

1. Add the `Dazinator.ResponsiveCore.EnabledHostedService` package to your project.

2. Create an `Options` class that you can then bind to config to represent whether the service is enabled or disabled.

```csharp
    public class HostedServiceOptions
    {
        public bool Enabled { get; set; }
    }
```

and

```csharp
    services.Configure<HostedServiceOptions>(Configuration.GetSection("HostedService"));
```

3. Replace your call to `services.AddHostedService` with  `AddEnabledHostedService`:

```csharp
     services.AddEnabledHostedService<HostedService>(o =>
            {
                var monitor = o.Services.GetRequiredService<IOptionsMonitor<HostedServiceOptions>>();
                var tokenProducer = new ChangeTokenProducerBuilder()
                                   .IncludeSubscribingHandlerTrigger((trigger) => monitor.OnChange((o, n) => trigger()))
                                   .Build(out var disposable);

                o.ServiceOptions.SetChangeTokenProducer(tokenProducer, disposable)
                                .SetShouldBeRunningCheck(() => monitor.CurrentValue?.Enabled ?? false);

            });

```

Note: Above I am basically configuring an `IChangeToken` producer, and a delegate that returns a boolean that will be run whenever the `IChangeToken` is signalled.
I am using the `Changify` library in the sample and in the tests to build the token producer, but you can provide a `Func<IChangeToken>` however you see fit.

4. Start your application with that config setting as `false`. Your service will not start. Whilst the application is running, change the config to be `true` and save that change. The change is detected and your service will now start. Again, whilst your application is running, change the config back to `false` - your service will be stopped. Repeat this ad infinitum - if you have the time, all the while basking in the glory of this responsivity.