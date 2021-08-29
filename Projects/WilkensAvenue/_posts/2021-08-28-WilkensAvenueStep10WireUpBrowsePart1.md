---
layout: post
title: Wilkens Avenue - Step 10 - Wire up Browse Page - Part 1 Read
author: James Clancy
tags: fsharp dotnet safe-stack bulma-tagsinput mocha fails
---

## Goals of this Step
* Browse Page view model includes all data to render the page
* Browse page receives model from update and inturn api and renders the page

## Process

For now I am going to move on from the tags input issue and come back to it later. I think I have identified some possible solutions but I think I am using to new a version of webpack to utilize them. 

This required the implementation of a new API endpoint, several mappings between `DataTransferFormats`, modifications to the page views to pass through the values, updates to the url router to fire off the API call as well as a new Msg type to handle completed search requests.

### New Api Endpoint

In `Shared`
{% highlight FSharp %}
type ILocationInformationApi =
    { getLocation: string -> Async<LocationDetailModel>
      searchLocations: LocationSearchRequest -> Async<LocationSearchResult> }
{% endhighlight %}

In the `Server` implementation

{% highlight FSharp %}
let locationInformationApi =
    { getLocation = fun id -> async { return exampleLocation }
      searchLocations =
          fun req ->
              async {
                  return
                      { SearchRequest = req
                        TotalResults = 50
                        TotalPages = 1
                        CurrentPage = 1
                        Results = Some(List.ofSeq generateABunchOfItems) }
              } }
{% endhighlight %}

### DataTransferFormat Mappings

To the Client `Index` (a poor location but it works for now) I added:

{% highlight FSharp %}

let mapLocationSearchRequestToBrowseFilterModel (locationSearchResult: LocationSearchRequest) totalPages totalResults : BrowseFilterModel =
    let searchFilter =
        match locationSearchResult.Query with
        | s when System.String.IsNullOrWhiteSpace(s) -> None
        | s -> Some s

    {
      TotalResults = totalResults
      TotalPages = totalPages
      CurrentPage = locationSearchResult.CurrentPage
      SearchFilter = searchFilter
      ResultsPerPage = locationSearchResult.ItemsPerPage
      OnlyFree = locationSearchResult.FilterToFree
      OnlyOpenAir = locationSearchResult.FilterToOpenAir
      OnlyPrivate = locationSearchResult.FilterToPrivate
      FilterToZipCode = locationSearchResult.DistanceFilter.IsSome
      DistanceToFilterTo =
          locationSearchResult.DistanceFilter
          |> Option.map (fun x -> x.OriginZipCode) 
          |> Option.fold (fun x y -> x ) ""
      ZipCodeToFilterTo =
          locationSearchResult.DistanceFilter
          |> Option.map (fun x -> x.MaxDistance.ToString()) 
          |> Option.fold (fun x y -> x ) ""

      AvailableTags = [ "B&O"; "Railroad" ]
      AvailableCategories = [ "Steel"; "Port"; "Colonial" ]
      AvailableNeighborhoods =
          [ "Pigtown"
            "Mt Clare"
            "Pratt Monroe"
            "Carrolltown Ridge" ]


      SelectedTags = locationSearchResult.TagFilterFilter
      SelectedCategories = locationSearchResult.CategoryFilter
      SelectedNeighborhoods = locationSearchResult.NeighborhoodFilter }

let mapBrowseFilterModelToLocationSearchRequest (locationSearchResult:  BrowseFilterModel) : LocationSearchRequest =

    let decimalFromString (s : string) =
        match System.Decimal.TryParse s with
        | (true, value) -> value
        | (_, _) -> 15.0m

    let searchFilter =
        match locationSearchResult.SearchFilter with
        | None -> ""
        | Some s -> s

    let filterToDistance : LocationDistanceFilter option =
      match locationSearchResult.FilterToZipCode with
        | true -> Some { MaxDistance = decimalFromString locationSearchResult.DistanceToFilterTo
                         OriginZipCode = locationSearchResult.ZipCodeToFilterTo
                         MilesFromZipCode = 0m}
        | false -> None


    { Query = searchFilter
      FilterToFree = locationSearchResult.OnlyFree
      FilterToOpenAir = locationSearchResult.OnlyOpenAir
      FilterToPrivate  = locationSearchResult.OnlyPrivate
      DistanceFilter = filterToDistance

      TagFilterFilter =  locationSearchResult.SelectedTags
      CategoryFilter  = locationSearchResult.SelectedCategories
      NeighborhoodFilter  = locationSearchResult.SelectedNeighborhoods
      CurrentPage = 1
      ItemsPerPage = 50}

let mapBrowsePageFilterChangeToLocationSearchRequsst (change: BrowsePageFilterChange) =
    match change with
    | FilterChanged b ->mapBrowseFilterModelToLocationSearchRequest b
    | LoadNextPage  b ->mapBrowseFilterModelToLocationSearchRequest b
    | LoadPreviousPage  b ->mapBrowseFilterModelToLocationSearchRequest b

let mapSearchResultToReceievedBrowsePageResult (searchResult: LocationSearchResult) : Msg =
    { Filter = mapLocationSearchRequestToBrowseFilterModel (searchResult.SearchRequest)  searchResult.TotalPages searchResult.TotalResults
      Results = searchResult.Results
    }
    |> ReceivedBrowsePageResult

{% endhighlight %}

### Page View Modifications

A number of of the changes were made to the BrowseLocationsView. The main changes resolved around passing models into the various function calls and using the values from the model to populate the view.

### Updates to URL Router

{% highlight FSharp %}



{% endhighlight %}

### New Msg Type & Handler

To the Client `Contracts` I added a new type of 


{% highlight FSharp %}
type BrowsePageFilterChange =
    | FilterChanged of BrowseFilterModel
    | LoadNextPage of BrowseFilterModel
    | LoadPreviousPage of BrowseFilterModel

{% endhighlight %}

I then extended the Client `Msg` type like:

{% highlight FSharp %}
type Msg =
    | GotTodos of Todo list
    | SetInput of string
    | AddTodo
    | AddedTodo of Todo
    | ReceivedLocationDetail of LocationDetailModel
    | BrowsePageFilterChanged of BrowsePageFilterChange
    | ReceivedBrowsePageResult of BrowsePageModel

{% endhighlight %}

I then updated the update method on the Client `Index`.
{% highlight FSharp %}

let update (msg: Msg) (model: Model) : Model * Cmd<Msg> =
    match msg with
    | ReceivedLocationDetail d -> modelWithNewPageModel model (d |> ViewLocationPageModel), Cmd.none
    | ReceivedBrowsePageResult d ->
        modelWithNewPageModel model (d |> BrowsePageModel), Cmd.none
    | BrowsePageFilterChanged d -> 
        (model,
         (Cmd.OfAsync.perform locationInformationApi.searchLocations (mapBrowsePageFilterChangeToLocationSearchRequsst d) mapSearchResultToReceievedBrowsePageResult))
    | _ -> model, Cmd.none

{% endhighlight %}

## Results

Currently, the page can be viewed [here](https://wilkens-avenue.herokuapp.com/#/browse/12312).



[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-10)