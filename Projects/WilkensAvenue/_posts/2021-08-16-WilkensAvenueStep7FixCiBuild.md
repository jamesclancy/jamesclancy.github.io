---
layout: post
title: Wilkens Avenue - Step 7 - Fixing the CI Build
author: James Clancy
tags: fsharp dotnet safe-stack ci test github github-actions
---

Locally, dotnet run bundle was working but on the CI it was failing on build.


## Goals of this Step
* Fix CI Build

## Process

It appears that the previous npm installs from the last step did not properly set up the packages/ packages.lock.

I reran

```
npm install react@17.0.2 --save
npm install react-dom@17.0.2 --save
npm install bulma@0.9.3 --save
npm install bulma-pageloader@2.1.0 --save

npm i 
```

Pushing that up the build now works.



[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-6)