# General CSharp
## Record
```
// declare
record Person(string Name, int Age);
// create
var alice = new Person("Alice", 30);
// change name
var bob = alice with { Name = "Bob" };
// copy
Var bob2 = alice with { }
// value-based equality
var alice2 = new Person ("Alice", 30);
Console.WriteLine(alice == alice2); // Output: True (values are equal)
```

## Pattern Matching
```
// is expression
if (shape is Point p)
{
    Console.WriteLine($”shape is a point at ({p.X, p.Y})");
}
else if (shape is Circle c)
{
    Console.WriteLine($”shape is a circle with radius {c.Radius}");
}
// check range
int number = 4;
bool between3And5 = number is >= 3 and <= 5 // output: true
// matching object properties
bool foundAlice = person is { Name: "Alice", Age: 30 } // output: true
// switch expression
string result = shape switch
{
    Point p => $"Point: ({p.X}, {p.Y})",
    Circle c => $"Circle: ({c.X}, {c.Y}), radius {c.Radius}",
    _ => "Unknown shape"
};
```
# Collection
```
// prior to dotnet 8
int[] numbers = { 1, 2, 3, 4, 5 };

// from dotnet 8
int[] first = [ 1, 2, 3, 4, 5];
int[] second = [6, 7];

var both = [..first, ..second];
```
# Raw String Literals
```
var letter = $"""
              Dear {Name},
             
              This letter is to inform …
              """;
```

# General dotnet 

## Tasks
Run tasks in parallels
```
var task1 = SomeTaskAsync();
var task2 = AnotherTaskAsync();
// await for both task to complete
await Task.WhenAll(task1, task2);
```
Run task sequentially
```
await SomeTaskAsync();
await AnotherTaskAsync();
```
Execute synchrous code using task
```
var task = Task.Run(() => {
    // some code
});

await task;
```
using `TaskCompletionSource<T>`
```
TaskCompletionSource<int> tcs = new ();

// Start a separate task that simulates an asynchronous operation
Task.Run(() =>
{
    Console.WriteLine("Task Thread ID: " + Thread.CurrentThread.ManagedThreadId);

    // Simulate some work
    Thread.Sleep(2000);

    // Set the result of the task
    tcs.SetResult(42);
});

// Await the task from TaskCompletionSource
int result = await tcs.Task;
```
using `CancellationToken`
```
sealed class SomeService
{
    private TaskCompletionSource<int> _tcs;
    private CancellationToken _token;

    public Task<int> DoSomethingAsync(CancellationToken token)
    {
        _tcs = new TaskCompletionSource<int>();
        _token = token;
        // run something in background
        Task.Run(MyAction);
        return _tcs.Task;
    }

    public void MyAction()
    {
        for(var i = 0; i<1000000; i++)
        {
            // do something
            Thread.Sleep(10);
            _token.ThrowIfCancellationRequested();
        }
        // set the result 
        _tcs.SetResult(100);
    }
}

// usage (create service, run task and cancel it after 1 second)
var s = new SomeService();
var cancellationTokenSource = new CancellationTokenSource();
var task = s.DoSomethingAsync(cancellationTokenSource.Token);
await Task.Delay(1000);
cancellationTokenSource.Cancel(); // OperationCanceledException would occurred here
```

## Send messages via `WeakReferenceMessenger`
declare message
```
record MyMessage(string Content);
```
send
```
WeakReferenceMessenger.Default.Send(new MyMessage("Helo"));
```
Receiving by register a closure
```
WeakReferenceMessenger.Default.Register<MyMessage>(this, async (receipient, message) =>
{
    Console.WriteLine(message.Content);
});
```
or register with class
```
public sealed class MyReceiver : IRecipient<MyMessage>
{
    public MyReceiver() 
    {
        WeakReferenceMessenger.Default.Register<MyMessage>(this);
    }

    public void Receive(MyMessage message)
    {
        Console.WriteLine(message.Content);
    }
}
```

## EF Core 
Bulk update
```
// set all customers over 50 to inactive
dbContext.Customers
    .Where(x => x.Age > 50)
    .ExecuteUpdate(x => x.SetProperty(p => p.Active, p => false));
```
# Web

## dotnet 8 Global exception handler 
```
internal sealed class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(
            exception, "Exception occurred: {Message}", exception.Message);

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Server error"
        };

        httpContext.Response.StatusCode = problemDetails.Status.Value;

        await httpContext.Response
            .WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```
Register the above class in `Program.cs`
```
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
...
var app = builder.Build();
app.UseExceptionHandler();
```

## prior to dotnet 8 via convention based middleware
```
public sealed class GlobalExceptionHandlerMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlerMiddleware> _logger;

    public GlobalExceptionHandlerMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlerMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            // execute code before request

            await _next(context);

            // execute code after request
        }
        catch (Exception ex)
        {
            // Handle exceptions here
            logger.LogError(ex, "Unhandled error");
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync("{ code = 123, error = \"server error\" }");
        }
    }
}
```
register
```
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
```

