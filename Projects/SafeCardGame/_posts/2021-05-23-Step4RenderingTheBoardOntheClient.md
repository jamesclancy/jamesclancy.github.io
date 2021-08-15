---
layout: post
title: SafeCardGame - Step 4 - Game Board Layout
author: James Clancy
tags: fsharp dotnet
---

### Bringing the board layout into the client
---

I feel like the next logical step is to bring the layout I just developed to the client.

First I updated the index.html file in the src/client to reference the bulmaswatch layout "darkly" like the template.

I then googled around and found an online tool to convert html to the format used by elmish to define html. This is usable and somewhat close to the Fulma format we would like to eventually support. ([HTML to Elmish Site] (https://mangelmaxime.github.io/html-to-elmish/))

Whatever we convert is going to be plugged into the <div id="elmish-app"></div> in the index.html so I needed to extract the content of the body of the layout from the body. I saved it separately: ([template to convert](documentation/html/InitialSketchOfBoardv2_toConvert.html))

This failed with no error message on the site so I am assuming it doesn't like the html somewhere.

In order to move forward, I began breaking down the html page into pieces and converting the page part by part (which is something that would eventually have to be done anyway).

This will require future refinement but I initially broke out the page into parts for the

```
    topNavigation
    enemyStats
    enemyCreatures
    playerControlCenter
    playerCreatures
    playerHand
    footerBand
```
With a general arrangement of
```
<div class="container is-fluid">
    topNavigation
    <br /><br />
    enemyStats
    enemyCreatures
    <section class="section">
        <div class="container py-r">
            playerControlCenter
            playerCreatures
        </div>
    </section>
    playerHand
    footerBand
</div>
```

Going in order I dropped each of these parts into the converter. When I reached `playerControlCenter` the converter stopped working. After playing around with the html for a while (no error message was returned) I was able to discover that the converter did not like the `disabled` on the buttons.

i.e. the blocks like:
```
<button class="button is-primary" disabled>
    <span>Draw</span>
</button>
```
needed to be changed to

```
<button class="button is-primary" disabled="true">
    <span>Draw</span>
</button>
```

I don't think this is needed from an html standpoint? It is probably just a feature of the converter.


I was eventually able to convert all the html subsections defining each as a variable (will eventually be converted to a function).

i.e. like:
{% highlight FSharp %}
let footerBand =
  footer [ Class "footer" ]
    [ div [ Class "container" ]
        [ div [ Class "level" ]
        ...
{% endhighlight %}

I then manually created the mainLayout as

{% highlight FSharp %}
let mainLayout =
  div [ Class "container is-fluid" ]
    [ topNavigation
      br [ ]
      br [ ]
      enemyStats
      enemyCreatures
      section [ Class "section" ]
        [ div [ Class "container py-r" ]
            playerControlCenter
            playerCreatures ] ]
      playerHand
      footerBand
{% endhighlight %}
The results of this process ae located in ([the documentation > html > ElmishElements.fs](documentation/html/ElmishElements.fs))


I then added a `PageLayoutElements.fs` to the src/client and added the file into the `Client.fsproj` to be included and compiled coming before the `Index.fs`.

I copied and pasted the contents from the ElmishElements into this file.

This broke the build as the file needed a module to be defined and the references imported so I added

{% highlight FSharp %}
module PageLayoutParts

open Elmish
open Fable.React
open Fable.React.Props
open Fulma
{% endhighlight %}

to the top of the file.


This still did not build. There was an issue with a `broken-css` being added by the converter for the background-image element. As well as an issue with the way `strong` elements were converted.

The indentation was

{% highlight FSharp %}

                                [ strong [ ]
                                    [ str "On Enter Playing Field" ]
                                      str ": Effect description." ]
{% endhighlight %}

I needed to change it to

{% highlight FSharp %}

                                [ strong [ ]
                                    [ str "On Enter Playing Field" ]
                                  str ": Effect description." ]
{% endhighlight %}

i.e. the function definition is strong str not strong and two lists like many other elements.

Finally, I found that the `mainLayout` thing I wrote was invalid. I had to change it to:

{% highlight FSharp %}
let mainLayout =
  div [ Class "container is-fluid" ]
    [ topNavigation
      br [ ]
      br [ ]
      enemyStats
      enemyCreatures
      section [ Class "section" ]
        [ div [ Class "container py-r" ]
            [
            playerControlCenter
            playerCreatures
            ]
        ]
      playerHand
      footerBand
    ]
{% endhighlight %}

It compiled and I was then able to upload the view function in Index.fs to reference the mainLayout function.

i.e.

{% highlight FSharp %}
let view (model : Model) (dispatch : Msg -> unit) =
    PageLayoutParts.mainLayout
{% endhighlight %}

After viewing the page a few elements had to be rearranged and some changes had to be made. For example, changing the html unicode codes to just the unicode characters.

For the next step, we will try to make the output actually pull information from the GameState model and break down these general page parts into smaller and more usable parts.

This is the final commit in the `step-4-implement-layout-in-client` branch.

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-4-implement-layout-in-client)