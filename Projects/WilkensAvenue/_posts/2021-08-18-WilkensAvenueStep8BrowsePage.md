---
layout: post
title: Wilkens Avenue - Step 8 - Start Scaffolding Page Layouts - The Browse Page
author: James Clancy
tags: fsharp dotnet safe-stack ci test github github-actions
---

I am thinking that sketching out some layouts of the various pages from Step 3 might help scope out the domain.


## Goals of this Step
* Sketch of BrowsePage

## Process

I mocked up a very basic Browse Page and came up with Filter/Search options of:

{% highlight FSharp %}
type LocationDistanceFilter =
    { MaxDistance: decimal
      OriginZipCode: string }

type LocationSearchRequest =
    { CurrentPage: int
      ItemsPerPage: int
      Query: string

      FilterToFree: bool
      FilterToOpenAir: bool
      FilterToPrivate: bool

      NeighborhoodFilter: string list option
      TagFilterFilter: string list option
      CategoryFilter: string list option
      DistanceFilter: LocationDistanceFilter option }
{% endhighlight %}

And displaying a list of:
{% highlight FSharp %}

type LocationSummaryViewModel =
    { Id: string
      NeighborhoodId: string
      Neighborhood: string
      Name: string
      SubTitle: string

      Summary: Option<string>

      Address: Option<AddressDetailModel>

      ThumbnailUrl: string
      ThumbnailHeight: int
      ThumbnailWidth: int }
{% endhighlight %}

This will be returned from the API in a container:

type LocationSummaryViewModel =
    { Id: string
      NeighborhoodId: string
      Neighborhood: string
      Name: string
      SubTitle: string

      Summary: Option<string>

      Address: Option<AddressDetailModel>

      ThumbnailUrl: string
      ThumbnailHeight: int
      ThumbnailWidth: int }
{% endhighlight %}

It looks like there is an issue with referencing the `Feliz.Bulma.TagsInput` library which was introduced in this PR. The application appears to still work fine but the tests are failing and react throws  `Invalid hook call. Hooks can only be called inside of the body of a function component.` I believe this may be a mismatched version of react referenced in the library? The application still appears to work so I am pushing it up and merging for now. I will have a spend some time investigating the cause of this.



## Results

Currently, the page can be viewed [here](https://wilkens-avenue.herokuapp.com/#/browse/12312).



[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-8)