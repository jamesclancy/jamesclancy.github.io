---
layout: post
title: Wilkens Avenue - Step 13 - Add Dependabot 
author: James Clancy
tags: fsharp dotnet safe-stack browse-page
---

## Goals of this Step
* Add Dependabot configuration to the repository
* Evaluate the Dependabot results.

## Process

I created a basic dependabot.yml file based on the template in the .github folder of the repository. I am not certain how well this will work with Packet/F# but I figured I would see what happens. 

## Results

Dependabot found 5 Javascript issues. I merged two which appeared to be very safe. I will have to evaluate if the others are alright to do in the future. 

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-13)