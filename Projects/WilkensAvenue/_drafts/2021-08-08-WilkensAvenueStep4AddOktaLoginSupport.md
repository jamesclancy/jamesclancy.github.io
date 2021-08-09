---
layout: post
title: Wilkens Avenue - Step 4 - Add Okta Support for Login
author: James Clancy
tags: fsharp dotnet safe-stack ci test github github-actions
---

Need to scope out and set up the initial pages for the first stage of the project. I will also set up the routing to these pages.

Initially at least, the site is going to be a SPA and not play nicely with SEO. This is largely fine as the project isn't truly a production site meant for general consumption.

## Goals of this Step
* Scope a list of pages
* Create Elmish routes to said pages

## Results

### List of normal pages
* Home
* Find
* Browse
* AddLocation
* YourLocations
* Login
* Register
* Logout
* YourAccount
* EditLocation
* ViewLocation
* About

### Pages without routes
* Not Found
* Unauthorized

### Implementation Details & Notes

Looking through the [Elmish documentation](https://elmish.github.io/browser/routing.html) this was quite simple. The call-out about tuples on that page was helpful, when creating the "Find" route I had to pass a lambda i.e. `map (fun x y z -> (x, y, z) |> Find) (s "Find" </> str </> str </> i32)`.

In addition to setting up the routing itself, I also extended the `Model` to become a discriminated union holding different typed models for each page. 

One thing that did cause some confusion was that the Elmish parser is case sensitive.


[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-3)