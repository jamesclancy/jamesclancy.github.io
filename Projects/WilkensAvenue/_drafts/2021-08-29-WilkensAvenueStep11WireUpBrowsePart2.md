---
layout: post
title: Wilkens Avenue - Step 10 - Wire up Browse Page - Part 2 Write
author: James Clancy
tags: fsharp dotnet safe-stack bulma-tagsinput mocha fails
---

## Goals of this Step
* Browse Page vie model includes all data to render the page
* Browse page receives model from update and inturn api and renders the page

## Process

For now I am going to move on from the tags input issue and come back to it later. I think I have identified some possible solutions but I think I am using to new a version of webpack to utilize them. 

This required the implementation of a new API endpoint, several mappings between `DataTransferFormats`, modifications to the page views to pass through the values, updates to the url router to fire off the API call as well as a new Msg type to handle completed search requests.

### New Api Endpoint

### DataTransferFormat Mappings

### Page View Modifications

### Updates to URL Router

### New Msg Type & Handler


## Results

Currently, the page can be viewed [here](https://wilkens-avenue.herokuapp.com/#/browse/12312).



[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-10)