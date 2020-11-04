---
categories: ["Dev"]
tags: ["Fable", "F#", ".NET", "IdentityServer4", "OpenID Connect"]
date:  2020-11-04
layout: post
title:  "Securing Fable application with IdentityServer4"
---

In this post I'd like to show you how to reproduce [the IdentityServer4 tutorial ("Adding a JavaScript client")](https://identityserver4.readthedocs.io/en/latest/quickstarts/4_javascript_client.html) of securing an example Fable application with the Authorization Code Flow and IdentityServer4 by performing the OpenID Connect protocol on the client side. Instead of doing it in JavaScript - we'll do it in F# (using Fable). There are probably more efficient and cleaner ways of implementing this in Fable apps, but my focus was to stay as close to the original tutorial as possible. Treat it as a starter project which is supposed to inspire you to implement something similar in your app : ) Finished project ready to be run is available [here](https://github.com/mjarosie/FableBrowserClientOpenIdConnect).

# Setup

There are 3 entities involved:

- IdentityServer4 (oAuth 2.0 Authorization Server, OpenID Provider)
- Web API (Resource Server)
- Fable Client (Client, aka Relying Party)

I'm assuming that you're familiar with how to set up the IdentityServer4 and secure a Web API with it. If not - take a look at my [previous post explaining it in more detail](/dev/2020/10/04/securing-saturn-framework-api-with-identityserver4-client-credentials.html). We'll communicate with a Web API having exactly the same functionality as the API used in the original tutorial - it should return user's claims when the `/identity` endpoint is called and the call is authenticated. As in the original tutorial - IdentityServer4 should be running on `https://localhost:5001` and the API should be available under `https://localhost:6001`. I assume these projects are put in `src` directory of the root directory of your solution.

## Setting up the Client projects structure

To set up the client - we'll actually require two Fable apps split into two pages: `index.html` and `callback.html`. Run the following to create a directory called `Client` in your solution's `src` directory, and set up two Fable projects: `App` (which we'll import in `index.html`) and `Auth` (which we'll import in `callback.html`):

```Powershell
dotnet new -i "Fable.Template::*" # You only need to invoke this command once - if you've never used the Fable template before.
cd src
mkdir Client
cd  Client
dotnet new fable -n App
dotnet new fable -n Auth
```

We'll also need to set up the `node.js` and `Webpack` configurations. I did it in the root directory of my solution.

First create `package.json`:

```json
{
  "private": true,
  "scripts": {
    "start": "webpack-dev-server"
  },
  "dependencies": {
    "@babel/core": "^7.8.4",
    "fable-compiler": "^2.4.15",
    "fable-loader": "^2.1.8",
    "oidc-client": "^1.10.1",
    "react": "^16.12.0",
    "react-dom": "^16.12.0",
    "webpack": "^4.41.6",
    "webpack-cli": "^3.3.11",
    "webpack-dev-server": "^3.10.3"
  }
}
```

In the same directory create a file called `webpack.config.js` and put in the following content:

```javascript
var path = require("path");

module.exports = {
    mode: "development",
    entry: {
        "app": "./src/Client/App/App.fsproj",
        "auth": "./src/Client/Auth/Auth.fsproj",
    },
    output: {
        path: path.join(__dirname, "./public"),
        filename: "[name].js",
    },
    devServer: {
        publicPath: "/",
        contentBase: "./public",
        port: 5003,
        https: true,
    },
    module: {
        rules: [{
            test: /\.fs(x|proj)?$/,
            use: "fable-loader"
        }]
    }
}
```

