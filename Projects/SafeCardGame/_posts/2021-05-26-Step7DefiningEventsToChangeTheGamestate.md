---
layout: post
title: SafeCardGame - Step 7 - Defining Events To Change the GameState
author: James Clancy
tags: fsharp dotnet
---

### Defining Events To Change the GameState
---

As players play the game they can perform a number of actions that trigger events that modify the GameState.

*Note from a technical standpoint:* The GameState is actually immutable and never changes but rather a new GameState is generated from an acton and the previous state. This new state replaces the past state.


Initially, I am going to start just writing out some potential events which could happen during the game and the associated event data they would carry.

* StartGame - Initializes a new game (Creates a new GameState)
  * A unique GameID
  * Pair of Players with associated decks
  * PlayerID of Player to start game
* DrawCard - Moves the top card from the deck to the hand on the playboard referenced by the PlayerId
  * GameId
  * PlayerId
* DiscardCard - Move card with CardInstanceId from the playerboard's hand to discard pile
  * GameId
  * PlayerId
  * CardInstanceID
* PlayCard -
    If resources are available on the player's board:
        remove the card from the player's hand
        - if a resource card add to the total resource pool
        - if an effect card trigger the effect
        - if a character card creates an inplay creature and trigger the enter event. If no active create exists place the in-play creature in the active position otherwise place it on the bench.
    If the resources are not available add an error message to the gamestate
  * GameId
  * PlayerId
  * CardInstanceID
* EndPlayStep - Move the gamestate to the attack state
  * GameId
  * PlayerId
* PerformAttack -
    If the resources are available on the player's board:
        if the opponent has an inplay creature deal the damage from the attack to that creature. If the creature has <= 0 heath the creature dies
        if the opponent has no inplay creature deal the damage to the player
    if the resources are not available on the player's board display a message
  * GameId
  * PlayerId
  * InPlayCreatureId
  * Attack
* SkipAttack - Move the gamestate to reconcile
  * GameId
  * PlayerId
* EndTurn - Mode the gamestate to the other player's draw state
  * GameId
  * PlayerId
* GameWon - Move the gamestate to the game over state
  * PlayerId of Winner
  * Winning Reason (string)

There will have to be additional events for things like tapping out/retreating active creatures but I think those would best be added after everything else is set up/works.

Going through this process I have identified a few initial things that need to be added.

    * need a list of notifications on the GameState
    * need a GameStep for GameOver
    * need an optional winner on the GameOver Step
    * need a game id on the GameState

I am not thinking that the CurrentPlayerTurn should actually be on the Game Step so I am refactoring that type.

I modified the domain models with:

{% highlight FSharp %}
    and GameOverStep =
        {
            WinnerId: Option<PlayerId>
            Message: string
        }
    and GameStep =
        NotCurrentlyPlaying
        | Draw of PlayerId
        | Play of PlayerId
        | Attack of PlayerId
        | Reconcile of PlayerId
        | GameOver of GameOverStep
    and Notification = string
    and GameState =
        {
            GameId: GameId
            NotificationMessages: Option<Notification list>
            CurrentPlayer: PlayerId
            OpponentPlayer: PlayerId
            Players: Map<PlayerId, Player>
            Boards: Map<PlayerId, PlayerBoard>
            CurrentStep:GameStep
            TurnNumber: int
        }
{% endhighlight %}

I then had to update the init function for the new fields like
{% highlight FSharp %}

