---
layout: post
title: Wilkens Avenue - Step 18 - Fix Navbar
author: James Clancy
tags: fsharp dotnet safe-stack browse-page
---

## Goals of this Step
    * Navbar burger clicked is handled via the elmish messaging

## Process

After pushing the change for the menu burger yesterday I noticed that it was not working properly with the hacky javascript so I will have to implement the burger expander properly. 

To do this I updated the client `Model` to include a `MenuBurgerExpanded`: 

{% highlight FSharp %}

type Model =
    { CurrentRoute: Option<string>
      PageModel: PageModel
      CurrentUser: Option<string>
      MenuBurgerExpanded: bool }

{% endhighlight %}

I added a default value for this new field in the initial client model:

{% highlight FSharp %}
let init (initialRoute: Option<ClientRoute>) : Model * Cmd<Msg> =
    if initialRoute = None then
        { CurrentRoute = None
          CurrentUser = None
          PageModel = HomePageModel
          MenuBurgerExpanded = false},
        Cmd.none
    else
        urlUpdate
            initialRoute
            { CurrentRoute = None
              CurrentUser = None
              PageModel = LoadingScreenPageModel
              MenuBurgerExpanded = false }
{% endhighlight %}

I then added a BurgerToggled `Msg` type:

{% highlight FSharp %}
type Msg =
    | ToggleBurger
    | ReceivedLocationDetail of LocationDetailModel
    | BrowsePageFilterChanged of BrowsePageFilterChange
    | ReceivedBrowsePageResult of BrowsePageModel
    | LocationDetailUpdated of LocationDetailUpdate
    | LocationDetailUpdateServerResponseReceived of LocationDetailModel * UpdateLocationDetailState
{% endhighlight %}

I then implemented the update method to handle this event:

{% highlight FSharp %}
let update (msg: Msg) (model: Model) : Model * Cmd<Msg> =
    match msg, model.PageModel with
    | ToggleBurger, _ ->
        { model with MenuBurgerExpanded = not model.MenuBurgerExpanded}, Cmd.none
    | ReceivedLocationDetail d, _ ->
        modelWithNewPageModel
        ...
{% endhighlight %}

I set up the navbar to actually publish these events:
{% highlight FSharp %}
let navBar (model: Model) (dispatch: Msg -> unit) =

    let burgerActiveClass =
        seq {
            if model.MenuBurgerExpanded then
                "is-active"
            else
                ""
        }

    Html.div [ prop.children [ Bulma.navbar [ color.isLight
                                              navbar.isTransparent
                                              navbar.hasShadow
                                              navbar.isTransparent
                                              prop.style []

                                              prop.children [ Bulma.navbarBrand.div [ Bulma.navbarItem.a [ prop.href
                                                                                                               "#/"
                                                                                                           prop.children [ Html.h1 [ prop.text
                                                                                                                                         "Wilkens Avenue"
                                                                                                                                     prop.className
                                                                                                                                         "title navBarHeader" ] ] ]

                                                                                      Bulma.navbarBurger [ prop.custom (
                                                                                                               "data-target",
                                                                                                               "nav-menu"
                                                                                                           )
                                                                                                           prop.classes
                                                                                                               burgerActiveClass
                                                                                                           prop.onClick
                                                                                                               (fun _ ->
                                                                                                                   ToggleBurger
                                                                                                                   |> dispatch)
                                                                                                           prop.children [ Html.span [  ]
                                                                                                                           Html.span [  ]
                                                                                                                           Html.span [  ] ] ] ]

                                                              Bulma.navbarMenu [ prop.id "nav-menu"
                                                                                 prop.classes burgerActiveClass
                                                                                 prop.children [ Bulma.navbarStart.div [ Bulma.navbarItem.a [ prop.text
                                                                                 ...
{% endhighlight %}

Finally, I updated my various function to pass along the Model so the navbar can consume it and removed the hack from the `index.html`. What may be a better future iteration would be to pass along the navbar itself instead of the model and the dispatch function. 

## Results

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-18)