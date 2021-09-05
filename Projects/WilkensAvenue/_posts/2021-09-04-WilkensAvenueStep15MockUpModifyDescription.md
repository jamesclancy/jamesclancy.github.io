---
layout: post
title: Wilkens Avenue - Step 15 - Mock Up Modify Description Modal
author: James Clancy
tags: fsharp dotnet safe-stack browse-page
---

## Goals of this Step
    * Mock up/ off a modal for the user to be able to modify descriptions of the locations. 
    * This would be the functionality the user encountered when they click `Submit Update to Description` on the location details pages.  

## Process

Doing some research Quill seams like a good way to add this functionality.  I found a [Feliz.Quill module](https://dzoukr.github.io/Feliz.Quill/#/). 

using `femto install Feliz.Quill .\src\Client\Client.fsproj` failed with 

```
[05:12:41 INF] Found package.json in C:\tests\WilkensAvenue
[05:12:44 INF] Executing required actions for package resolution
[05:12:44 INF] Uninstalling [mocha]
[05:12:48 INF] Installing dependencies [react-quill@2.0.0-beta1, quill-blot-formatter@1.0.5]
[05:12:50 ERR] Error while installing react-quill@2.0.0-beta1, quill-blot-formatter@1.0.5
```
Mocha appears to have tried to uninstall mocha and install old versions of quill which are no longer available. This is undesirable behavior so I updated my packages.json to add mocha back and added `"react-quill": "^2.0.0-beta.4"` and `"quill-blot-formatter": "^1.0.5"` under the dependencies. 


I then modified the `LocationDetails` page like 

{% highlight FSharp %}
...
    let rec editablePageSection sectionTitle previousValue =
        match previousValue with
          | None -> editablePageSection sectionTitle (Some "")
          | Some s -> 
                Quill.editor [
                    editor.toolbar [
                        [ Header (ToolbarHeader.Dropdown [1..4]) ]
                        [ ForegroundColor; BackgroundColor ]
                        [ Bold; Italic; Underline; Strikethrough; Blockquote; Code ]
                        [ OrderedList; UnorderedList; DecreaseIndent; IncreaseIndent; CodeBlock ]
                    ]
                    editor.defaultValue s
                ]

    let pageContent (model: LocationDetailModel) =
        [ Bulma.title [ title.is1
                        prop.className "mb-5"
                        prop.text model.Name ]
          Bulma.title [ title.is2
                        prop.className "has-text-grey "
                        prop.text model.SubTitle ]
          editablePageSection "Summary" model.Summary
          editablePageSection "Description" model.Description

...
{% endhighlight %}

and then I can see the rich text editors:

![Screenshot RTE-1](\assets\img\post-media\2021-09-03-WilkensAvenueStep15MockUpModifyDescription\RTE-1.png)

I need to make this toggle from editable to non-editable, add titles and maybe add a save button locally near the text you are editing. 

After this I needed to add some way so toggle the editing on and off so I added a boolEditingEnabled like 

{% highlight FSharp %}
    let rec editablePageSection (sectionTitle: string) previousValue boolEditingEnabled =
      match previousValue with
      | None -> editablePageSection sectionTitle (Some "") boolEditingEnabled
      | Some s ->
        if not boolEditingEnabled then 
            seq {
                yield
                    Bulma.section [ prop.children [ Bulma.title [ prop.children [ Html.span [prop.text sectionTitle]; Html.i [ prop.className "fas fa-edit is-pulled-right" ]  ] ]
                                                    Html.p [ prop.text s ] ] ]
            }
        else
                 seq {
                            yield
                                Bulma.section [ prop.children [ Bulma.title [ prop.text sectionTitle  ]
                                                                Quill.editor [
                                                                    editor.toolbar [
                                                                        [ Header (ToolbarHeader.Dropdown [1..4]) ]
                                                                        [ ForegroundColor; BackgroundColor ]
                                                                        [ Bold; Italic; Underline; Strikethrough; Blockquote; Code ]
                                                                        [ OrderedList; UnorderedList; DecreaseIndent; IncreaseIndent; CodeBlock ]
                                                                    ]
                                                                    editor.defaultValue s
                                                                    editor.onTextChanged (fun x -> Console.WriteLine(x))
                                                                ]
                                                                Bulma.button.button [
                                                                    button.isLarge
                                                                    prop.text "Save"
                                                                ] ] ]
                        }

    let pageContent (model: LocationDetailModel) =
        [ Bulma.title [ title.is1
                        prop.className "mb-5"
                        prop.text model.Name ]
          Bulma.title [ title.is2
                        prop.className "has-text-grey "
                        prop.text model.SubTitle ]
          yield! editablePageSection "Summary" model.Summary true
          yield! editablePageSection "Description" model.Description false
          yield! addressSection model.Address
          yield! displayImages model.Images
          yield!
              modifyTextInParagraphOrYieldNothing
                  "has-text-grey mb-5"
                  (fun x -> "Description from: " + x)
                  model.DescriptionCitation
          Html.div [ prop.className "buttons"
                     prop.children [ Html.a [ prop.className "button is-primary"
                                              prop.href "https://github.com/jamesclancy/WilkensAvenue"
                                              prop.text "Find Directions" ]
                                     Html.a [ prop.className "button is-primary"
                                              prop.href "https://github.com/jamesclancy/WilkensAvenue"
                                              prop.text "Upload Images" ]
                                     Html.a [ prop.className "button is-primary"
                                              prop.href "https://github.com/jamesclancy/WilkensAvenue"
                                              prop.text "Submit Update to Description" ] ] ] ]
{% endhighlight %}

This allows the "editable fields" to be toggled on and off. When off the user can click an edit icon (which doesn't do anything at the moment) and save the results when editing which should save the results and shift the field into the uneditable state. 

To actually wire this up I added a Client Msg subtype:

{% highlight FSharp %}
type LocationDetailUpdate =
    | SummaryStartEdit
    | SummaryTextChanged of string
    | SummaryTextSaved
    | SummaryTextChangeCanceled
    | DescriptionStartEdit
    | DescriptionTextChanged of string
    | DescriptionTextSaved
    | DescriptionTextChangeCanceled

type Msg =
    | ReceivedLocationDetail of LocationDetailModel
    | BrowsePageFilterChanged of BrowsePageFilterChange
    | ReceivedBrowsePageResult of BrowsePageModel
    | LocationDetailUpdated of LocationDetailUpdate
{% endhighlight %}

I also added a type for teh editor state and added it to the ViewLocationPageModel like:

{% highlight FSharp %}
type UpdateLocationDetailState =
    {
        EditingSummary: bool
        NewSummaryContent: string
        EditingDescription: bool
        NewDescriptionContent: string
    }

type PageModel =
    | HomePageModel
    | FindPageModel of string * string * int
    | LogoutPageModel
    | YourAccountPageModel
    | EditLocationPageModel of string
    | ViewLocationPageModel of LocationDetailModel * UpdateLocationDetailState
    | AboutPageModel
    | NotFound
    | Unauthorized
    | LoadNextPage of BrowseFilterModel
    | LoadPreviousPage of BrowseFilterModel
{% endhighlight %}

I then added a default edit model generator (for initial state) and an update function:

{% highlight FSharp %}
let stringEmptyOrValue (s :  string option) =
    Option.fold (fun x y -> x) String.Empty s

let defaultEditState (model: LocationDetailModel) : UpdateLocationDetailState =
    {
        EditingSummary = false
        NewSummaryContent = stringEmptyOrValue model.Summary
        EditingDescription= false
        NewDescriptionContent= stringEmptyOrValue model.Description
    }

let updateLocationDetailsModel  (d: LocationDetailUpdate) (currPage : LocationDetailModel) (editState : UpdateLocationDetailState) =
    match d with
    | SummaryStartEdit -> currPage, { editState with EditingSummary = true }
    | SummaryTextChanged s->
        Console.WriteLine(s)
        currPage, { editState with NewSummaryContent = s }
    | SummaryTextSaved ->  { currPage with Summary = Some editState.NewSummaryContent}, { editState with EditingSummary = false }
    | SummaryTextChangeCanceled -> currPage, { editState with EditingSummary = false; NewSummaryContent = stringEmptyOrValue currPage.Summary }
    | DescriptionStartEdit -> currPage, { editState with EditingDescription = true }
    | DescriptionTextChanged s -> currPage, { editState with NewDescriptionContent = s }
    | DescriptionTextSaved -> { currPage with Description = Some editState.NewDescriptionContent}, { editState with EditingDescription = false }
    | DescriptionTextChangeCanceled -> currPage, { editState with EditingDescription = false; NewDescriptionContent = stringEmptyOrValue currPage.Description }
{% endhighlight %}

I updated the main update functions/ message handling function to handle these new functionality:

{% highlight FSharp %}
let update (msg: Msg) (model: Model) : Model * Cmd<Msg> =
    match msg, model.PageModel with
    | ReceivedLocationDetail d, _ -> modelWithNewPageModel model ( (d, Pages.LocationDetails.defaultEditState d) |> ViewLocationPageModel), Cmd.none
    | ReceivedBrowsePageResult d, _  -> modelWithNewPageModel model (d |> BrowsePageModel), Cmd.none
    | BrowsePageFilterChanged d, _  ->
        (model,
         (Cmd.OfAsync.perform
             locationInformationApi.searchLocations
             (mapBrowsePageFilterChangeToLocationSearchRequsst d)
             mapSearchResultToReceievedBrowsePageResult))
    | LocationDetailUpdated d, ViewLocationPageModel (currPage, currentEditState)  ->
       { model with PageModel = Pages.LocationDetails.updateLocationDetailsModel d currPage currentEditState |> ViewLocationPageModel }, Cmd.none
    | _, _  -> model, Cmd.none

let view (model: Model) (dispatch: Msg -> unit) =
    match model.PageModel with
    | HomePageModel -> homeView dispatch
    | LoadingScreenPageModel -> SharedComponents.loadingScreen
    | ViewLocationPageModel (d, e) -> locationDetailView d e dispatch
    | BrowsePageModel bpm -> browseView bpm dispatch
    | NotFound -> str "404"
    | _ -> str "???"
{% endhighlight %}

Finally, I update the view to publish these message and reflect the bool is editing value like:

{% highlight FSharp %}

    let rec editablePageSection (sectionTitle: string) previousValue boolEditingEnabled (editFunction: string -> unit) (saveChanges: unit -> unit) (cancelChanges:  unit -> unit) (startEditing:  unit -> unit)  = 
      match previousValue with
      | None -> editablePageSection sectionTitle (Some "") boolEditingEnabled editFunction saveChanges cancelChanges startEditing
      | Some s ->
        if not boolEditingEnabled then 
            seq {
                yield
                    Bulma.section [ prop.children [ Bulma.title [ prop.children [
                                                                        Html.span [prop.text sectionTitle];
                                                                        Html.i [ prop.className "fas fa-edit is-pulled-right"
                                                                                 prop.onClick (fun x -> startEditing ())]  ] ]
                                                    Html.p [ prop.innerHtml s ] ] ]
            }
        else
                 seq {
                            yield
                                Bulma.section [ prop.children [ Bulma.title [ prop.text sectionTitle  ]
                                                                Quill.editor [
                                                                    editor.toolbar [
                                                                        [ Header (ToolbarHeader.Dropdown [1..4]) ]
                                                                        [ ForegroundColor; BackgroundColor ]
                                                                        [ Bold; Italic; Underline; Strikethrough; Blockquote; Code ]
                                                                        [ OrderedList; UnorderedList; DecreaseIndent; IncreaseIndent; CodeBlock ]
                                                                    ]
                                                                    editor.defaultValue s
                                                                    editor.onTextChanged (fun x -> editFunction x)
                                                                ]
                                                                Bulma.field.div [
                                                                    Bulma.field.hasAddons
                                                                    prop.children [
                                                                        Bulma.control.p [
                                                                            Bulma.button.button [
                                                                                Bulma.color.isSuccess
                                                                                prop.onClick (fun x -> saveChanges ())
                                                                                prop.children [
                                                                                    Bulma.icon [ Html.i [ prop.className "fas fa-save" ] ]
                                                                                    Html.span [ prop.text "Save" ] ]
                                                                            ]
                                                                        ]
                                                                        Bulma.control.p [
                                                                            Bulma.button.button [
                                                                                Bulma.color.isWarning
                                                                                prop.onClick (fun x -> cancelChanges ())
                                                                                prop.children [
                                                                                    Bulma.icon [ Html.i [ prop.className "fas fa-trash" ] ]
                                                                                    Html.span [ prop.text "Cancel" ] ]
                                                                            ]
                                                                        ]
                                                                    ]
                                                                ] ] ]
                        }

    let pageContent (model: LocationDetailModel) (editState : UpdateLocationDetailState) (dispatch: LocationDetailUpdate -> unit)=
        [ Bulma.title [ title.is1
                        prop.className "mb-5"
                        prop.text model.Name ]
          Bulma.title [ title.is2
                        prop.className "has-text-grey "
                        prop.text model.SubTitle ]
          yield! editablePageSection "Summary" model.Summary editState.EditingSummary (fun x -> x |> SummaryTextChanged |> dispatch) (fun x -> SummaryTextSaved |> dispatch) (fun x -> SummaryTextChangeCanceled |> dispatch)  (fun x -> SummaryStartEdit |> dispatch)
          yield! editablePageSection "Description" model.Description editState.EditingDescription (fun x -> x |> DescriptionTextChanged |> dispatch) (fun x -> DescriptionTextSaved |> dispatch) (fun x -> DescriptionTextChangeCanceled |> dispatch)  (fun x -> DescriptionStartEdit |> dispatch)
          yield! addressSection model.Address
          yield! displayImages model.Images
          yield!
              modifyTextInParagraphOrYieldNothing
                  "has-text-grey mb-5"
                  (fun x -> "Description from: " + x)
                  model.DescriptionCitation
          Html.div [ prop.className "buttons"
                     prop.children [ Html.a [ prop.className "button is-primary"
                                              prop.href "https://github.com/jamesclancy/WilkensAvenue"
                                              prop.text "Find Directions" ]
                                     Html.a [ prop.className "button is-primary"
                                              prop.href "https://github.com/jamesclancy/WilkensAvenue"
                                              prop.text "Upload Images" ] ] ] ]
{% endhighlight %}

This needs to be refactored and actually hooked up to the backend but I am ending the step here. 

## Results


[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-15)