---
title: Once More with Handlers
layout: post
published: true
---

I previously wrote about [creating a custom authentication middleware in ASP .NET Core][old]. With the move to ASP .NET Core 2.0 this approach at creating custom authentication providers no longer works. It's time drag custom authentication into the Year of the Fruit Bat with an ASP .NET Core 2.0 `AuthenticationHandler`.

Where previously we registered a custom middleware when configuring our app we instead just need to register the builtin  authentication with `app.UseAuthentication()`. The configuration of our custom auth provider takes place when configuring services instead:

```csharp
public void ConfigureServices(
    IServiceCollection services)
{
    services
        .AddAuthentication(o =>
        {
            o.DefaultAuthenticateScheme =
                TokenAuthOptions.DefaultScheme;
            o.DefaultChallengeScheme =
                TokenAuthOptions.DefaultScheme;
        })
        .AddSimpleTokenAuth(o => {
            // configure our auth options here
        });
}
```

After calling `AddAuthentication` on the service collection you receive an `AuthenticationBuilder` object. For our custom auth provider we need to create an extension method for it to register our handler and options:

```csharp
public static AuthenticationBuilder AddSimpleTokenAuth(this AuthenticationBuilder builder, Action<TokenAuthOptions> options)
{
    return builder
        .AddScheme<TokenAuthOptions,
                   SimpleTokenAuthHandler>(
             TokenAuthOptions.DefaultScheme, options);
}
```

For this to work we need two new classes. The first replaces our old middleware class, this time deriving from `AuthenticationHandler`. It contains the logic for authenticating users based on the current request. I've chosen to call it `SimpleTokenAuthHandler`. The second is the `TokenAuthOptions` class. It's job is to provide additional configuration for custom authentication schemes. The `AddScheme` call ties these two together under the named scheme `TokenAuthOptions.DefaultScheme`.

First up let's set up the `TOkenAuthOptions` class:

```csharp
public class TokenAuthOptions : AuthenticationSchemeOptions
{
    public const string DefaultScheme =
        nameof(SimpleTokenAuthHandler);

    public string Scheme => DefaultScheme;
}
```

If your authentication scheme request additional data to be configured then this is the place to put it. Users can then update the settings in this options instance from the `AddSimpleTokenAuth` call in `ConfigureServices`. Following on from the previous middleware example however we won't be taking any extra options either.

The last part of the puzzle is the `SimpleTokenAuthHandler` class. This inherits from the base `AuthenticationHandler` provided by ASP .NET. We override the `HandleAuthenticationAsync` method and perform our custom authentication check. There are three main results that you might want to return from this method: [`AuthenticateResult.Success`] when you have been able validate the user correctly, [`AuthenticateResult.Fail`] when there was a problem with the authentication details provided, and [`AuthenticateResult.NoResult`] when there wasn't any authentication attempt made for your custom scheme.

```csharp
protected override Task<AuthenticateResult> HandleAuthenticateAsync()
{
    if (Request.Query.ContainsKey("auth"))
    {
        var identity = new GenericIdentity(
            "test_user", "queryString");
        var ticket = new AuthenticationTicket(
            new ClaimsPrincipal(identity),
            Options.Scheme);

        return Task.FromResult(
            AuthenticateResult.Success(ticket));
    }

    return Task.FromResult(
        AuthenticateResult.NoResult());
}
```

With this all in pace we can make and authenticated and unauthenticated request:

```http
GET /?auth HTTP/1.1
Host: localhost:5000
Accept: */*

HTTP/1.1 200 OK
Server: Kestrel

Identity
  Name: test_user
  Label: 
  AuthType: queryString
  Authenticated?: True
  Claims: test_user
```

```http
GET / HTTP/1.1
Host: localhost:5000
Accept: */*

HTTP/1.1 200 OK
Server: Kestrel

Identity
  Name: 
  Label: 
  AuthType: 
  Authenticated?: False
  Claims: 
```

This is a little bit different from the identities we ended up with in the old ASP .NET Core 1 version. Where previously we added a second identity which was authenticated the use of the standard `AuthenticationHandler` has meant we end up with just a single identity, which is either authenticated or unauthenticated.

The full code for this example is available [as an update to the  Gist][new-gist]. Happy authenticating!

 [old]: /2017/08/17/authentication-middleware-in-asp-net-core.html
 [new-gist]: https://gist.github.com/iwillspeak/1e2d78f36c89a898891148c47befdf4b
 [`AuthenticateResult.Success`]: https://docs.microsoft.com/en-gb/dotnet/api/microsoft.aspnetcore.authentication.authenticateresult.success?view=aspnetcore-2.0#Microsoft_AspNetCore_Authentication_AuthenticateResult_Success_Microsoft_AspNetCore_Authentication_AuthenticationTicket_
 [`AuthenticateResult.Fail`]: https://docs.microsoft.com/en-gb/dotnet/api/microsoft.aspnetcore.authentication.authenticateresult.fail?view=aspnetcore-2.0#Microsoft_AspNetCore_Authentication_AuthenticateResult_Fail_System_String_
 [`AuthenticateResult.NoResult`]: https://docs.microsoft.com/en-gb/dotnet/api/microsoft.aspnetcore.authentication.authenticateresult.noresult?view=aspnetcore-2.0#Microsoft_AspNetCore_Authentication_AuthenticateResult_NoResult