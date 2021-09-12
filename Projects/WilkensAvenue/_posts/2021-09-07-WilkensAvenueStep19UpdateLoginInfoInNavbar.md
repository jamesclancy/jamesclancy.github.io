---
layout: post
title: Wilkens Avenue - Step 19 - Update Login Information in Navbar
author: James Clancy
tags: fsharp dotnet safe-stack browse-page
---

## Goals of this Step
    * Navbar reflects if the user is logged in
    * If the user is not logged in they are offered the option to login or register
    * If the user is logged in they are offered the option to logout

## Process

In order to check if a user is logged in I am adding a new `IPublicAccountApi` which is going to return the current logged in user and `ISecureAccountApi` which prompts the user to login. 

{% highlight FSharp %}
type IPublicAccountApi =
    {
        getCurrentUser: unit -> Async<Option<string>>
    }

type ISecureAccountApi =
    {
        login: unit ->  Async<Option<string>>
    }
{% endhighlight %}

I added function to `Authentication` to pull the user information from the `HttpContext`

{% highlight FSharp %}
let userInformationFromContext (ctx :HttpContext) =
    let nameClaim = Seq.filter (fun (x: System.Security.Claims.Claim) -> x.Type = "playerName") ctx.User.Claims |> Seq.toList
    let idClaim = Seq.filter (fun (x: System.Security.Claims.Claim) -> x.Type = "playerId") ctx.User.Claims |> Seq.toList

    match nameClaim, idClaim with
    | [ x ], [ y ] -> Some (x.Value, y.Value)
    | [] , [y] -> Some (y.Value, y.Value)
    | _,_ -> None

let userDisplayNameFromContext (ctx : HttpContext) =
    match userInformationFromContext ctx with
    | None ->  None
    | Some (x, y) -> Some x
{% endhighlight %}

I could then implement the server like:

{% highlight FSharp %}
let generalPurposeApi ctx =
        {
            getCurrentUser = fun unit -> async { return userDisplayNameFromContext ctx }
        }

let accountInformationApi ctx =
        {
            login = fun unit -> async { return userDisplayNameFromContext ctx }
        }

let buildLocationApi next ctx =
    task {
        let handler =
            Remoting.createApi ()
            |> Remoting.withRouteBuilder Route.builderWithoutApiPrefix
            |> Remoting.fromValue locationInformationApi
            |> Remoting.buildHttpHandler
        return! handler next ctx
    }

let buildSecureAccountApi next ctx =
        task {
            let handler =
                Remoting.createApi ()
                |> Remoting.withRouteBuilder Route.builderWithoutApiPrefix
                |> Remoting.fromValue accountInformationApi
                |> Remoting.buildHttpHandler
            return! handler next ctx
        }
        
let buildPublicAccountApi next ctx =
                task {
                    let handler =
                        Remoting.createApi ()
                        |> Remoting.withRouteBuilder Route.builderWithoutApiPrefix
                        |> Remoting.fromValue generalPurposeApi
                        |> Remoting.buildHttpHandler
                    return! handler next ctx
                }

let routes : HttpFunc -> HttpContext -> HttpFuncResult =
    choose [ route "/loggedinhomepage"
             >=> (authChallenge
                  >=> htmlString "You are logged in now.")
             subRoute "/api" ( buildLocationApi)
             subRoute "/api" ( buildPublicAccountApi)
             subRoute "/api" ( authChallenge >=> buildSecureAccountApi)]
{% endhighlight %}

In order to implement this on the client I registered instances of the IPublicAccountApi and ISecureAccountApi

{% highlight FSharp %}
let secureAccountApi =
    Remoting.createApi ()
    |> Remoting.withRouteBuilder Route.builder
    |> Remoting.buildProxy<ISecureAccountApi>
let publicAccountApi =
    Remoting.createApi ()
    |> Remoting.withRouteBuilder Route.builder
    |> Remoting.buildProxy<IPublicAccountApi>
{% endhighlight %}

I know need to add a subscription in the elmish app to get current logged in user. 

To accomplish this I first added two subtypes to the `Msg`

{% highlight FSharp %}
type Msg =
    | ToggleBurger
    | UserInformationRequired
    | UserInformationFetched of string option
    | ReceivedLocationDetail of LocationDetailModel
    | BrowsePageFilterChanged of BrowsePageFilterChange
    | ReceivedBrowsePageResult of BrowsePageModel
    | LocationDetailUpdated of LocationDetailUpdate
    | LocationDetailUpdateServerResponseReceived of LocationDetailModel * UpdateLocationDetailState
{% endhighlight %}

Then I added the logic to respond to the messages. 

{% highlight FSharp %}
let update (msg: Msg) (model: Model) : Model * Cmd<Msg> =
    match msg, model.PageModel with
    | ToggleBurger, _ ->
        { model with
              MenuBurgerExpanded = not model.MenuBurgerExpanded },
        Cmd.none
    | UserInformationRequired ,_ ->
        model, Cmd.OfAsync.perform publicAccountApi.getCurrentUser () UserInformationFetched
    | UserInformationFetched u, _ ->
        { model with
              CurrentUser = u },
        Cmd.none
    | ReceivedLocationDetail d, _ ->
        modelWithNewPageModel
        ...
{% endhighlight %}


