---
layout: post
title: SafeCardGame - Step 11 - Switching the player for the GameState
author: James Clancy
tags: fsharp dotnet
---

## Switching the player for the GameState

To facilitate testing, I am not going to add logic to swap Player 1 and Player 2. This should allow us to step through an entire game.

To do this I will add a Msg of type SwapPlayer.

i.e.

{% highlight FSharp %}
type Msg =
    | StartGame of StartGameEvent
    | DrawCard of DrawCardEvent
    | DiscardCard of DiscardCardEvent
    | PlayCard of PlayCardEvent
    | EndPlayStep of EndPlayStepEvent
    | PerformAttack of PerformAttackEvent
    | SkipAttack of SkipAttackEvent
    | EndTurn of EndTurnEvent
    | DeleteNotification of Guid
    | GameWon of GameWonEvent
    | SwapPlayer
{% endhighlight %}

I will then handle the message in the update like:

{% highlight FSharp %}
let update (msg: Msg) (model: GameState): GameState * Cmd<Msg> =
    match msg with
    ...
    | SwapPlayer ->
        { model with PlayerOne = model.PlayerTwo; PlayerTwo = model.PlayerOne }, Cmd.none

{% endhighlight %}

Now I can add a button to switch players on the top nav bar like:

{% highlight FSharp %}
let topNavigation dispatch =
             ...
              div [ Class "navbar-end" ]
                [ div [ Class "navbar-item" ]
                    [ div [ Class "buttons" ]
                        [ a [ Class "button is-light"
                              OnClick (fun _ -> SwapPlayer |> dispatch)
                              Href "#" ]
                            [ str "Switch Player" ]
                          a [ Class "button is-light"
                              Href "#" ]
                            [ str "Log in" ]
             ...
{% endhighlight %}

While doing this I found a bug in the, it should have been written like:

{% highlight FSharp %}
let currentStepInformation (player: Player) (gameState : GameState)  =
    div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [ p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses gameState (player.PlayerId |> GameStep.Draw))
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Draw" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses gameState (player.PlayerId |> GameStep.Play))
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Play" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses gameState (player.PlayerId |> GameStep.Attack))
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Attack" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses gameState (player.PlayerId |> GameStep.Reconcile))
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Reconcile" ] ] ] ] ]
{% endhighlight %}
Also, I just noticed that the Draw is moving to the Attack step instead of the play step.

To fix this I changed the event that was being dispatched like

{% highlight FSharp %}
let modifyGameStateFromDrawCardEvent (ev: DrawCardEvent) (gs: GameState) =
    getExistingPlayerBoardFromGameState ev.PlayerId gs
    |> Result.bind (moveCardsFromDeckToHand gs ev.PlayerId)
    |> Result.bind (migrateGameStateToNewStep (ev.PlayerId |> Play))
    |> function
        | Ok g -> g
        | Error e -> { gs with NotificationMessages = appendNotificationMessageToListOrCreateList gs.NotificationMessages e }
{% endhighlight %}

Also, in this branch, it was recommended to me that you can zoom cards. I am attempting to add a model that displays card details to the hand.

To try this I am trying to add a modal on click for the hand.


Meanwhile, I am noticing I really need to refactor out operators for stuff like :
{% highlight FSharp %}
    |        |> Result.bind (applyUpdatedPlayerBoardResultToGamesState playerId gs x)
{% endhighlight %}
and
{% highlight FSharp %}
        |> Result.bind (fun x -> (applyUpdatedPlayerBoardResultToGamesState playerId gs x) |> Ok)

{% endhighlight %}
i did this by implementing infix operators like

{% highlight FSharp %}

let (>>=) twoTrackInput switchFunction =
    Result.bind switchFunction twoTrackInput

let (>=>) switch1 switch2 x =
    match switch1 x with
    | Ok s -> switch2 s
    | Error f -> Error f

{% endhighlight %}

Additionally, I moved many of the constructors for domain objects to the Shared Domain and deleted unused test generator functions.

Eventually, I was able to get the modal to work by adding a property to the game board: `ZoomedInCard` this is an optional CardInstanceId. I then added functions to check if the zoomed-in card was set to the current card. If it is the view displays the modal, if not it is hidden. I also added a Msg ZoomedInCardToggled and wired that up to the update function and the click of the thumbnail image in the hand.

This all builds and is the final commit in the `step-11-allow-the-user-to-switch-to-being-the-other-player` branch.

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-11-allow-the-user-to-switch-to-being-the-other-player)