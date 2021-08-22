---
layout: post
title: Wilkens Avenue - Step 9 - Investigate why test is failing to start 
author: James Clancy
tags: fsharp dotnet safe-stack bulma-tagsinput mocha fails
---

## Goals of this Step
* get npm test to work

## Process

I spend an extensive amount of time trying to troubleshoot this. I believe the issue is `"@creativebulma/bulma-tagsinput"` is incorrectly registering its dependencies on react causing mocha to blow up. 

In my research it looks like the proper thing to do is to update the library. I do not think this is possible. While looking into this I migrated all the html to utilize Feliz and refactored some of the html. 

I will have to continue investigating this issues but since it appears to only effect mocha (webpack must be doing something to correct) I am pushing the migration to Feliz up as step-9.



## Results

Currently, the page can be viewed [here](https://wilkens-avenue.herokuapp.com/#/browse/12312).



[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-9)