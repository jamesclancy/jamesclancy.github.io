---
layout: post
title: Wilkens Avenue - Step 4 - Add Okta Support for Login
author: James Clancy
tags: fsharp dotnet safe-stack ci test github github-actions
---

I previously used Github OAuth to authenticate the SafeCardGame, I wanted to try using Okta. 


## Goals of this Step
* Create Okta account
* Create way to read secret keys/environmental variables with Okta api keys
* Add Okta configuration and a route which requires login.

## Results

This appeared to be very straightforward, unfortunately I was unable to find a way to cleanly pull the configuration from the service configuration in the Saturn provided `use_open_id_auth_with_config`. All examples I could find were pulling from static methods and not from the service configuration. While this may be the *functional way*, I did want to be able to utilize the Microsoft supplied `AddUserSecrets` and `AddEnvironmentVariables` which register values in the `IConfiguration` in the service configuration. 

Therefor, I had to implement my own `use_open_id_auth_with_config_from_service_collection`. To do this I essentially copy and pasted the original [`use_open_id_auth_with_config`](https://github.com/SaturnFramework/Saturn/blob/9593e90e1397309c4653abe703478893b60daa5e/src/Saturn.Extensions.Authorization/OAuth.fs#L189) and modified it to accept a function with maps from a IServiceCollection to the required confuration like so:

{% highlight FSharp %}
type Saturn.Application.ApplicationBuilder with
    [<CustomOperation("use_open_id_auth_with_config_from_service_collection")>]
    member __.UseOpenIdAuthWithConfigFromServiceCollection(state: ApplicationState, (config: IServiceCollection -> Action<OpenIdConnect.OpenIdConnectOptions>)) =
        let middleware (app : IApplicationBuilder) =
            app.UseAuthentication()

        let service (s: IServiceCollection) =
            let authBuilder = s.AddAuthentication(fun authConfig ->
                authConfig.DefaultScheme <- CookieAuthenticationDefaults.AuthenticationScheme
                authConfig.DefaultChallengeScheme <- OpenIdConnectDefaults.AuthenticationScheme
                authConfig.DefaultSignInScheme <- CookieAuthenticationDefaults.AuthenticationScheme)
            if not state.CookiesAlreadyAdded then authBuilder.AddCookie() |> ignore
            authBuilder.AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, (config s)) |> ignore
            s

        { state with
            ServicesConfig = service::state.ServicesConfig
            AppConfigs = middleware::state.AppConfigs
            CookiesAlreadyAdded = true }
{% endhighlight %}

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-4)