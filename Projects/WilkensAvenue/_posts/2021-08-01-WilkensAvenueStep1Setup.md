---
layout: post
title: Wilkens Avenue - Step 1 - Setup
author: James Clancy
tags: fsharp dotnet safe-stack heroku
---

Similar to the SafeCardGame, I am going to utilize the `SAFE Template` to scaffold the initial site. This time though before making any code changes I am going to set up the Heroku setup and database configuration along with the appropriate configuration for test and scripts to publish. Previously, I delayed that and it was quite a pain point to do after the fact. Luckily, this time I am using a newer version of the `SAFE` template which rolls with .NET 5 out of the box. In teh `SafeCardGame` project I manually upgraded that which turned out to be quite a pain.

## Goals of this Step
* Create the project via the template
* Create the Repo on Github
* Create the setup for hosting
* Create the docker files etc for the container
* Have a script to build and push container to Heroku
* Add some basic database access via Dapper to the project, locally it should pull connection information from the App Secrets, when deployed it should use the Environmental Variables