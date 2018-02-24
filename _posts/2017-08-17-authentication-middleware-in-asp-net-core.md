---
title: Authentication Middleware in ASP.NET Core
layout: post
published: true
---

There are two sides to the auth story in ASP .NET Core: authentication and authorisation. Authorisation is usually controlled in an MVC app with the `[Authorize]` attribute. The simplest, no-argument `[Authorize]` checks that the user has a single identity which is authenticated. Authentication is responsible for inspecting the request and adding known identity information about the user. In many cases this will be validating OAuth or JWT.

Supposing you have something simpler though. An app which doesn't need to issue tokens of its own; instead it just needs to validate tokens issued by another app. In that case it's time to create your own custom authentication middleware.

First we'll add an `app.UseSimpleTokenAuth()` to our application build chain. This needs to come before any parts of the pipeline which depend on authentication. Obviously that extension method doesn't exist yet. The next step is to create it:

{% highlight C# %}
public static class MiddlewareExtensions
{
    public static IApplicationBuilder
    UseSimpleTokenAuth(
        this IApplicationBuilder app)
    {
        return app
          .UseMiddleware<SimpleTokenAuthMiddleware>();
    }
}
{% endhighlight %}

This method registers everything needed by our middleware. At the moment that's just the middleware itself. If we want to pass configuration information to the middleware through dependency injection at a later date this would be the location to add the `.Configure<>(..)` calls.

Once we have our middleware registered it's time to write it. A simple middleware just receives a `RequestDelegate` in its constructor, and calls that delegate when `Invoke` is called:

{% highlight C# %}
public class SimpleTokenAuthMiddleware
{
    private readonly RequestDelegate _next;

    public SimpleTokenAuthMiddleware(
        RequestDelegate next)
    {
        _next = next;
    }

    public Task Invoke(HttpContext context)
    {
        // This is where we will check auth later
        return _next(context);
    }
}
{% endhighlight %}

Now comes the difficult bit. Inspecting the request to see if the user is authenticated. In this simple case I'm just going to check for the presence of an `auth` query parameter. In reality though you'd want to look for a token in headers or cookies and validate it somehow.

{% highlight C# %}
if (context.Request.Query.ContainsKey("auth"))
{
    var identity = new GenericIdentity(
        "test_user", "queryString");

    context.User.AddIdentity(identity);
}
{% endhighlight %}

If the user is authenticated we create a new [`GenericIdentity`](https://msdn.microsoft.com/en-us/library/system.security.principal.genericidentity(v=vs.110).aspx) for the user. The first parameter specifies the user's name and the second the authentication method used to generate the identity. You must include some value for the authentication method otherwise the identity can't be used with the `[Authorize]` attribute.

That's it. If a request has `auth` in the query string an identity will be added and the user will be considered authenticated. To verify this we can create a simple `Run` method in our `Startup.cs`:

{% highlight C# %}
app
    .UseSimpleTokenAuth()
    .Run(async context =>
            {
                if (context.User?.Identities == null)
                {
                    await context.Response.WriteAsync(
                    "No user identities");
                }

                foreach (var id in 
                         context.User.Identities)
                {
                    var sb = new StringBuilder();

                    sb.AppendLine("Identity");
                    sb.AppendLine(
                      $"  Name: {id.Name}");
                    sb.AppendLine(
                      $"  Label: {id.Label}");
                    sb.AppendLine(
                      $"  AuthType: {id.AuthenticationType}");
                    sb.AppendLine(
                      $"  Authenticated?: {id.IsAuthenticated}");
                    var claims = string.Join(
                        ", ",
                        id.Claims.Select(
                            c => c.Value));
                    sb.AppendLine($"  Claims: {claims}");

                    await context.Response.WriteAsync(
                        sb.ToString());
                }
            });

{% endhighlight %}

Now if you hit your app you should see a blank anonymous identity:

{% highlight http %}
GET / HTTP/1.1

HTTP/1.1 200 OK

Identity
  Name: 
  Label: 
  AuthType: 
  Authenticated?: False
  Claims:  
{% endhighlight %}

Hit it with `auth` in the query string and a second, authenticated, identity appears:

{% highlight http %}
GET /?auth HTTP/1.1

HTTP/1.1 200 OK

Identity
  Name: 
  Label: 
  AuthType: 
  Authenticated?: False
  Claims: 
Identity
  Name: test_user
  Label: 
  AuthType: queryString
  Authenticated?: True
  Claims: test_user
{% endhighlight %}

You can find the full code for this post [in a gist](https://gist.github.com/iwillspeak/1e2d78f36c89a898891148c47befdf4b/9df41d2c17bde20c4bb787f3eb28ef901eb2f661).

**EDIT**: For an updated version of this authentication scheme which works with ASP .NET Core 2.0 check out [Once More with Handlers][new-post].

 [new-post]: /2018/02/24/once-more-with-handlers.html