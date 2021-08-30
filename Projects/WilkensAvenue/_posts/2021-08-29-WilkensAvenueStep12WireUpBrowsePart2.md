---
layout: post
title: Wilkens Avenue - Step 12 - Wire up Browse Page - Part 2 Send back events
author: James Clancy
tags: fsharp dotnet safe-stack browse-page
---

## Goals of this Step
* Browse Page view dispatches events when filters are changed
* Client researches for updates from the API when filter change events occur

## Process

I created a new module `Shared.StaticData` which contains some static lists for possible tags, categories and neighborhoods. 

I added `onClick` hooks to the various left menu inputs. Each of these hooks modifies the passed BrowseMenuFilter to reflect the changed value and packages it up to resend to the event loop. 

Lastly, I added Next and Previous buttons to the bottom of the search results. 

After going through this process I noticed a few issues. The InputTags (despite having case sensitivity off) was still being case sensitive. I added a relatively hacky implementation to force the filter to not be case sensitive. 

{% highlight FSharp %}

tagsInput.autoCompleteSource (
                          (fun text ->
                              async {
                                  let lowerCaseText = text.ToString().ToLower()

                                  return
                                      possibleValues
                                      |> List.filter (fun x -> x.ToString().ToLower().Contains(lowerCaseText))
                              })
                      )

{% endhighlight %}

Also, my results page was not displaying the correct number of results so I has to update the view like:

{% highlight FSharp %}

let displaySearchResults (browseViewModel : BrowsePageModel) (dispatch: Msg -> unit) =
    let menuDispatch menuBrowse =
        menuBrowse |> BrowsePageFilterChanged |> dispatch
    let locations = browseViewModel.Results
    match locations with
    | None
    | Some [] -> [ Html.div [ prop.children [ Html.b [ prop.text "nothing found" ] ] ] ]
    | Some items ->
        [ Html.div [ prop.children [ Html.b [ prop.text (sprintf "A total of %i results found. You are on page %i of %i"  browseViewModel.Filter.TotalResults browseViewModel.Filter.CurrentPage browseViewModel.Filter.TotalPages)  ] ] ]
          Bulma.container [ container.isFluid
  
{% endhighlight %}

## Results

Currently, the page can be viewed [here](https://wilkens-avenue.herokuapp.com/#/browse/12312).

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-12)