I then registered a timer like:

{% highlight FSharp %}
let userInformationUpdateRequired initial =
    let sub dispatch=
        window.setInterval((fun _ ->
            dispatch (UserInformationRequired)), 1000) |> ignore
    Cmd.ofSub sub

Program.mkProgram Index.init Index.update Index.view
|> Program.withSubscription userInformationUpdateRequired
|> Program.toNavigable (parseHash clientRouter) urlUpdate
#if DEBUG
|> Program.withConsoleTrace
#endif
|> Program.withReactSynchronous "elmish-app"
#if DEBUG
|> Program.withDebugger
#endif
|> Program.run

{% endhighlight %}

Note this had to go above the Program.toNavigable. If the subscription is added after hte `toNavigatable` the type passed to the dispatch is incorrect. 

This now loads and runs but the Api requests fail with: `"Protocol definition must be encoded as a record type. The input type 'FSharpFunc``2'` was not a record."` This is because the Api need to return a fully defined type and not an `option string`.

The shared `DataTransferFormats` I added `CurrentUserLoginStatusResponse`:

{% highlight FSharp %}
type CurrentUserLoginStatusResponse =
    {
        CurrentlyLoggedIn: bool
        CurrentUserId: string option
        CurrentUserName: string option
    }
{% endhighlight %}

In authorization I updated the function to pull information from the context:

{% highlight FSharp %}
let userInformationFromContext (ctx: HttpContext) =
    let nameClaim =
        Seq.filter (fun (x: System.Security.Claims.Claim) -> x.Type = "playerName") ctx.User.Claims
        |> Seq.toList

    let idClaim =
        Seq.filter (fun (x: System.Security.Claims.Claim) -> x.Type = "playerId") ctx.User.Claims
        |> Seq.toList

    match nameClaim, idClaim with
    | [ x ], [ y ] ->
        { CurrentlyLoggedIn = true
          CurrentUserId = Some y.Value
          CurrentUserName = Some x.Value }
    | [], [ y ] ->
        { CurrentlyLoggedIn = true
          CurrentUserId = Some y.Value
          CurrentUserName = Some y.Value }
    | _, _ ->
        { CurrentlyLoggedIn = false
          CurrentUserId = None
          CurrentUserName = None }
{% endhighlight %}

I updated the apis to return this type.

{% highlight FSharp %}
type IPublicAccountApi =
    { getCurrentUser: unit -> Async<CurrentUserLoginStatusResponse> }

type ISecureAccountApi =
    { login: unit -> Async<CurrentUserLoginStatusResponse> }
{% endhighlight %}

And implemented it with:

{% highlight FSharp %}
let generalPurposeApi ctx =
    { getCurrentUser = fun unit -> async { return userInformationFromContext ctx } }

let accountInformationApi ctx =
    { login = fun unit -> async { return userInformationFromContext ctx } }
{% endhighlight %}

This now loads and runs but the Api requests fails with the same error: `"Protocol definition must be encoded as a record type. The input type 'FSharpFunc``2'` was not a record."`

It turns out I was mistaken about hte reason for the error. I just had a typo in my buildApi methods. I needed to change them to:

{% highlight FSharp %}s
let buildSecureAccountApi next ctx =
    task {
        let handler =
            Remoting.createApi ()
            |> Remoting.withRouteBuilder Route.builderWithoutApiPrefix
            |> Remoting.fromValue (accountInformationApi ctx)
            |> Remoting.buildHttpHandler

        return! handler next ctx
    }

let buildPublicAccountApi next ctx =
    task {
        let handler =
            Remoting.createApi ()
            |> Remoting.withRouteBuilder Route.builderWithoutApiPrefix
            |> Remoting.fromValue (accountPublishApi ctx)
            |> Remoting.buildHttpHandler

        return! handler next ctx
    }
{% endhighlight %}

Now it is running correctly. 


I then had to add the logic to actually login and logout of the site as normal paths on the server

In the `Server` I updated:

