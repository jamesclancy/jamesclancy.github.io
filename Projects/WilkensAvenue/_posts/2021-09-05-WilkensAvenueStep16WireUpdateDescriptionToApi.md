---
layout: post
title: Wilkens Avenue - Step 15 - Wire Update Description to Api
author: James Clancy
tags: fsharp dotnet safe-stack browse-page
---

## Goals of this Step
    * Saving edits to the summary & description on the Location Details page should fire an api to the backend server and display the error message if one is sent. 

## Process

To wire this up I first created data transfer formats:

{% highlight FSharp %}{% raw %}
type LocationDetailUpdateRequest =
    { Id: string
      Summary: string
      Description: string }

type LocationDetailUpdateResult = { ErrorMessage: string option }
{% endraw %}{% endhighlight %}

I then added an `updateLocationDetails` to the `ILocationInformationApi` like 

{% highlight FSharp %}{% raw %}
type ILocationInformationApi =
    { getLocation: string -> Async<LocationDetailModel>
      searchLocations: LocationSearchRequest -> Async<LocationSearchResult>
      updateLocationDetails: LocationDetailUpdateRequest -> Async<LocationDetailUpdateResult> }
{% endraw %}{% endhighlight %}

I update the server to return accept the request and return a response (but not actually do anything)

{% highlight FSharp %}{% raw %}
let locationInformationApi =
    { getLocation = fun id -> async { return exampleLocation }
      searchLocations =
          ...
      updateLocationDetails = fun req -> async { return { ErrorMessage = None } }
    }
{% endraw %}{% endhighlight %}

I added an `ErrorMessage` to the `UpdateLocationDetailState`:

{% highlight FSharp %}{% raw %}
type UpdateLocationDetailState =
    { EditingSummary: bool
      NewSummaryContent: string
      EditingDescription: bool
      NewDescriptionContent: string
      ErrorMessage: string option }
{% endraw %}{% endhighlight %}

And I added a `LocationDetailUpdateServerResponseReceived` the client `Msg` type:

{% highlight FSharp %}{% raw %}
type Msg =
    | ReceivedLocationDetail of LocationDetailModel
    | BrowsePageFilterChanged of BrowsePageFilterChange
    | ReceivedBrowsePageResult of BrowsePageModel
    | LocationDetailUpdated of LocationDetailUpdate
    | LocationDetailUpdateServerResponseReceived of LocationDetailModel * UpdateLocationDetailState
{% endraw %}{% endhighlight %}

I set up the mapping to create the request from the states and generate the new `LocationDetailUpdateServerResponseReceived` from the result:

{% highlight FSharp %}{% raw %}
let mapLocationDetailToEditRequest (id: string) currentEditState =
    { Id = id
      Summary = currentEditState.NewSummaryContent
      Description = currentEditState.NewDescriptionContent }

let mapUpdateResultsToUpdateDetailResponseReceived
    (d: LocationDetailUpdate)
    (currPage: LocationDetailModel)
    (editState: UpdateLocationDetailState)
    response
    =

    (currPage,
     { editState with
           ErrorMessage = response.ErrorMessage })
    |> LocationDetailUpdateServerResponseReceived
{% endraw %}{% endhighlight %}


In the LocationDetails, I then updated the `defaultEditState` to include the `ErrorMessage` and the `updateLocationDetailsModel` to also return a command which can trigger the result. 

{% highlight FSharp %}{% raw %}
let defaultEditState (model: LocationDetailModel) : UpdateLocationDetailState =
    { EditingSummary = false
      NewSummaryContent = stringEmptyOrValue model.Summary
      EditingDescription = false
      NewDescriptionContent = stringEmptyOrValue model.Description
      ErrorMessage = None }

let updateLocationDetailsModel
    (d: LocationDetailUpdate)
    (currPage: LocationDetailModel)
    (editState: UpdateLocationDetailState)
    updateMethod
    =

    let buildCmdForSave =
        Cmd.OfAsync.perform
            updateMethod
            (ContractMappings.mapLocationDetailToEditRequest currPage.Id editState)
            (ContractMappings.mapUpdateResultsToUpdateDetailResponseReceived d currPage editState)

    match d with
    | SummaryStartEdit -> currPage, { editState with EditingSummary = true }, Cmd.none
    | SummaryTextChanged s ->
        currPage, { editState with NewSummaryContent = s }, Cmd.none
    | SummaryTextSaved ->
        { currPage with
              Summary = Some editState.NewSummaryContent },
        { editState with
              EditingSummary = false }, buildCmdForSave
    | SummaryTextChangeCanceled ->
        currPage,
        { editState with
              EditingSummary = false
              NewSummaryContent = stringEmptyOrValue currPage.Summary }, Cmd.none
    | DescriptionStartEdit ->
        currPage,
        { editState with
              EditingDescription = true }, Cmd.none
    | DescriptionTextChanged s ->
        currPage,
        { editState with
              NewDescriptionContent = s }, Cmd.none
    | DescriptionTextSaved ->
        { currPage with
              Description = Some editState.NewDescriptionContent },
        { editState with
              EditingDescription = false }, buildCmdForSave
    | DescriptionTextChangeCanceled ->
        currPage,
        { editState with
              EditingDescription = false
              NewDescriptionContent = stringEmptyOrValue currPage.Description }, Cmd.none
{% endraw %}{% endhighlight %}

I update the view to display the error message if present like:

{% highlight FSharp %}{% raw %}
    let errorMessage (msg : string option) =
        match msg with
        | None -> Seq.empty
        | Some s -> seq {
            Bulma.notification [
                Bulma.color.isDanger
                prop.text s
                ]
            }

    let pageContent
        (model: LocationDetailModel)
        (editState: UpdateLocationDetailState)
        (dispatch: LocationDetailUpdate -> unit)
        =
        [ Bulma.title [ title.is1
                        prop.className "mb-5"
                        prop.text model.Name ]
          Bulma.title [ title.is2
                        prop.className "has-text-grey "
                        prop.text model.SubTitle ]
          yield! errorMessage editState.ErrorMessage
          yield!
              editablePageSection
              ...
{% endraw %}{% endhighlight %}

Lastly, I hooked this into the general index update function:


{% highlight FSharp %}{% raw %}
let update (msg: Msg) (model: Model) : Model * Cmd<Msg> =
    match msg, model.PageModel with
    | ReceivedLocationDetail d, _ ->
        modelWithNewPageModel
            model
            ((d, Pages.LocationDetails.defaultEditState d)
             |> ViewLocationPageModel),
        Cmd.none
    | ReceivedBrowsePageResult d, _ -> modelWithNewPageModel model (d |> BrowsePageModel), Cmd.none
    | BrowsePageFilterChanged d, _ ->
        (model,
         (Cmd.OfAsync.perform
             locationInformationApi.searchLocations
             (mapBrowsePageFilterChangeToLocationSearchRequest d)
             mapSearchResultToReceievedBrowsePageResult))
    | LocationDetailUpdated d, ViewLocationPageModel (currPage, currentEditState) ->
        let (pm,ed, cmd) =  Pages.LocationDetails.updateLocationDetailsModel d currPage currentEditState locationInformationApi.updateLocationDetails
        { model with
              PageModel =
                 (pm, ed)
                  |> ViewLocationPageModel }, cmd

    | _, _ -> model, Cmd.none
{% endraw %}{% endhighlight %}

## Results

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-16)