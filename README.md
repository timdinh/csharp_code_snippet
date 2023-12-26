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
// list (prior or dotnet 8)
int[] numbers = { 1, 2, 3, 4, 5 };
// list (from dotnet 8)
int[] numbers = [ 1, 2, 3, 4, 5];
```
# Raw String Literals
```
var letter = $"""
              Dear {Name},
             
              This letter is to inform …
              """;
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

## prior to dotnet 8 via middleware
```
public class GlobalExceptionHandlerMiddleware
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
            await _next(context);
        }
        catch (Exception ex)
        {
            // Handle exceptions here
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        logger.LogError(exception, "Unhandled error");
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsync("{ code = 123, error = \"server error\" }");
    }
}
```

## prior to dotnet 8 
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

# Blazor
