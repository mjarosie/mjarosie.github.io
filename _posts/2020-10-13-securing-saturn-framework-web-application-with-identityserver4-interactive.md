---
categories: ["Dev"]
tags: ["Saturn", "IdentityServer4", "F#", ".NET"]
date:  2020-10-13
layout: post
title:  "Securing a web application written in Saturn Framework with Authorization Code Flow and IdentityServer4"
---

Comparing to the Client Credentials Flow which I described [in my previous post](/dev/2020/10/04/securing-saturn-framework-api-with-identityserver4-client-credentials.html) - the [Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) involves one more entity - the End-User (aka Resource Owner). That makes the whole process "interactive", since the End-User needs to take an action - log in and allow our application (the Client) to have access to a Protected Resource (for instance - retrieving user's email, setting, or some user-specific value on behalf of the user, stored behind the API that is protected by the Identity Provider). David Neal explained this flow realy well in [his post](https://developer.okta.com/blog/2019/10/21/illustrated-guide-to-oauth-and-oidc). I'm assuming you're familiar with the [C# equivalent setup of the IdentityServer4 for Interactive Applications with ASP.NET Core](https://identityserver4.readthedocs.io/en/latest/quickstarts/2_interactive_aspnetcore.html) - if not, give it a go, it's explained really well and should give you a good understanding of how things work.

Just to refresh - here's the Authorization Code Flow sequence diagram:

![Authorization Code Flow sequence diagram](/assets/2020-10-13-securing-saturn-framework-web-application-with-identityserver4-interactive/authorization-code-flow-sequence-diagram.png)

Krzysztof Cieślak [has explained how to use the GitHub provider in Saturn](https://medium.com/lambda-factory/using-oauth-with-saturn-c8eba5d63e1c), but even though the setup is really similar, it still took me a while to make it work with IdentityServer4 properly, I've also struggled with logging out which I'll elaborate on below.

You can follow along by cloning the source code from [this repository](https://github.com/mjarosie/SaturnWithIdentityServerInteractive).

# Solution setup

The solution involves two services: the web application itself, and the Identity Provider (IdentityServer4 in this specific case).

For the former - you can either use [the template](https://saturnframework.org/tutorials/how-to-start.html) (simpler option):

```powershell
dotnet new -i Saturn.Template # Run it only once - when you've never used the Saturn template before.
mkdir SaturnWithIdentityServerInteractive && cd SaturnWithIdentityServerInteractive
dotnet new saturn -lang F#
dotnet tool restore
```

... or set it up manually (if you know what you're doing). I've used the template, but removed all the irrelevant code which would obscure the main topic of this post (error handling, database migration etc).

Setting up IdentityServer4 is really simple, you just need to create the project out of the `is4inmem` template (run these commands from the root directory of the project, `SaturnWithIdentityServerInteractive` in my case):

```powershell
cd src
dotnet new -i IdentityServer4.Templates # Run it only once - when you've never used IdentityServer4 templates before.
dotnet new is4inmem -n IdentityServer
cd ..
dotnet sln add .\src\IdentityServer\IdentityServer.csproj
```

Then in `Config.cs` set up the `Clients` variable to:

{% highlight C# %}
public static IEnumerable<Client> Clients =>
    new Client[]
    {
        // interactive client using code flow + pkce
        new Client
        {
            ClientId = "interactive",
            ClientSecrets = { new Secret("secret".Sha256()) },
            
            AllowedGrantTypes = GrantTypes.Code,

            RedirectUris = { "https://localhost:8085/signin-oidc" },
            FrontChannelLogoutUri = "https://localhost:8085/signout-oidc",
            PostLogoutRedirectUris = { "https://localhost:8085/signout-callback-oidc" },

            AllowOfflineAccess = true,
            AllowedScopes = { "openid", "profile", "scope2" }
        },
    };
{% endhighlight %}

This should give you a ready-to-run instance with two logins: `alice` (password: `alice`) and `bob` (password: `bob`) and a set of client credentials (client_id: `interactive`, client_secret: `secret`). Sweet!

# Saturn application settings

To use the OpenID Authentication we need to set up the services to use Authentication, Cookie and OpenIdConnect handlers. In C# [we'd do it like this](https://identityserver4.readthedocs.io/en/latest/quickstarts/2_interactive_aspnetcore.html#creating-an-mvc-client):

{% highlight C# %}
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.AspNetCore.Authentication.Cookies;
using System.IdentityModel.Tokens.Jwt;

// ...

public void ConfigureServices(IServiceCollection services)
{
    // ...

    JwtSecurityTokenHandler.DefaultMapInboundClaims = false;

    services.AddAuthentication(options =>
    {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme; // Equals to "Cookies"
        options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme; // Equals to "OpenIdConnect";
    })
    .AddCookie(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, options =>
    {
        options.Authority = "https://localhost:5001";

        options.ClientId = "mvc";
        options.ClientSecret = "secret";
        options.ResponseType = "code";

        options.SaveTokens = true;
    });

    // ...
}

// ...

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ...
    app.UseAuthentication();
    // ...
}
{% endhighlight %}

Luckily - [Saturn.Extensions.Authorization](https://saturnframework.org/reference/Saturn.Extensions.Authorization/global-saturn.html) package contains an [Application Computation Expression](https://saturnframework.org/reference/Saturn/saturn-application-applicationbuilder.html) Custom Operation: `use_open_id_auth_with_config` which helps us to achieve exactly this. In order to be able to use it - you need to add `nuget Saturn.Extensions.Authorization` to `paket.dependencies` and `Saturn.Extensions.Authorization` to `src\SaturnWithIdentityServerInteractive\paket.references` files. Then run `dotnet paket install` and `dotnet restore`.

Looking at [the implementation](https://github.com/SaturnFramework/Saturn/blob/e2151f8839951baa5ea5616ea6002579cf4cb506/src/Saturn.Extensions.Authorization/OAuth.fs#L189) (as of 13.10.2020) we can see that the code is semantically similar to the C# equivalent mentioned above:

{% highlight FSharp %}
/// Enables OpenId authentication with custom configuration
[<CustomOperation("use_open_id_auth_with_config")>]
member __.UseOpenIdAuthWithConfig(state: ApplicationState, (config: Action<OpenIdConnect.OpenIdConnectOptions>)) =
    let middleware (app : IApplicationBuilder) =
        app.UseAuthentication()

    let service (s: IServiceCollection) =
        let authBuilder = s.AddAuthentication(fun authConfig ->
            authConfig.DefaultScheme <- CookieAuthenticationDefaults.AuthenticationScheme
            authConfig.DefaultChallengeScheme <- OpenIdConnectDefaults.AuthenticationScheme
            authConfig.DefaultSignInScheme <- CookieAuthenticationDefaults.AuthenticationScheme)
        addCookie state authBuilder
        authBuilder.AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, config) |> ignore

        s

    { state with
        ServicesConfig = service::state.ServicesConfig
        AppConfigs = middleware::state.AppConfigs
        CookiesAlreadyAdded = true }
{% endhighlight %}

If you're not sure why exactly do we specify different Schemes in the `authConfig` (as asked [here](https://stackoverflow.com/q/52492666)) - see [this brilliant answer](https://stackoverflow.com/a/52493428).

We should use the `use_open_id_auth_with_config` operation like so:

{% highlight FSharp %}
open Saturn
open Microsoft.AspNetCore.Authentication
open Microsoft.AspNetCore.Authentication.Cookies

let app = application {
    // ...
    use_open_id_auth_with_config (fun (options: OpenIdConnect.OpenIdConnectOptions) ->
        options.Authority <- "https://localhost:5001";

        options.ClientId <- "interactive";
        options.ClientSecret <- "secret";
        options.ResponseType <- "code";

        options.SaveTokens <- true;
        
        // Say our Client wants to call the API which requires `scope2` scope.
        options.Scope.Add("scope2")

        // Following lines are added so that our app can have access to the user's profile data, such as name, website etc.
        // See: https://identityserver4.readthedocs.io/en/latest/quickstarts/2_interactive_aspnetcore.html#getting-claims-from-the-userinfo-endpoint
        options.Scope.Add("profile")
        options.GetClaimsFromUserInfoEndpoint <- true;
    )
    // ...
}
{% endhighlight %}

How can we leverage this protection now? Let's say our application wants to access a protected secret of the user - their favourite colour. Our `Router.fs` would look like this:

{% highlight FSharp %}
let protectedView = router {
    pipe_through ProtectedHandlers.protectedViewPipeline
    
    get "/" ProtectedHandlers.protectedHandler
}

let defaultView = router {
    get "/" (htmlView Index.layout)
    get "/index.html" (redirectTo false "/")
    get "/default.html" (redirectTo false "/")
}

let browser = pipeline {
    plug acceptHtml
    plug putSecureBrowserHeaders
}

let browserRouter = router {
    pipe_through browser

    forward "" defaultView
    forward "/protected" protectedView
}

let appRouter = router {
    forward "" browserRouter
}
{% endhighlight %}

We've got the `defaultView` - the home page which can be accessed by anyone, and the `protectedView` (under `/protected`). It can be accessed only by authenticated users, and is set up like this (in `ProtectedHandlers.fs`):

{% highlight FSharp %}
module ProtectedHandlers

open Giraffe
open Saturn
open Microsoft.AspNetCore.Authentication
open Microsoft.AspNetCore.Authentication.OpenIdConnect
open FSharp.Control.Tasks.V2.ContextInsensitive
open System.Threading.Tasks

let protectedViewPipeline = pipeline {
    requires_authentication (Giraffe.Auth.challenge OpenIdConnectDefaults.AuthenticationScheme)
}

let protectedHandler : HttpHandler = fun next ctx ->
    task {
        let getUserName (): string =
            ctx.User.Claims 
            |> Seq.pick (fun claim -> if claim.Type = "given_name" then Some claim else None)
            |> (fun claim -> claim.Value)
            
        let getUserSecret (userName: string): Task<string option> =
            task {
                let! accessToken = ctx.GetTokenAsync("access_token")
                // Call an API with the accessToken...
                // ...

                // Mock:
                match userName with
                | "Alice" -> return Some "Red"
                | "Bob" ->  return Some "Blue"
                | _ ->  return None
            }

        let userNameId = getUserName()
        match! getUserSecret(userNameId) with
        | Some secret -> return! htmlView (Protected.protectedResourceView userNameId secret) next ctx
        | None -> return! htmlView (Protected.noProtectedResourceView userNameId) next ctx
    }
{% endhighlight %}

`protectedViewPipeline` uses the Giraffe's [`requiresAuthentication`](https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md#requiresauthentication) together with [`challenge`](https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md#challenge) - which uses the `OpenIdConnectDefaults.AuthenticationScheme` scheme (being equal to `"OpenIdConnect"` as mentioned above). __Pay special attention to this value__ - we don't want to be using `CookieAuthenticationDefaults.AuthenticationScheme` as this is a different scheme which doesn't handle the OpenIdConnect protocol. It will not redirect the user to the Identity Provider (`https://localhost:5001/account/login`), but to the application itself (`https://localhost:8085/account/login`) which will result in `404` error if you don't have the `account/login` endpoint. That's a no-no!

`protectedHandler` retrieves user's details (`given_name`) from the token claims (remember to add the `profile` scope to `OpenIdConnectOptions` in the application setup!), retrieves user's secret colour by calling the API with the Client Credentials Flow and provides these values to the view which returns a personalised "secret" webpage. If the user doesn't have the secret colour yet, one is informed about it as well.

# Logging the user out

There's one more bit which took me a while to figure out. Giraffe provides a [`signOut`](https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md#signout) handler which signs the user out from a given authentication scheme. The documentation gives an example of how to use it:

{% highlight FSharp %}
let logout = signOut "Cookie" >=> redirectTo false "/"

let webApp =
    choose [
        route "/"     >=> text "Hello World"
        route "/user" >=>
            requiresAuthentication (challenge "Cookie") >=>
                choose [
                    GET  >=> readUserHandler
                    POST >=> submitUserHandler
                    route "/user/logout" >=> logout
                ]
    ]
{% endhighlight %}

Since we want to sign the user out of both `"Cookies"` and `"OpenIdConnect"` schemes - you'd probably try doing something similar in our Saturn router:

{% highlight FSharp %}
let defaultView = router {
    // ...
    get "/logout" (Giraffe.Auth.signOut CookieAuthenticationDefaults.AuthenticationScheme // Sign out of "Cookies" scheme
                >=> Giraffe.Auth.signOut OpenIdConnectDefaults.AuthenticationScheme // Sign out of "OpenIdConnect" scheme
                >=> redirectTo false "/") // Redirect to the home page.
    // ...
}

let browserRouter = router {
    // ...
    forward "" defaultView
    // ...
}

let appRouter = router {
    forward "" browserRouter
}
{% endhighlight %}

If you log in as `bob`, go to `/logout` page - you'll be logged out indeed - but the next time you'd like to log in - the IdentityServer session will still be active - so instead of being taken to the login screen (where now you'd like to log in as `alice`) - you'll be automatically redirected back to your app logged in as `bob`. [This StackOverflow question](https://stackoverflow.com/questions/47489966/) describes exactly this scenario. [It turns out](https://stackoverflow.com/a/51613121) - we need to use overloaded `SignOutAsync` function which accepts `AuthenticationParameters` but Giraffe doesn't offer it out of the box. Hence, we need to write our own wrapper for calling this function (the implementation is based on the [`signOut` function](https://github.com/giraffe-fsharp/Giraffe/blob/9f1ea96b796e6ecb41e862eb7c5d74776a19326a/src/Giraffe/Auth.fs#L32)):

{% highlight FSharp %}
let signOutAndRedirect (authScheme : string) (redirectUrl: string) : HttpHandler =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let props = AuthenticationProperties()
            props.RedirectUri <- redirectUrl
            do! ctx.SignOutAsync (authScheme, props)
            return! next ctx
        }

let defaultView = router {
    // ...
    get "/logout" (Giraffe.Auth.signOut CookieAuthenticationDefaults.AuthenticationScheme
                    >=> signOutAndRedirect OpenIdConnectDefaults.AuthenticationScheme "index.html") // After logout - the Identity Provider will redirect the user to this page.
    // ...
}

let browserRouter = router {
    // ...
    forward "" defaultView
    // ...
}

let appRouter = router {
    forward "" browserRouter
}
{% endhighlight %}

Let's try it again and it... works! Logging it as `bob`, then logging out and logging back in again - we'll be taken to the login page where we can log in as `alice` now.

That should be it! If you've got any questions, feel free to post one below or reach out [through Twitter](https://twitter.com/mjarosie).