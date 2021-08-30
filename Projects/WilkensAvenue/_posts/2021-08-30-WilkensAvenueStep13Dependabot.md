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

Dependabot is a free Github action which scans projects for old dependencies and created pull requests for upgrades to the dependencies. 

I created a basic dependabot.yml file based on the template in the .github folder of the repository. I am not certain how well this will work with Packet/F# but I figured I would see what happens. 

the file looked like

```
version: 2
updates:
  - package-ecosystem: "npm" # See documentation for possible values
    directory: "/" # Location of package manifests
    schedule:
      interval: "daily"
  - package-ecosystem: "nuget" # See documentation for possible values
    directory: "/" # Location of package manifests
    schedule:
      interval: "daily"
  - package-ecosystem: "docker" # See documentation for possible values
    directory: "/" # Location of package manifests
    schedule:
      interval: "daily"
  - package-ecosystem: "github-actions"
    # Workflow files stored in the
    # default location of `.github/workflows`
    directory: "/"
    schedule:
      interval: "daily"
```

One thing that is strange with the results. For nuget, it appears results are generated for all projects in the solution but they are grouped under the name `Build.fsproj`.

![Dependabot Build .fsproj Screenshot](\assets\img\post-media\2021-08-30-WilkensAvenueStep13Dependabot\DependabotResultsScreenshot.png)


## Results

Dependabot found 5 Javascript issues. I merged two which appeared to be very safe. I will have to evaluate if the others are alright to do in the future. 

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-13)