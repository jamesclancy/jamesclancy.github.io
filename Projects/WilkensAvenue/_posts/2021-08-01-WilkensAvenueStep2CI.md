---
layout: post
title: Wilkens Avenue - Step 2 - Add CI Test Runner
author: James Clancy
tags: fsharp dotnet safe-stack ci test github github-actions
---

Unlike what I did during the development of the SafeCardGame, I am going to prioritize tests when creating this project. In order to better facilitate this I am going to create a github action to automatically run tests on PRs to master.

## Goals of this Step
* Add github action to build project and run all server and client tests when a PR is created to master.

## Issues encountered
By default the client tests in the safe template run in a headed web browser. This was problematic from a action standpoint. I end ups have to spend a good amount of time to googling about it. I eventually looked read the [mocha documentation](https://github.com/Zaid-Ajaj/Fable.Mocha#running-the-tests-on-nodejs-with-mocha). 

Installing mocha & fable-splitter to npm and adding the 

```
"pretest": "fable-splitter tests/Client -o nodetests/ --commonjs",
"test": "mocha nodetests"
```

to the package.json allowed me to add an action running `npm run test` to test the client. 

This ran locally but not inside the action. I also had to add the `fable-compiler` in the packages. I am assuming I must have the fable-compiler installed globally on my local box but it obviously isn't on the action runner.

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-2)