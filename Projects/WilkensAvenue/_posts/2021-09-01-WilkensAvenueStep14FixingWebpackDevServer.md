---
layout: post
title: Wilkens Avenue - Step 14 - Fixing a number of the Dependabot Upgrades
author: James Clancy
tags: fsharp dotnet safe-stack browse-page
---

## Goals of this Step
* Update webpack-cli
* Update webpack-dev-server

## Process

The Dependabot flagged upgrades for `webpack-cli` and `webpack-dev-server`. These are important and large upgrades which introduced breaking changes (even though Dependabot said dev server was 100% compatible.)

First I updated the packages.json file to later version of these two libraries like:

```
        "webpack-cli": "^4.8.0",
        "webpack-dev-server": "^4.0.0"
```

This caused `dotnet run` to fail catastrophically when trying to start the client. 

To more quickly troubleshoot the problems I switched to running `npx webpack-dev-server --config webpack.config.js`

This gave me a number of issues to step through with the configuration:
```
PS C:\tests\WilkensAvenue> npx webpack-dev-server --config webpack.config.js
[webpack-cli] Invalid options object. Dev Server has been initialized using an options object that does not match the API schema.
 - options has an unknown property 'inline'. These properties are valid:
   object { allowedHosts?, bonjour?, client?, compress?, devMiddleware?, headers?, historyApiFallback?, host?, hot?, http2?, https?, ipc?, liveReload?, magicHtml?, onAfterSetupMiddleware?, onBeforeSetupMiddleware?, onListening?, open?, port?, proxy?, setupExitSignals?, static?, watchFiles?, webSocketServer? }
PS C:\tests\WilkensAvenue> npx webpack-dev-server --config webpack.config.js
Bundling for development...
[webpack-cli] Invalid options object. Dev Server has been initialized using an options object that does not match the API schema.
 - options has an unknown property 'contentBase'. These properties are valid:
   object { allowedHosts?, bonjour?, client?, compress?, devMiddleware?, headers?, historyApiFallback?, host?, hot?, http2?, https?, ipc?, liveReload?, magicHtml?, onAfterSetupMiddleware?, onBeforeSetupMiddleware?, onListening?, open?, port?, proxy?, setupExitSignals?, static?, watchFiles?, webSocketServer? }
PS C:\tests\WilkensAvenue> npx webpack-dev-server --config webpack.config.js
Bundling for development...
[webpack-cli] Invalid options object. Dev Server has been initialized using an options object that does not match the API schema.
 - options has an unknown property 'publicPath'. These properties are valid:
   object { allowedHosts?, bonjour?, client?, compress?, devMiddleware?, headers?, historyApiFallback?, host?, hot?, http2?, https?, ipc?, liveReload?, magicHtml?, onAfterSetupMiddleware?, onBeforeSetupMiddleware?, onListening?, open?, port?, proxy?, setupExitSignals?, static?, watchFiles?, webSocketServer? }
```

* I removed the `inline` property from `module.exports.devServer` in `webpack.config.js`
* I switched `contentBase` to `static` in  `module.exports.devServer` in `webpack.config.js`
* I removed `publicPath`  from `module.exports.devServer` in `webpack.config.js` 

Now `npx webpack-dev-server --config webpack.config.js` is working but `dotnet run` still fails with a:

```
client: WARNING in configuration
client: The 'mode' option has not been set, webpack will fallback to 'production' for this value. Set 'mode' option to 'development' or 'production' to enable defaults for each environment.
client: You can also set it to 'none' to disable any default behavior. Learn more: https://webpack.js.org/configuration/mode/
client: ERROR in Entry module not found: Error: Can't resolve './src' in 'C:\tests\WilkensAvenue\src\Client'
client: ERROR in multi (webpack)-dev-server/client?protocol=ws%3A&hostname=0.0.0.0&port=8080&pathname=%2Fws&logging=info (webpack)/hot/dev-server.js ./src
client: Module not found: Error: Can't resolve './src' in 'C:\tests\WilkensAvenue\src\Client'
client:  @ multi (webpack)-dev-server/client?protocol=ws%3A&hostname=0.0.0.0&port=8080&pathname=%2Fws&logging=info (webpack)/hot/dev-server.js ./src main[2]
```

This appears to be because the dev server is no longer pulling in the correct configuration by default. To remedy this I update the `Build.fs` file changing the run definition from: 
```
Target.create "Run" (fun _ ->
    run dotnet "build" sharedPath
    [ "server", dotnet "watch run" serverPath
      "client", dotnet "fable watch --run webpack-dev-server" clientPath ]
    |> runParallel
)
```

to 

```
Target.create "Run" (fun _ ->
    run dotnet "build" sharedPath
    [ "server", dotnet "watch run" serverPath
      "client", dotnet "fable watch --run webpack-dev-server --config ../../webpack.config.js" clientPath ]
    |> runParallel
)
```

and `dotnet run` now works again.

Now it looks like dotnet run Bundle fails with 
```
client: .> cmd /C ..\..\node_modules\.bin\webpack -p
client: [webpack-cli] Error: Unknown option '-p'
client: [webpack-cli] Run 'webpack --help' to see available commands and options
client: Run process failed
Finished (Failed) 'Bundle' in 00:00:21.3183767
```

The `Bundle` was defined as 

```
Target.create "Bundle" (fun _ ->
    [ "server", dotnet $"publish -c Release -o \"{deployPath}\"" serverPath
      "client", dotnet "fable --run webpack -p" clientPath ]
    |> runParallel
)
```

To fix the command I had to switch out the `-p` with ` --mode production --env production` and also explicitly specify the configuration with a `--config ../../webpack.config.js`. The led to the `Bundle` being replaced with:

```
Target.create "Bundle" (fun _ ->
    [ "server", dotnet $"publish -c Release -o \"{deployPath}\"" serverPath
      "client", dotnet "fable --run webpack --mode production --env production --config ../../webpack.config.js" clientPath ]
    |> runParallel
)
```

Now `dotnet run Bundle` works.

## Results

[Git branch for this step](https://github.com/jamesclancy/WilkensAvenue/tree/step-14)