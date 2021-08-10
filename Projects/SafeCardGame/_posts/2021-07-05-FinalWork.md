---
layout: post
title: SafeCardGame - Final Work
author: James Clancy
tags: fsharp dotnet
---

The majority of this project is done and isn't really going anywhere exciting anymore. I am going to touch up some of the game ot make it a little more playable and probably stop working on it.

## Moving Stuff around and making the game a little more playable

A number of times during this process I have encountered stupid issues which could have been avoided had a spent a little time refactoring & writing tests. I didn't do this as I didn't think this would be a complicated or time consuming of a project. Since the game psuedo works now I am going to to take the time to do some house keeping.

I added some authentication via Oauth with Github.

Additionally, I added user secrets and configuration registration. I used Rider spell check to fix a number of spelling mistakes and reformat a number of things.

Apparently, string interpolation does not work with fable/react and completely broke the site. I migrated everything from sprintf via the refactor and then had to manually redo it all which was endless fun.

I am trying to add a user management controller now. I tried using saturn cli. This conflicted with the safe template so I created it in a different repo and am trying to copy it over.



[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-19-improvements-to-waiting-and-screens)