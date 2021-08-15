---
layout: post
title: SafeCardGame - Step 10 - Adding more realistic cards data and wiring up the Ui to send some events -Continued - the Player Control Center
author: James Clancy
tags: fsharp dotnet
---

## Adding more realistic cards data and wiring up the Ui to send some events -Continued - the Player Control Center

Now I will try to implement the turn step changes via conditional buttons in the player control center.

Currently, a placeholder for these buttons is located in the `playerControlCenter` function. I will pull out a `stepNavigation` function and rewrite the `playerControlCenter` as

{% highlight FSharp %}
let playerControlCenter  (player : Player) playerBoard gameState dispatch =
  nav [ Class "navbar is-fullwidth is-primary" ]
    [ div [ Class "container" ]
        [ div [ Class "navbar-brand" ]
            [ p [ Class "navbar-item title is-5"
                  Href "#" ]
                [ str player.Name ] ]
          div [ Class "navbar-menu" ]
            [ div [ Class "navbar-start" ]
                [ yield! playerStats player playerBoard
                  currentStepInformation player gameState ]
              div [ Class "navbar-end" ]
                [ stepNavigation player playerBoard gameState dispatch ] ] ] ]
{% endhighlight %}
* note I also added the dispatch to the function

This `stepNavigation` function is going to rely on switching the current step and player to tell what options & actions are available.

I can implement the changes like:
{% highlight FSharp %}

let renderStepOptionButton dispatch buttonText classType message =
                        p [ Class "control" ]
                            [ button [
                                Class (sprintf "button %s is-large" classType)
                                OnClick  (fun _-> (message |>  dispatch))
                                ]
                                [ span [ ]
                                    [ str buttonText ] ] ]

let stepNavigation  (player: Player) (playerBoard: PlayerBoard) (gameState : GameState) dispatch =
    match gameState.CurrentStep with
    |  GameStep.Draw d when d = player.PlayerId ->
            div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [
                            (renderStepOptionButton dispatch "Draw" "is-warning" (({ GameId = gameState.GameId; PlayerId = player.PlayerId; } : DrawCardEvent) |> DrawCard) )
                            ] ]
    |  GameStep.Play d when d = player.PlayerId ->
            div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [
                            (renderStepOptionButton dispatch "End Play" "is-danger" (({ GameId = gameState.GameId; PlayerId = player.PlayerId; } : EndPlayStepEvent) |> EndPlayStep) )
                            ] ]
    |  GameStep.Attack d when d = player.PlayerId ->
            div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [
                            (renderStepOptionButton dispatch "Skip Attack" "is-danger" (({ GameId = gameState.GameId; PlayerId = player.PlayerId; } : SkipAttackEvent) |> SkipAttack) )
                        ] ]
    |  GameStep.Reconcile d when d = player.PlayerId ->
            div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [
                            (renderStepOptionButton dispatch "End Turn" "is-danger" (({ GameId = gameState.GameId; PlayerId = player.PlayerId; } : EndTurnEvent) |> EndTurn) )
                        ] ]
    |  GameStep.NotCurrentlyPlaying ->
            div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [
                            (renderStepOptionButton dispatch "Start New Game" "is-danger" (({ GameId = gameState.GameId; PlayerId = player.PlayerId; } : EndTurnEvent) |> EndTurn) )
                        ] ]
    |  GameStep.GameOver g ->
            div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [
                            (renderStepOptionButton dispatch "Start New Game" "is-danger" (({ GameId = gameState.GameId; PlayerId = player.PlayerId; } : EndTurnEvent) |> EndTurn) )
                        ] ]
    | _ -> div [] []
{% endhighlight %}

Now I can see I start on the draw stage and am able to draw but that is not transitioning me to the next step. I will have to investigate the `modifyGameStateFromDrawCardEvent` function.

Investigating this function it appears I forgot to add the state shifting functionality.

I implemented this by changing the `modifyGameStateFromDrawCardEvent` function like
{% highlight FSharp %}
let migrateGameStateToNewStep newStep (gs: GameState) =
    // maybe some vaidation could go here?
    Ok {
        gs with CurrentStep = newStep
    }

let moveCardsFromDeckToHand gs playerId pb =
    let newDeck, newHand =  drawCardsFromDeck 1 pb.Deck pb.Hand
    Ok { gs with Boards = (gs.Boards.Add (playerId, { pb with Deck = newDeck; Hand = newHand })  ) }


let modifyGameStateFromDrawCardEvent (ev: DrawCardEvent) (gs: GameState) =
    getExistingPlayerBoardFromGameState ev.PlayerId gs
    |> Result.bind (moveCardsFromDeckToHand gs ev.PlayerId)
    |> Result.bind (migrateGameStateToNewStep (ev.PlayerId |> Attack))
    |> function
        | Ok g -> g
        | Error e -> { gs with NotificationMessages = appendNotificationMessageToListOrCreateList gs.NotificationMessages e }
{% endhighlight %}

Refreshing the site I am not able to click through all the steps of my turn!

I am leaving this as the final commit of branch `step-10-better-sample-data-and-wiring-up-ui-cont`.

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-10-better-sample-data-and-wiring-up-ui-cont)