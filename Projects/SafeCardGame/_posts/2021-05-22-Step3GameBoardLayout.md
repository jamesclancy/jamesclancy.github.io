---
layout: post
title: SafeCardGame - Step 3 - Game Board Layout
author: James Clancy
tags: fsharp dotnet
---

### Defining and Drawing the Gameboard
---

My idea for the next step is to lay out the general game board.
This serves a great base to force me to actually think out the domain, show visible progress and have a base to actually plug future developments into and visualize.

I first made a sketch of what I was thinking about for the board.

![Basic Sketch](documentation/img/InitialSketchOfBoard.png)

I used shuffle.dev to scaffold a basic ui using Bulma ([editor avail here](https://shuffle.dev/editor?project=10d217c2a045a04ac447cd87a95c49662d831217)). Fulma, a F# strongly typed wrapper for Bulma is baked into the Safe Stack which will be utilized later on.

From my initial sketch, I scaffolded out some stuff using shuffle, downloaded the HTML, and modified the HTML to create a basic template based on my sketch.

([Bulma template for the board](documentation/html/InitialSketchOfBoard.html))

I moved things around a number of times, I was unable to make it all fit on one page (the goal) but I think I was able to improve it somewhat.

([Bulma template for the board v2](documentation/html/InitialSketchOfBoardv2.html))

I am going to move forward with this layout. In the future, I am thinking the user could toggle something to transform the cards into tables or something along those lines.

This is the final commit in the `step-3-build-the-game-board-basic-layout` branch.

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-3-build-the-game-board-basic-layout)