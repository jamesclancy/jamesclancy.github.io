---
layout: post
title: Wilkens Avenue - Step 17 - Add Map to Location Details
author: James Clancy
tags: fsharp dotnet safe-stack browse-page
---

## Goals of this Step
    * Location details should display a map of where the historical site it located
    * this location can be determined by the latitude and longitude already displayed on the page

## Process

It looks like [Feliz.PigeonMaps](https://zaid-ajaj.github.io/Feliz/#/PigeonMaps/Overview) would be a good & easy option to add a map. 

I added `"pigeon-maps": "^0.19.7"` to the packages.json and added `Feliz.PigeonMaps` to the general project and client `paket.reference` files. 

I then ran `npm ci` and `paket install`.

Verifying, `dotnet run` still works. Based on the PigeonMaps documentation I tested adding to the `LocationDetails`

{% highlight FSharp %}

    let addressSection (address: AddressDetailModel option) =

        let tryReturnString (optStr: string option) =
            match optStr with
            | None -> Seq.empty
            | Some s -> seq { yield Html.span s }

        match address with
        | None -> Seq.empty
        | Some a ->
            seq {
                yield
                    Bulma.section [ Bulma.title "Location"
                                    Html.p [ prop.children [ yield! tryReturnString a.Address1
                                                             yield! tryReturnString a.Address2
                                                             yield! tryReturnString a.Address3
                                                             yield! tryReturnString a.City
                                                             yield! tryReturnString a.State
                                                             yield! tryReturnString a.Zipcode
                                                             yield PigeonMaps.map [
                                                                 map.center(System.Decimal.ToDouble(a.Latitude), System.Decimal.ToDouble(a.Longitude))
                                                                 map.zoom 12
                                                                 map.height 350
                                                                 map.markers [
                                                                     PigeonMaps.marker [
                                                                         marker.anchor(System.Decimal.ToDouble(a.Latitude), System.Decimal.ToDouble(a.Longitude))
                                                                         marker.offsetLeft 15
                                                                         marker.offsetTop 30
                                                                         marker.render (fun marker -> [
                                                                             Html.i [
                                                                                 if marker.hovered
                                                                                 then prop.style [ style.color.red; style.cursor.pointer ]
                                                                                 prop.className [ "fa"; "fa-map-marker"; "fa-2x" ]
                                                                             ]
                                                                         ])
                                                                     ]
                                                                 ]
                                                             ]
                                                             yield Html.span (sprintf "Latitude: %f" a.Latitude)
                                                             yield Html.span (sprintf "Longitude: %f" a.Longitude) ] ] ]
{% endhighlight %}

This did not work and is giving me an error `error FABLE: System.Decimal.ToDouble is not supported by Fable`. Changing the `System.Decimal.ToDouble` to `float` appears to fix the issue. 

To make the address display a little better I wanted to add a popover for the marker. To do this I added `Feliz.Bulma.Popover` via paket and added `bulma-popover` via npm. 

Reading the [MapQuest api docs](https://developer.mapquest.com/documentation/tools/link-to-mapquest/) I could generate links to find directions by doing `sprintf "https://mapquest.com/latlng/%f,%f" a.Latitude a.Longitude`.

I was then able to render the information like:

{% highlight FSharp %}
    let addressSection (address: AddressDetailModel option) =

        let tryReturnString (optStr: string option) =
            match optStr with
            | None -> Seq.empty
            | Some s -> seq { yield Html.span s }

        let stringOrEmpty (optStr: string option) =
            match optStr with
            | None -> String.Empty
            | Some s -> s

        let formattedAddress a =
           [ Html.p [ prop.children [
                     yield! tryReturnString a.Address1
                     yield! tryReturnString a.Address2
                     yield! tryReturnString a.Address3
                 ] ]
             Html.p [ prop.text  (sprintf "%s, %s, %s" (stringOrEmpty a.City) (stringOrEmpty a.State) (stringOrEmpty a.Zipcode))
             ] 
             Html.p [ prop.text (sprintf "LongLat: (%.2f, %.2f)" a.Longitude a.Latitude) ] ]

        match address with
        | None -> Seq.empty
        | Some a ->
            seq {
                yield
                    Bulma.section [ Bulma.title "Location"
                                    Html.p [ prop.children [ yield PigeonMaps.map [
                                                                 map.center((float a.Latitude) + (float 0.003), (float a.Longitude) + (float 0.005))
                                                                 map.zoom 15
                                                                 map.height 350
                                                                 map.markers [
                                                                     PigeonMaps.marker [
                                                                         marker.anchor(float a.Latitude, float a.Longitude)
                                                                         marker.offsetLeft 10
                                                                         marker.offsetTop 50
                                                                         marker.render (fun marker -> [
                                                                            Popover.popover [
                                                                                Html.a [
                                                                                    prop.children [
                                                                                        Html.i [
                                                                                            if marker.hovered
                                                                                            then prop.style [ style.color.red; style.cursor.pointer ]
                                                                                            prop.className [ "fa"; "fa-map-marker"; "fa-2x" ]
                                                                                        ] ]
                                                                                    popover.trigger
                                                                                    prop.href (sprintf "https://mapquest.com/latlng/%f,%f" a.Latitude a.Longitude)
                                                                                    prop.target "_blank"
                                                                                ]
                                                                                Popover.content [ Bulma.card [
                                                                                    Bulma.cardHeader [
                                                                                        Bulma.cardHeaderTitle.p model.Name
                                                                                    ]
                                                                                    Bulma.cardContent [
                                                                                        Bulma.content (formattedAddress a)
                                                                                    ]
                                                                                ] ] ]
                                                                         ] )
                                                                     ]
                                                                 ]
                                                             ] ] ]
                                    yield! formattedAddress a
                                    Html.p [ Html.a [
                                        prop.text "Find Directions"
                                        popover.trigger
                                        prop.href (sprintf "https://mapquest.com/latlng/%f,%f" a.Latitude a.Longitude)
                                        prop.target "_blank"
                                    ]  ] ]
            }
{% endhighlight %}

This now appears to work (albeit with a clunky UI). One this I did notice is the the top menu does not work with small screens. 

To fix this I added an id of "nav-menu" to the navbar menus and added a burger under the menu brand like:


{% highlight FSharp %}
let navBar (dispatch: Msg -> unit) =

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

                                                                                      Bulma.navbarBurger [
                                                                                         prop.custom ("data-target", "nav-menu")
                                                                                         prop.children [
                                                                                             Html.span [ ]
                                                                                             Html.span [ ]
                                                                                             Html.span [ ]
                                                                                         ]
                                                                                     ]]
                                                              
                                                              Bulma.navbarMenu [ prop.id "nav-menu"
                                                                                 prop.children [
{% endhighlight %}

I then copied the provided javascript to make the burger work from [Bulma site](https://bulma.io/documentation/components/navbar/) to the index.html. This is a very messy solution but functions for now. 



## Results

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-17)