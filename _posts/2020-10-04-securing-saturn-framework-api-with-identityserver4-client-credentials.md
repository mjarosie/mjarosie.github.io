---
categories: ["Dev"]
tags: ["Saturn", "IdentityServer4", "F#", ".NET"]
date:  2020-10-04
redirect_from:
  - /dev/2020/09/24/securing-saturn-framework-api-with-identityserver4-client-credentials.html
layout: post
title:  "Securing an API written in Saturn Framework with Client Credentials Flow and IdentityServer4"
---

If you're trying to figure out how to secure your API written in [Saturn Framework](https://saturnframework.org/) with [Client Credentials Flow](https://auth0.com/docs/flows/client-credentials-flow) - this post is for you.

I asume you've got at least a basic familiarity with terms such as:
- (web) API
- oAuth
- identity provider
- [App startup in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup?view=aspnetcore-3.1) (how the `Startup.cs` file usually looks like and what is it used for)
- [Saturn Framework itself](https://saturnframework.org/explanations/overview.html) (you can read more about its philosophy [here](http://kcieslak.io/Reinventing-MVC-for-web-programming-with-F))

You've also probably digged through the [IdentityServer4 quickstart tutorial](https://identityserver4.readthedocs.io/en/latest/quickstarts/1_client_credentials.html) which this post is based on.

# Client Credentials oAuth flow

Say you want to create a web API which will be called by some other application running without any input from the end-user (perhaps your backend services calling one another).

Taking a look at the Client Credentials Flow diagram - it's the case of one service (client) calling the other (API), but first authenticating with the identity provider of your choice (IdentityServer in this case):

![Client Credentials Flow sequence diagram](/assets/2020-10-04-securing-saturn-framework-api-with-identityserver4-client-credentials/client-credentials-flow-sequence-diagram.png)

## JWT Authentication setup

Let's first look at how the configuration of the API would look like [if we've written it in C#](https://identityserver4.readthedocs.io/en/latest/quickstarts/1_client_credentials.html#configuration):

```C#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();

        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options =>
            {
                options.Authority = "https://localhost:5001";

                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateAudience = false
                };
            });
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseRouting();

        app.UseAuthentication();
        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```
... where `JwtBearerDefaults.AuthenticationScheme` is equal to `"Bearer"`. The bits that we're interested in are:

```C#
public void Configure(IApplicationBuilder app)
{
    // ...
    app.UseAuthentication();
    // ...
}
```

and

```C#
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options =>
        {
            options.Authority = "https://localhost:5001";

            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateAudience = false
            };
        });
    // ...
}
```

We want to replicate the above while using Saturn's [application](https://saturnframework.org/reference/Saturn/saturn-application-applicationbuilder.html) Computation Expression. Luckily it does have a custom operation called `use_jwt_authentication_with_config` which we can use to configure the application's `IServiceCollection services` and `IApplicationBuilder app` as shown above. Let's look at the (current as of 04.10.2020) [implementation of `use_jwt_authentication_with_config` operation](https://github.com/SaturnFramework/Saturn/blob/4b11a4685e5e27f1df6d6195c1d3e965adfb0ec1/src/Saturn/Application.fs#L384) to better understand it:

```F#
///Enables JWT authentication with custom configuration
[<CustomOperation("use_jwt_authentication_with_config")>]
member __.UseJWTAuthConfig(state: ApplicationState, (config : JwtBearerOptions -> unit)) =
  let middleware (app : IApplicationBuilder) =
    app.UseAuthentication()

  let service (s : IServiceCollection) =
    s.AddAuthentication(fun cfg ->
      cfg.DefaultScheme <- JwtBearerDefaults.AuthenticationScheme
      cfg.DefaultChallengeScheme <- JwtBearerDefaults.AuthenticationScheme)
      .AddJwtBearer(Action<JwtBearerOptions> config) |> ignore
    s

  { state with
      ServicesConfig = service::state.ServicesConfig
      AppConfigs = middleware::state.AppConfigs
  }
```

Application Computation Expression works like a builder pattern, where you're modifying its internal `state: ApplicationState` until you're ready to build and actually use the app. The function above adds the `UseAuthentication()` middleware and adds the authentication service to the DI container (`AddAuthentication(...)`). How would we actually invoke this operation so that we can enable JWT authentication?

`use_jwt_authentication_with_config` expects us to provide a function of type `JwtBearerOptions -> unit`. Looking at it, we can conclude that this function has to:
- take already existing `JwtBearerOptions`
- somehow modify these options in place
- return a `unit`

Here's an example of how you'd use this operation:

```F#
open Saturn
open Microsoft.IdentityModel.Tokens
open Microsoft.AspNetCore.Authentication.JwtBearer

// ...

let app = application {
    // ...
    use_jwt_authentication_with_config (fun (opt: JwtBearerOptions) ->
        opt.Authority <- "https://localhost:5001"

        let tvp = TokenValidationParameters()
        tvp.ValidateAudience <- false

        opt.TokenValidationParameters <- tvp
     )
    // ...
}
```

## Authorization at the API

We also need to specify which endpoints require authorization for being accessed. In C#, as the original tutorial says, we should first define the policy in the `ConfigureServices` method in API's `Startup` class:

```C#
services.AddAuthorization(options =>
{
    options.AddPolicy("ApiScope", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("scope", "api1");
    });
});
```

... so that we can then enforce this policy at various levels, for instance at the controller level by using the [AuthorizeAttribute](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authorization.authorizeattribute?view=aspnetcore-3.1):

```C#
[Authorize(Policy = "ApiScope")]
public class MyController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return new JsonResult("Hello World!");
    }
}
```

The first bit (adding Authorization Policy to services) we can solve by using the `use_policy` operation. Unfortunately (or perhaps I just couldn't find an option) Saturn doesn't let you use the [AuthorizationPolicyBuilder](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authorization.authorizationpolicybuilder?view=aspnetcore-3.1) as shown above out of the box, so you have to work around it and inspect the [AuthorizationHandlerContext](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authorization.authorizationhandlercontext?view=aspnetcore-3.1) directly as described in [Policy-based authorization in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-3.1#use-a-func-to-fulfill-a-policy):

```F#
let app = application {
    // ...
    use_policy "ApiScope" (fun context ->
        context.User <> null && 
        context.User.HasClaim(fun claim ->
            (claim.Type = "scope") &&
            (claim.Value = "api1") && 
            (claim.Issuer = "https://localhost:5001"))
    )
    // ...
}
```

When it comes to enforcing the policy, we can use Giraffe's [Authentication and Authorization functions](https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md#authentication-and-authorization), in this specific case - [`authorizeByPolicyName`](https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md#authorizebypolicyname) that we plug in into the [router](https://saturnframework.org/explanations/routing.html)'s [pipeline](https://saturnframework.org/explanations/pipeline.html). The content of an example `HelloController.fs` could be:

```F#
namespace Hello

open Saturn
open Giraffe.ResponseWriters
open Giraffe

module Controller =

    let authFailedHandler: HttpHandler =
        // You'd probably want to respond with more structured message, perhaps JSON.
        setStatusCode 401 >=> text "Unauthorized"
    
    let mustHaveApiScope: HttpHandler =
        // The `authorizeByPolicyName` function takes two parameters - the name of the policy and
        // the handler which should be invoked when the policy check fails.
        authorizeByPolicyName "ApiScope" authFailedHandler 

    let helloProtectedPipeline: HttpHandler = pipeline {
        plug mustHaveApiScope
    }
    
    // Private handler, you're supposed to be authenticated and authorized to get the response.
    let indexProtectedAction (name: string): HttpHandler = 
        json ("Hello, " + name + "!")
    
    // Public handler, you can access it without being authenticated.
    let indexAction: HttpHandler = 
        json "Hello!"

    let helloProtectedRouter: HttpHandler = router {
        pipe_through helloProtectedPipeline
        
        getf "/%s" indexProtectedAction
    }

    let helloRouter: HttpHandler = router {
        get "/" indexAction // Requests to /hello/ go through the "public" handler.
        forward "" helloProtectedRouter // All other requests (/hello/*) go through the "protected" handler.
    }
```

Then in `Router.fs` we could route incoming requests to our `Hello.Controller.helloRouter` router function:

```F#
open Saturn

let apiRouter = router {
    forward "/hello" Hello.Controller.helloRouter
}

let appRouter = router {
    forward "/api" apiRouter
}
```

This way we achieve the following:

- `/api/hello/` resource can be accessed by anyone
- `/api/hello/*` resources can be accessed only by authenticated clients with `api1` scope in their JWT token.

You'd obviously also need to configure the IdentityServer by defining a client that can access this API. For explanation I'm again redirecting you to the IdentityServer's official quickstart tutorial mentioned above.

## Testing the API

We can then check that the authorization works by running the API and IdentityServer instances and invoking some cURL requests:

```powershell
# Send a request to the public, unprotected endpoint:
curl --request GET --url 'https://localhost:8085/api/hello/'

# Send a request to the protected endpoint without attaching the access token (in Authorization header).
# You should get 401 Unauthorized:
curl --request GET --url 'https://localhost:8085/api/hello/test'

# Retrieve an access token and store it into a variable:
$access_token = curl --request POST --url 'https://localhost:5001/connect/token' `
--header 'content-type: application/x-www-form-urlencoded' `
--data 'grant_type=client_credentials&client_id=client&client_secret=secret&scope=api1' `
| jq .access_token

# Send a request to the protected endpoint with the access token attached:
curl --request GET --url 'https://localhost:8085/api/hello/test' `
--header "Authorization: Bearer $access_token"
```

Excercise for the reader: add a new scope (say `api2`) in IdentityServer configuration, allow the client to retrieve tokens against it, and try getting a token for this new scope only. Then - call the API with it. You should receive `401 Unauthorized` response, as the API allows only tokens with `api1` scope!

The project with the runnable code can be found [here](https://github.com/mjarosie/SaturnWithIdentityServerClientCredentials).