## prior to dotnet 8 via Factory based middleware
```
public sealed class GlobalExceptionHandlerMiddleware : IMiddleware
{
    private readonly ILogger<GlobalExceptionHandlerMiddleware> _logger;

    public GlobalExceptionHandlerMiddleware(ILogger<GlobalExceptionHandlerMiddleware> logger)
    {
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        try
        {
            // execute code before request

            await next(context);

            // execute code after request
        }
        catch (Exception ex)
        {
            // Handle exceptions here
            logger.LogError(ex, "Unhandled error");
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync("{ code = 123, error = \"server error\" }");
        }
    }
}
```
register
```
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
```
## prior to dotnet 8 via request delegate based middleware
```
app.UseExceptionHandler(errorApp =>
    {
        errorApp.Run(async context =>
        {
            var exceptionHandlerPathFeature = context.Features.Get<IExceptionHandlerPathFeature>();
            var exception = exceptionHandlerPathFeature?.Error;
            var logger = errorApp.ApplicationServices.GetService<ILogger<Program>>();
            logger.LogError(exception, "Unhandled error");
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsync("{ code = 123, error = \"server error\" }");
        });
    });
```

## health check
Add appropriate packages
```
    <PackageReference Include="AspNetCore.HealthChecks.Redis" Version="6.0.4" />
    <PackageReference Include="AspNetCore.HealthChecks.NpgSql" Version="6.0.2" />
```
Configure in app
```
builder.Services.AddHealthChecks()
    .AddRedis(redisConnectionString)
    .AddNpgSql(npgSqlConnectionString);

var app = builder.Build();
app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();

        endpoints.MapHealthChecks("/_health", new HealthCheckOptions
        {
            ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
        });
    });
```

## rate limit

```
builder.Services.AddRateLimiter(ConfigureRateLimiterOptions);
var app = builder.Build();
app.UseRateLimiter();

private void ConfigureRateLimiterOptions(RateLimiterOptions rlo)
{
    rlo.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // allow 200 requests per minute from each IP address and accumulate up to 1000 tokens
    rlo.AddPolicy("default", httpContext => RateLimitPartition.GetTokenBucketLimiter(
        partitionKey: httpContext.Connection.RemoteIpAddress?.ToString(), 
        factory: _ => new TokenBucketRateLimiterOptions
        {
            AutoReplenishment = true,
            TokenLimit = 1000,
            ReplenishmentPeriod = TimeSpan.FromMinutes(1),
            TokensPerPeriod = 200
        }));

    rlo.AddConcurrencyLimiter("concurrency_1", opt =>
    {
        opt.PermitLimit = 1;
        opt.QueueLimit = 1;
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
}
```

# Xamarin Forms
## Using `CommandFactory`
Can handle multile execution (i.e double clicks, re-entry) and centralise exception handling

Add package
```
<PackageReference Include="Xamarin.CommunityToolkit" Version="2.0.6" />
```
Create `ExceptionHanlder` class to handle exception centrally
```
public static class ExceptionHandler
{
    public static void Log(Exception ex) 
    {
        Console.WriteLine(ex.ToString());
    }

    public static void DisplayAlert(Exception ex, string title, string message)
    {
        Log(ex);
        Application.Current.MainPage.DisplayAlert(title, message, "DISMISS");
    }
} 
```
Using the `CommandFactory`
```
public sealed class ViewModel
{
    public IAsyncCommand ButtonClickedCommand { get; }

    public ViewModel()
    {
        ButtonClickedCommand = CommandFactory.Create<string>(
            execute: ButtonAction,
            allowsMultipleExecutions: false,
            onException: ExceptionHelper.LogAndShowAlert
        );
    }

    public Task ButtonAction() 
    {
        // do something, no need to handle exception or re-entry here
    }
}
```

# dotnet MAUI
## Send logs to HTTP endpoint via serialog
```
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.WithExceptionData()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("device",
        new
        {
            Platform = DeviceInfo.Current.Platform.ToString(),
            Model = DeviceInfo.Current.Model,
            Manufacturer = DeviceInfo.Current.Manufacturer,
            Name = DeviceInfo.Current.Name,
            AppVersion = VersionTracking.CurrentVersion,
            AppBuild = VersionTracking.CurrentBuild
        }, true)
    .WriteTo.DurableHttpUsingFileSizeRolledBuffers(
        requestUri: "https://some.server.net/logs/app-name/key",
        bufferBaseFileName: Path.Combine(FileSystem.Current.CacheDirectory, "log_buffer"),
        bufferFileSizeLimitBytes: 10_000_000L,
        period: TimeSpan.FromSeconds(5))
    .CreateLogger();


builder.Services.AddLogging(loggingBuilder =>
    {
        loggingBuilder.ClearProviders();
#if DEBUG
        loggingBuilder.AddDebug();
#endif
        loggingBuilder.AddSerilog();

        loggingBuilder.SetMinimumLevel(LogLevel.Debug);
        loggingBuilder.AddFilter("Microsoft", LogLevel.Warning);
        loggingBuilder.AddFilter("Microsoft.Hosting.Lifetime", LogLevel.Information);
    });
```
# Blazor
## Extension on `IJSRuntime`
```
public static class IJSRuntimeExtensions
{
    /// <summary> Navigate back </summary>
    public static ValueTask NavigateBack(this IJSRuntime jsRuntime)
        => jsRuntime.InvokeVoidAsync("history.back");

    /// <summary> log to browser console </summary>
    public static ValueTask ConsoleLog(this IJSRuntime js, string message)
        => js.InvokeVoidAsync("console.log", message);
}
```
usage
```
@inject IJSRuntime JS
JS.ConsoleLog("Hello");
JS.NavigateBack();
```