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


## Results


[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-15)