Above we've configured the Webpack server to:
- use two separate [entry points](https://webpack.js.org/guides/code-splitting/#entry-points) - `app` (from `App` project) and `auth` (from `Auth` project)
- produce an output from these two separate projects into a `public` directory in root directory of this solution (this will generate `app.js` and `auth.js` respectively)
- use https and serve the content of `public` directory on port `5003`
- use [Webpack loader for Fable](https://www.npmjs.com/package/fable-loader) for F# files.

## index.html

Create `index.html` file in the `public` directory (again, in the root directory of the solution) with the following content:

```html
<!doctype html>
<html>
<head>
  <title>Fable Client Auth Demo</title>
  <meta charset="utf-8" />
</head>
<body>
  <button id="login">Login</button>
  <button id="api">Call API</button>
  <button id="logout">Logout</button>

  <pre id="results"></pre>
  <script src="app.js"></script>
</body>
</html>
```

We've created a page with 3 buttons (`login`, `api` and `logout`) and a preformatted text element (`results`). We've also imported the content of `app.js` script (a file which will be produced by Webpack through Fable loader from our F# code defined in `./src/Client/App` project). Let's take a look at the content of `App.js`:

{% highlight FSharp %}
module App

open Browser.Dom
open Fable.OidcClient
open Fable.Core.JsInterop
open Thoth.Fetch
open Thoth.Json

let log (arguments: string list) =
    let results = document.getElementById("results")
    results.innerText <- "";

    List.map (fun (msg) -> 
        results.innerText <- results.innerText + msg + "\r\n";
    ) arguments |> ignore
    ()

type Claim =
    { Type : string
      Value : string }

let settings: UserManagerSettings = 
    !!{| 
        authority = Some "https://localhost:5001"
        client_id = Some "js"
        redirect_uri = Some "https://localhost:5003/callback.html"
        response_type = Some "code"
        scope = Some "openid profile scope1"
        post_logout_redirect_uri = Some "https://localhost:5003/index.html"
        
        filterProtocolClaims = Some true
        loadUserInfo = Some true
    |}

let mgr: UserManager = Oidc.UserManager.Create settings

let loginButton = document.getElementById("login") :?> Browser.Types.HTMLButtonElement
let apiButton = document.getElementById("api") :?> Browser.Types.HTMLButtonElement
let logoutButton = document.getElementById("logout") :?> Browser.Types.HTMLButtonElement

loginButton.onclick <- fun _ ->
    mgr.signinRedirect()

apiButton.onclick <- fun _ -> promise {
    let! user = mgr.getUser(): Fable.Core.JS.Promise<User option>

    match user with
    | Some u -> 
        let authHeader: Fetch.Types.HttpRequestHeaders = Fetch.Types.HttpRequestHeaders.Authorization ("Bearer " + u.access_token)
        console.log (sprintf "%A" authHeader)
        let! claims = Fetch.get<_, Claim list> (url = "https://localhost:6001/identity", headers = [authHeader], caseStrategy=CamelCase)
        let json = Encode.Auto.toString(4, claims)
        log([json])
    | _ -> log(["Log in first!"])
}
logoutButton.onclick <- fun _ ->
    mgr.signoutRedirect()

promise {
    let! user = mgr.getUser(): Fable.Core.JS.Promise<User option>
    match user with
    | Some u -> 
        let profile = Encode.toString 4 u.profile
        log(["User logged in:\n" + profile])
    | _ -> log(["User not logged in"])
} |> ignore
{% endhighlight %}

Compare the content of the above script to [the original one](https://identityserver4.readthedocs.io/en/latest/quickstarts/4_javascript_client.html#add-your-html-and-javascript-files) written in JavaScript. Let's go through above code step by step.

First we've opened required namespaces:

{% highlight FSharp %}
open Browser.Dom
open Fable.OidcClient
open Fable.Core.JsInterop
open Thoth.Fetch
open Thoth.Json
{% endhighlight %}

To be able to use the above mentioned libraries we need to [reference them first](https://fable.io/docs/your-fable-project/use-a-fable-library.html). Invoke the following commands in `./src/Client/App`:

```Powershell
dotnet add package Fable.Core -v 3.1.6 # Fable itself
dotnet add package Fable.Browser.Dom -v 1.1.0 # Required for manipulating DOM elements
dotnet add package Fable.OidcClient -v 1.0.2 # Required for handling the OpenID Connect protocol
dotnet add package Thoth.Fetch -v 2.0.0 # Required for calling the API
dotnet add package Thoth.Json -v 5.0.0 # Required for stringifying the API response to JSON format
```

Then we've defined a helper `log` function that puts a given list of strings into our page:

{% highlight FSharp %}
let log (arguments: string list) =
    let results = document.getElementById("results")
    results.innerText <- "";

    List.map (fun (msg) -> 
        results.innerText <- results.innerText + msg + "\r\n";
    ) arguments |> ignore
    ()
{% endhighlight %}

Next we've defined the type of a response that we'll be receiving from the API:

{% highlight FSharp %}
type Claim =
    { Type : string
      Value : string }
{% endhighlight %}

It's followed by creating a User Manager instance. We'll use it to invoke the OpenID Connect protocol steps, such as login, logout or retrieving the authenticated user information. Here we specify the client id, address of our IdentityServer4 (or any other OpenID Connect provider), required scopes, and redirect URIs:

{% highlight FSharp %}
let settings: UserManagerSettings = 
    !!{| 
        authority = Some "https://localhost:5001"
        client_id = Some "js"
        redirect_uri = Some "https://localhost:5003/callback.html"
        response_type = Some "code"
        scope = Some "openid profile scope1"
        post_logout_redirect_uri = Some "https://localhost:5003/index.html"
        
        filterProtocolClaims = Some true
        loadUserInfo = Some true
    |}

let mgr: UserManager = Oidc.UserManager.Create settings
{% endhighlight %}

Then we're retrieving references to html tags defined in our `index.html` file:

{% highlight FSharp %}
let loginButton = document.getElementById("login") :?> Browser.Types.HTMLButtonElement
let apiButton = document.getElementById("api") :?> Browser.Types.HTMLButtonElement
let logoutButton = document.getElementById("logout") :?> Browser.Types.HTMLButtonElement
{% endhighlight %}

Later we're assigning a behaviour to the `onClick` event for each of these buttons:

{% highlight FSharp %}
loginButton.onclick <- fun _ ->
    mgr.signinRedirect()

apiButton.onclick <- fun _ -> promise {
    let! user = mgr.getUser(): Fable.Core.JS.Promise<User option>

    match user with
    | Some u -> 
        let authHeader: Fetch.Types.HttpRequestHeaders = Fetch.Types.HttpRequestHeaders.Authorization ("Bearer " + u.access_token)
        console.log (sprintf "%A" authHeader)
        let! claims = Fetch.get<_, Claim list> (url = "https://localhost:6001/identity", headers = [authHeader], caseStrategy=CamelCase)
        let json = Encode.Auto.toString(4, claims)
        log([json])
    | _ -> log(["Log in first!"])
}

logoutButton.onclick <- fun _ ->
    mgr.signoutRedirect()
{% endhighlight %}

Login and logout are quite straightforward - we just call `signinRedirect` and `signoutRedirect`. When clicking on the `api` button - we're first retrieving the user details from the User Manager instance, and then if the App user is authenticated - we're extracting the access token and call the API with the help of [Thoth.Fetch](https://thoth-org.github.io/Thoth.Fetch/) library. It's followed by stringifying the response (using [Thoth.Json](https://thoth-org.github.io/Thoth.Json/)) and logging it onto the screen.

The last bit of code is invoked right after the page is loaded. It retrieves user's details - and depending on whether the user is authenticated or not - logs a corresponding message to the screen:

{% highlight FSharp %}
promise {
    let! user = mgr.getUser(): Fable.Core.JS.Promise<User option>
    match user with
    | Some u -> 
        let profile = Encode.toString 4 u.profile
        log(["User logged in:\n" + profile])
    | _ -> log(["User not logged in"])
} |> ignore
{% endhighlight %}

## callback.html

Let's set up the page responsible for receiving the OpenID connect protocol callback from the IdentityServer4 after the end-user logged in (and perhaps gave a consent for the client application to access user's resources - or maybe not):

```html
<!doctype html>
<html>
<head>
  <title>Fable Client Auth Demo (Callback)</title>
  <meta charset="utf-8" />
</head>
<body>
    <script src="auth.js"></script>
</body>
</html>
```

This file just loads the content of `auth.js` which will be produced from this F# code (in `./src/Client/Auth` project):

{% highlight FSharp %}
module Auth

open Fable.OidcClient
open Fable.Core.JsInterop
open Browser

let mgr: UserManager = Oidc.UserManager.Create !!{| response_mode = Some "query" |}

promise {
    let! user = mgr.signinRedirectCallback()
    window.location.href <- "index.html"
} |> ignore
{% endhighlight %}

When the page loads - we instantiate the User Manager with the `query` response mode, then we call `signinRedirectCallback` method on it and redirect the user to `index.html` page.

Invoke the following commands in `./src/Client/Auth` to reference required libraries:

```Powershell
dotnet add package Fable.Core -v 3.1.6 # Fable itself
dotnet add package Fable.Browser.Dom -v 1.1.0 # Required for manipulating DOM elements
dotnet add package Fable.OidcClient -v 1.0.2 # Required for handling the OpenID Connect protocol with oidc-client JavaScript library
dotnet add package Fable.Promise -v 2.1.0 # Required for defining a JavaScript promise in F# code
```

## IdentityServer4 Client settings

IdentityServer needs to define the client configuration: the client ID, the application URIs (such as callback redirect URL or where to redirect the user after one logs out) and scopes that this client is allowed to request access to:

{% highlight C# %}
public static IEnumerable<Client> Clients =>
    new Client[]
    {
        // Some other clients
        // ...
        // JavaScript Client
        new Client
        {
            ClientId = "js",
            ClientName = "JavaScript Client",
            AllowedGrantTypes = GrantTypes.Code,
            RequireClientSecret = false,

            RedirectUris =           { "https://localhost:5003/callback.html" },
            PostLogoutRedirectUris = { "https://localhost:5003/index.html" },
            AllowedCorsOrigins =     { "https://localhost:5003" },

            AllowedScopes =
            {
                IdentityServerConstants.StandardScopes.OpenId,
                IdentityServerConstants.StandardScopes.Profile,
                "scope1"
            }
        }
    };
{% endhighlight %}

## Api CORS settings

The Web API server needs to have appropriate CORS headers enabled. In case of Saturn framework the basic app configuration would look like the following:

{% highlight FSharp %}
let app =
    application {
        url "https://0.0.0.0:6001"
        // routing, static files, JWT authentication settings etc...
        use_cors "default" (fun policy ->
            policy.WithOrigins("https://localhost:5003").AllowAnyHeader().AllowAnyMethod() |> ignore
        )
    }
{% endhighlight %}

# Running the app

When all services are up and running (run the client with `npm start` from the root of your solution) - you should be able to go to `https://localhost:5003` and use all 3 buttons - initially you'll be logged out. When you click `Login` button - you'll be taken to the login page served by IdentityServer. There you can use one of these credentials: `alice`:`alice` or `bob`:`bob`. Then you'll be taken back to the `index.html` (with `callback.html` shortly visible as an intermediate page while the OpenID Connect protocol is carried out). From there you'll be able to use the `Call API` button to see the response from the API showing your identity claims. Finally - you can log out of the application, then you'll be redirected back to `index.html`.

# Proof Key for Code Exchange (PKCE)

As described on [IdentityServer4 Grant Types page](https://identityserver4.readthedocs.io/en/latest/topics/grant_types.html#interactive-clients) - the recommended way of securing a SPA is to use the Authorization Code Flow combined with Proof Key for Code Exchange (PKCE, pronounced "pixie") mechanism.

Enabling it is as simple as adding one line to the client configuration in IdentityServer4 `Config.cs`:

{% highlight C# %}
var client = new Client
{
    ClientId = "js",
    ClientName = "JavaScript Client",
    AllowedGrantTypes = GrantTypes.Code,
    RequirePkce = true,
    RequireClientSecret = false // Here!
};
{% endhighlight %}

If after making this change you're getting an error saying "Uncaught (in promise) Error: No matching state found in storage" in the browser console on `callback.html` page - restart the browser, that should fix it.

That's it folks! If you've got any questions, feel free to post one below or reach out [on Twitter](https://twitter.com/mjarosie).