let init =
    let player1 = createPlayer "Player1" "Player1" 10 "https://picsum.photos/id/1000/2500/1667?blur=5"
    let player2 = createPlayer "Player2" "Player2" 10 "https://picsum.photos/id/10/2500/1667?blur=5"
    let gameId =  NonEmptyString.build "GameIDHere" |> Result.map GameId

    match player1, player2, gameId with
    | Ok p1, Ok p2, Ok g ->
        let playerBoard1 = playerBoard p1
        let playerBoard2 = playerBoard p2
        match playerBoard1, playerBoard2 with
        | Ok pb1, Ok pb2 ->
          let model =
            {
                GameId = g
                Players =  [
                            p1.PlayerId, p1;
                            p2.PlayerId, p2
                           ] |> Map.ofList
                Boards =   [
                            pb1.PlayerId, pb1;
                            pb2.PlayerId, pb2
                           ] |> Map.ofList
                NotificationMessages = None
                CurrentStep=  p1.PlayerId |> Attack
                TurnNumber = 1
                CurrentPlayer = p1.PlayerId
                OpponentPlayer = p2.PlayerId
            }
          let cmd = Cmd.ofMsg GameStarted
          Ok (model, cmd)
        | _ -> "Failed to create player boards" |> Error
    | _ -> "Failed to create players" |> Error
{% endhighlight %}

I similarly had to update the currentStepInformation to utilize the updated GameStep type. Using the fact that whose turn it is now embedded in the state I was able to simplify the yourCurrentStepClasses function like:

{% highlight FSharp %}
let yourCurrentStepClasses (gameState : GameState) (gamesStep: GameStep) =
        if gameState.CurrentStep = gamesStep then "button is-danger"
        else "button is-primary"

let currentStepInformation (player: Player) (gameState : GameState) =
    div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [ p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses gameState (player.PlayerId |> GameStep.Draw))
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Draw" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses gameState (player.PlayerId |> GameStep.Draw))
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Play" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses gameState (player.PlayerId |> GameStep.Draw))
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Attack" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses gameState (player.PlayerId |> GameStep.Draw))
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Reconcile" ] ] ] ] ]
{% endhighlight %}

At this point everything builds and I am checking it in with a commit message of `Step 7 Updates to GameState`.

In the Client/Index.fs I can now modify the Msg type to be a discriminated union of all the above-listed types. First, I am moving the Msg type into its own module/file Events/Events.fs.

Based on the above description I was able to define the events as follows:

{% highlight FSharp %}
type StartGameEvent =
    {
        GameId: GameId
        Players: Map<Player, Deck>
        StartingPlayer:PlayerId
    }
type DrawCardEvent =
    {
        GameId: GameId
        PlayerId: PlayerId
    }
type DiscardCardEvent =
    {
        GameId: GameId
        PlayerId: PlayerId
        CardInstanceId: CardInstanceId
    }
type PlayCardEvent =
    {
        GameId: GameId
        PlayerId: PlayerId
        CardInstanceId: CardInstanceId
    }
type EndPlayStepEvent =
    {
        GameId: GameId
        PlayerId: PlayerId
    }
type PerformAttackEvent =
    {
        GameId: GameId
        PlayerId: PlayerId
        InPlayCreatureId: InPlayCreatureId
        Attack: Attack
    }
type SkipAttackEvent =
    {
        GameId: GameId
        PlayerId: PlayerId
    }
type EndTurnEvent =
    {
        GameId: GameId
        PlayerId: PlayerId
    }
type GameWonEvent =
    {
        GameId: GameId
        Winner: Option<PlayerId>
        Message: Option<Notification list>
    }

type Msg =
    | GameStarted
    | StartGame of StartGameEvent
    | DrawCard of DrawCardEvent
    | DiscardCard of DiscardCardEvent
    | PlayCard of PlayCardEvent
    | EndPlayStep of EndPlayStepEvent
    | PerformAttack of PerformAttackEvent
    | SkipAttack of SkipAttackEvent
    | EndTurn of EndTurnEvent
    | GameWon of GameWonEvent

{% endhighlight %}

I am not receiving compiler errors for incomplete match statements but it builds. At this point, I am committing these changes with the message `Update Msg type to include a variety of events`.

I am keeping this as the final commit in the branch `step-7-creating-events-to-update-state`

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-7-creating-events-to-update-state)