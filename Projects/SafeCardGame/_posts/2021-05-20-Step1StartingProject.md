---
layout: post
title: SafeCardGame - Step 1 Starting Project
author: James Clancy
tags: fsharp dotnet
---

# Creating a Card Game with the Safe Stack

I have become interested in playing around with the `SAFE` Stack and have decided that it would be interesting to create an online “Card Game” using the stack. This card game should largely be similar to a simplified version of *MTG* or *Pokemon TCG*.

## Starting Project ##

Created folder for project

```
 mkdir SafeCardGame
 cd SafeCardGame
 ```

 Installed and created project from a template
```
dotnet new -i SAFE.Template
dotnet new SAFE
```

```
dotnet tool restore
dotnet fake build --target run
```

This should scaffold a basic todo app. Progress to here is located in the branch `step-1-intial-from-temp`

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-1-intial-from-temp)