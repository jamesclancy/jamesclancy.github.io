---
layout: post
title: Wilkens Avenue - Step 5 - Start Scaffolding Page Layouts
author: James Clancy
tags: fsharp dotnet safe-stack ci test github github-actions
---

I am thinking that sketching out some layouts of the various pages from Step 3 might help scope out the domain.


## Goals of this Step
* Sketch of ViewLocation

## Process

First I pulled out the navBar into a separate `SharedComponents` module.

I eventually pulled out a number of view components and outlined the general view model and view for the "View Location" page. It doesn't look very good and returns static information from wikipedia but I think it illustrates the needed information. 

The location detail view turned out like:

{% highlight FSharp %}

let locationDetailView model (dispatch: Msg -> unit) =

    let addressSection (address : AddressDetailModel option) =

        let tryReturnString optStr =
            match optStr with
            | None -> Seq.empty
            | Some s -> seq { yield str s }

        match address with
        | None -> Seq.empty
        | Some a -> seq { yield section [ Class "section" ]
                                [
                                    h1 [ Class "title" ] [ str "Location" ]
                                    p [] [
                                            yield! tryReturnString a.Address1
                                            yield! tryReturnString a.Address2
                                            yield! tryReturnString a.Address3
                                            yield! tryReturnString a.City
                                            yield! tryReturnString a.State
                                            yield! tryReturnString a.Zipcode

                                            yield str (sprintf "Latitude: %f" a.Latitude)
                                            yield str (sprintf "Longitude: %f" a.Longitude)
                                         ]
                                    ] }
            
    let displayImages images =
        match images with
        | None -> Seq.empty
        | Some x ->
            seq { yield section [ Class "section" ]
                        [
                            yield h1 [ Class "title" ] [ str "Images" ]
                            yield! buildListOfThumbnails x
                            yield str "Upload your own photos!"
                            ]  }

    section [ Class "section pt-0 is-relative" ]
      [
        yield! (leftHalfPageImageRotation model.Images)
        (navBar dispatch)
        div [ Class "container" ]
              [ div [ Class "pt-5 columns is-multiline" ]
                  [ div [ Class "column is-12 is-5-desktop ml-auto" ]
                      [ div [ Class "mb-5" ]
                          [ h2 [ Class "mb-5 is-size-1 is-size-3-mobile has-text-weight-bold" ]
                              [ str model.Name ]
                            p [ Class "subtitle has-text-grey mb-5" ]
                              [ str model.SubTitle ]
                            yield! sectionOrYieldNothing "" "Summary" model.Summary
                            yield! sectionOrYieldNothing "" "Description" model.Description
                            yield! addressSection model.Address
                            yield! displayImages model.Images
                            yield! modifyTextInParagraphOrYieldNothing "has-text-grey mb-5" (fun x -> "Description from: " + x) model.DescriptionCitation
                            div [ Class "buttons" ]
                              [ a [ Class "button is-primary"
                                    Href "https://github.com/jamesclancy/WilkensAvenue" ]
                                  [ str "Directions" ]
                                a [ Class "button is-primary"
                                    Href "https://github.com/jamesclancy/WilkensAvenue" ]
                                  [ str "Upload Images" ]
                                a [ Class "button is-primary"
                                    Href "https://github.com/jamesclancy/WilkensAvenue" ]
                                  [ str "Submit Update to Description" ]
                                  ] ] ] ] ] ]


{% endhighlight %}

With a view model like:

{% highlight FSharp %}

type AddressDetailModel =
    {
         Address1: Option<string>
         Address2: Option<string>
         Address3: Option<string>
         City: Option<string>
         State: Option<string>
         Zipcode: Option<string>

         Longitude: decimal
         Latitude: decimal  
    }

type DetailImageModel =
    {
        Id: string
        Name: string

        Order: int

        IsCurrentlySelected: bool

        FullSizeUrl: string
        ThumbnailUrl: string

        FullSizeHeight: int
        FullSizeWidth: int
        ThumbnailHeight: int
        ThumbnailWidth: int

        Author: Option<string>
        Description: Option<string>
        Copyright: Option<string>
        MoreInforamtionLink: Option<string>


        IsOwnedByCurrentUser: bool
        SubmissionDate: DateTimeOffset
        Submitter: string
        SubmitterId: string
    }

type LocationDetailModel =
    {
        Id: string
        Name: string
        SubTitle: string

        Summary: Option<string>
        Description: Option<string>
        DescriptionCitation: Option<string>

        Address: Option<AddressDetailModel>

        Images: DetailImageModel list option
    }  

{% endhighlight %}


## Results

Currently, the page can be viewed [here](https://wilkens-avenue.herokuapp.com/#/viewlocation/sdf).



[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-5)