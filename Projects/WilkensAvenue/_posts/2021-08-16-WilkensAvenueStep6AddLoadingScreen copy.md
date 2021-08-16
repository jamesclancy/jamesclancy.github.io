---
layout: post
title: Wilkens Avenue - Step 6 - Add a loading screen for when waiting for network calls to complete
author: James Clancy
tags: fsharp dotnet safe-stack ci test github github-actions
---

After working on Step 5 and starting building out one of the pages I noticed that when you initially load an api response reliant page it loads the home page first and then changes pages. This is not an ideal user experience. To help improve this I am adding a loading screen to display as the api information is fetched over the network.


## Goals of this Step
* Added loading screen
* On load the the the init state of the app is pointing to the loading screen.
* When waiting for the location detail screen it should show the loading screen

## Process

Per https://fulma.github.io/Fulma/#fulma-extensions/pageloader

I ran

```
paket add Fulma.Extensions.Wikiki.PageLoader --project .\src\Client\Client.fsproj
dotnet tool install femto
dotnet femto .\src\Client\Client.fsproj
```

Then I followed the provided instructions running 

```
npm uninstall react
npm install react@17.0.2 --save
npm uninstall react-dom
npm install react-dom@17.0.2 --save
npm install bulma@0.9.3 --save
npm install bulma-pageloader@2.1.0 --save
```

Additionally, (and not documented anywhere I saw) I had to add a reference to the CSS package for the bulma loader. to do this I added `<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma-pageloader@0.3.0/dist/css/bulma-pageloader.min.css" />` to my index.html.

## Results

Currently, the page can be viewed [here](https://wilkens-avenue.herokuapp.com/#/viewlocation/sdf).



[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-6)