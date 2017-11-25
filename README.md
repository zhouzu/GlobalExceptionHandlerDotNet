# Global Exception Handling for ASP.NET Core

[![Build status](https://ci.appveyor.com/api/projects/status/kdbepiak0m6olxw7?svg=true)](https://ci.appveyor.com/project/JoeMighty/globalexceptionhandlerdotnet)

GlobalExceptionHandlerDotNet allows you to configure exception handling as a convention with your ASP.NET Core application pipeline as opposed to explicitly handling them within each controller action. This could be particularly helpful in the following circumstances:

- Reduce boiler plate try-catch logic in your controllers
- Catch and appropriately handle exceptions outside of the ASP.NET Core framework
- You don't want error codes being visible by consuming APIs (return 500 for every exception)

This middleware targets the ASP.NET Core pipeline with an optional dependency on the MVC framework for content negotiation if so desired.

## Installation

GlobalExceptionHandler is [available on NuGet](https://www.nuget.org/packages/GlobalExceptionHandler/) and can be installed via the below commands depending on your platform:

```
$ Install-Package GlobalExceptionHandler
```
or via the .NET Core CLI:

```
$ dotnet add package GlobalExceptionHandler
```

## Basic Setup

```csharp
// Startup.cs

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseExceptionHandler().WithConventions(x => {
        x.ContentType = "application/json";
        x.MessageFormatter(s => JsonConvert.SerializeObject(new
        {
            Message = "An error occured whilst processing your request"
        }));
    });
    
    app.Map("/error", x => x.Run(y => throw new Exception()));
}
```

Any exception thrown by your application will result in the follow response:

```http
HTTP/1.1 500 Internal Server Error
Date: Fri, 24 Nov 2017 09:17:05 GMT
Content-Type: application/json
Server: Kestrel
Cache-Control: no-cache
Pragma: no-cache
Transfer-Encoding: chunked
Expires: -1

{
  "Message": "An error occured whilst processing your request"
}
```

## Handling specific exceptions

You can handle specific exceptions explicitly like so:

```csharp

app.UseWebApiGlobalExceptionHandler(x =>
{
    x.ForException<PageNotFoundException>().ReturnStatusCode(HttpStatusCode.NotFound);
    x.MessageFormatter(exception => JsonConvert.SerializeObject(new
    {
        error = new
        {
            exception = exception.GetType().Name,
            message = exception.Message
        }
    }));
});
```

Alternatively you can set the formatter to be unique per exception registered. This will override the root `x.MessageFormatter` referenced above.

```csharp
app.UseWebApiGlobalExceptionHandler(x =>
{
    x.ForException<ArgumentException>().ReturnStatusCode(HttpStatusCode.BadRequest).UsingMessageFormatter(
        exception => JsonConvert.SerializeObject(new
        {
            error = new
            {
                message = "Oops, something went wrong"
            }
        }));
    x.MessageFormatter(exception => "This formatter will be overridden when an ArgumentException is thrown");
});
```

## Configuration Options:

- `OnError(Func<Exception, HttpContext, Task>)`  
Logging endpoint allowing you to capture exception information.
```csharp
// Example
x.OnError((exception, httpContext) =>
{
    _logger.Error(exception.Message);
    return Task.CompletedTask;
});
```

- `ContentType`  
Specify the returned content type (default is `application/json)`.

- `MessageFormatter(Func<Exception, string>)`  
Set a default message formatter that any unhandled exception will trigger.

```csharp
x.MessageFormatter((exception) => {
    return "Oops, something went wrong! Check the logs for more information.";
});
```

- `DebugMode`
Enabling debug mode will cause GlobalExceptionHandlerDotNet to return the full exception thrown. **This is disabled by default and should not be set in production.**

## Content Negotiation

Because GlobalExceptionHandlerDotNet plugs into the .NET Core pipeline, it can also take advantage of content negotiation. This means that if a user requests a resource and sets the `Accept` header to `text/xml`, if an exception occurs then the content type will be formatted to the requested format type.

To enable content negotiation against ASP.NET Core MVC you will need to include [GlobalExceptionHandler.ContentNegotiation.Mvc](#)
