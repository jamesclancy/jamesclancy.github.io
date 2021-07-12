---
layout: post
title: SafeCardGame - Project Retrospective
author: James Clancy
tags: fsharp dotnet
---

## Overview
I came up with the idea that implementing a *Wingspan*-like combo-based card game using the safe stack. I thought this would be an interesting project as each card would have to have uniquely defined effects which would apply to the game state in a stack and provide me with a great opportunity to learn more about F# and the safe stack in general. Upon further thought I decided that a Wingspan-like game might be to complicated as a first real SafeStack project and decided to make a game sort of along the lines of Pokemon TCG or MTG which would be easier to iteratively test. Basically, this type of game would be more "playable" when it isn't complete. I don't think a game based on combos would be that playable without an extensive number of special effect card defined. 

## Process and Reflection
The process I went through is documented [here](https://github.com/jamesclancy/SafeCardGame/blob/main/README.md). Overall, I sort of lost interest and stopped working on it but it does pseudo-work. The end result is code base which I am not proud of but I did find the process of creating it relatively rewarding and I believe that it both exposed me to a number of new f# libraries while giving me some incites into what I should not do in future projects. One of the largest issues is how chatty the client-server, in teh initial development everything was running locally in the browser, using Elmish Bridge I then started piping these messages to the server. The fact that I could do this so easily was/is very cool but the end result leaves much to be desired. Additionally, I continuously faced stupid issues which could have been easily avoided if I had started writing unit tests at the beginning of the process. I sort of assumed this was going to be really simple and it wasn't worth the time to really test the domain logic. That turned out to the incorrect.

## Links

* [Visit the Repository](https://github.com/jamesclancy/SafeCardGame)
* [View Live](https://testing-demo-card-game.herokuapp.com/)