{% highlight FSharp %}
let routes : HttpFunc -> HttpContext -> HttpFuncResult =
    choose [ route "/login"  >=> (authChallenge >=> redirectTo false "/")
             route "/signout" >=> signOut >=>   htmlString "Logged Out."
             subRoute "/api" (buildLocationApi)
             ...
{% endhighlight %}

Under authentication I added 

{% highlight FSharp %}
let signOut (next : HttpFunc) (ctx : HttpContext) =
  task {
    do! ctx.SignOutAsync()
    return! next ctx
  }
{% endhighlight %}

I then added the `login`, `logout` and `signin-oidc` paths to the webpack proxy for the dev server. 

```
var CONFIG = {
    // The tags to include the generated JS and CSS will be automatically injected in the HTML template
    // See https://github.com/jantimon/html-webpack-plugin
    indexHtmlTemplate: './src/Client/index.html',
    fsharpEntry: './src/Client/App.fs.js',
    outputDir: './deploy/public',
    assetsDir: './src/Client/public',
    devServerPort: 8080,
    // When using webpack-dev-server, you may need to redirect some calls
    // to a external API server. See https://webpack.js.org/configuration/dev-server/#devserver-proxy
    devServerProxy: {
        // redirect requests that start with /api/ to the server on port 8085
        '/api/**': {
            target: 'http://localhost:' + (process.env.SERVER_PROXY_PORT || "8085"),
            changeOrigin: true
        },
        // redirect websocket requests that start with /socket/ to the server on the port 8085
        '/socket/**': {
            target: 'http://localhost:' + (process.env.SERVER_PROXY_PORT || "8085"),
            ws: true
        },
        // redirect websocket requests that start with /socket/ to the server on the port 8085
        '/login**': {
            target: 'http://localhost:' + (process.env.SERVER_PROXY_PORT || "8085"),
            changeOrigin: true
        },
        // redirect websocket requests that start with /socket/ to the server on the port 8085
        '/logout**': {
            target: 'http://localhost:' + (process.env.SERVER_PROXY_PORT || "8085"),
            changeOrigin: true
        },
        // redirect websocket requests that start with /socket/ to the server on the port 8085
        '/signin-oidc**': {
            target: 'http://localhost:' + (process.env.SERVER_PROXY_PORT || "8085"),
            changeOrigin: true
        }
    }
}
```

I then added the html to render the login/register or logout. 

{% highlight FSharp %}
    let navBarLoginContent =
        match model.CurrentUser with
        | None ->
            [ Bulma.navbarItem.div [ Bulma.buttons [ Bulma.button.a [ prop.text "Login or Register"; prop.href "/login" ] ] ] ]
        | Some s ->  [ Bulma.navbarItem.div [ Bulma.buttons [ Bulma.button.a [ prop.text "Logout"; prop.href "/logout" ] ] ] ]
{% endhighlight %}

I am now brought to Okta to login but when it completes the redirect page is broken. 

` The cookie '.AspNetCore.OpenIdConnect.Nonce... has set 'SameSite=None' and must also set 'Secure'.`

I was able to update the cookie settings like so

{% highlight FSharp %}
type Saturn.Application.ApplicationBuilder with
    [<CustomOperation("use_open_id_auth_with_config_from_service_collection")>]
    member __.UseOpenIdAuthWithConfigFromServiceCollection
        (
            state: ApplicationState,
            (config: IServiceCollection -> Action<OpenIdConnect.OpenIdConnectOptions>)
        ) =
        let middleware (app: IApplicationBuilder) = app.UseAuthentication()

        let service (s: IServiceCollection) =
            let authBuilder =
                s.AddAuthentication
                    (fun authConfig ->
                        authConfig.DefaultScheme <- CookieAuthenticationDefaults.AuthenticationScheme
                        authConfig.DefaultChallengeScheme <- OpenIdConnectDefaults.AuthenticationScheme
                        authConfig.DefaultSignInScheme <- CookieAuthenticationDefaults.AuthenticationScheme
                        )

            if not state.CookiesAlreadyAdded then
                authBuilder.AddCookie((fun opt ->
                                                  opt.Cookie.SameSite <- SameSiteMode.Lax
                                                  opt.Cookie.SecurePolicy <- CookieSecurePolicy.None
                                                  opt.Cookie.IsEssential <- true )) |> ignore

            authBuilder.AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, (config s))
            |> ignore

            s

        { state with
              ServicesConfig = service :: state.ServicesConfig
              AppConfigs = middleware :: state.AppConfigs
              CookiesAlreadyAdded = true }
{% endhighlight %}

I am still getting an error ` cookie not found.`. I am thinking this is because the url/ ip generated for the redirect is at port 8085 while the client is being served by webpack at 8080. i.e. Webpack is proxying the request to the server @ 8085 which uses the cookie info from 8080 and the port from the server instance to create the request to Okta. Okta is then redirecting back to 8085 which is unable to find the cookie from the client port. 

I tried enabling forwarded headers in asp.net:

{% highlight FSharp %}
        let service (s: IServiceCollection) =
            s.Configure<ForwardedHeadersOptions>(fun (options:ForwardedHeadersOptions) ->
                options.ForwardedHeaders <- ForwardedHeaders.All
                options.KnownNetworks.Clear() |> ignore
                options.KnownProxies.Clear() |> ignore
            ) |> ignore
            
            let authBuilder =
            ...
{% endhighlight %}

This did not do what was needed. I am assuming I also need to configure WebPack to set forward headers.


After updating Webpack to forward headers it still was not working. After reading a lot of blog posts I found that that issue was 

{% highlight FSharp %}
let addAuth (app: IApplicationBuilder) =
    app.UseCookiePolicy(new CookiePolicyOptions(MinimumSameSitePolicy=SameSiteMode.Lax,Secure=CookieSecurePolicy.None)).UseAuthentication()
{% endhighlight %}

which was being called last in the ApplicationBuilder, this was wiping out the cookies. I removed the line and it started to work in firefox but not in Chrome.

## Results

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-19)