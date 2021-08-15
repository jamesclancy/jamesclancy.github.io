---
layout: post
title: SafeCardGame - Step 8 - Wire up events to modify GameState
author: James Clancy
tags: fsharp dotnet
---

In the next step/section I will be wiring up the events to actually modify the game state. Finally, we will be removing the `GameStarted` event and replace the initializing function with a hydrated `StartGame` event.

### Wire up events to modify GameState
---

These updates to the gamestate will occur in the update function of the client Index.fs


First I scaffolded out the update function like:

{% highlight FSharp %}
let update (msg: Msg) (model: GameState): GameState * Cmd<Msg> =
    match msg with
    | GameStarted ->
        model, Cmd.none
    | StartGame ev ->
        model, Cmd.none
    | DrawCard  ev ->
        model, Cmd.none
    | DiscardCard ev ->
        model, Cmd.none
    | PlayCard ev ->
        model, Cmd.none
    | EndPlayStep ev ->
        model, Cmd.none
    | PerformAttack  ev ->
        model, Cmd.none
    | SkipAttack ev ->
        model, Cmd.none
    | EndTurn ev ->
        model, Cmd.none
    | GameWon ev ->
        model, Cmd.none
{% endhighlight %}

First I will implement to `StartGame` handler.


In order to do this I first created a `takeDeckDealFirstHandAndReturnNewPlayerBoard` function:

{% highlight FSharp %}
let takeDeckDealFirstHandAndReturnNewPlayerBoard (intitalHandSize: int) (playerId : PlayerId) (deck : Deck) =
    let emptyHand =
      {
        Cards = list.Empty
      }
    let deckAfterDraw, hand = drawCardsFromDeck intitalHandSize deck emptyHand

    {
        PlayerId=  playerId
        Deck= deckAfterDraw
        Hand=hand
        ActiveCreature= None
        Bench=  None
        DiscardPile= {
            TopCardsExposed = 0
            Cards = List.empty
        }
        TotalResourcePool= ResourcePool Seq.empty
        AvailableResourcePool =  ResourcePool Seq.empty
    }
{% endhighlight %}

This references a new function to draw cards:
{% highlight FSharp %}
let drawCardsFromDeck (cardsToDraw: int) (deck : Deck) (hand: Hand) =
    if deck.Cards.IsEmpty then
        deck, hand
    else
        let cardsToTake = List.truncate cardsToDraw deck.Cards
        { deck with Cards = List.skip cardsToTake.Length deck.Cards}, {hand with Cards = hand.Cards @ cardsToTake}

{% endhighlight %}

Then I was able to initialize the state as:
{% highlight FSharp %}
let intitalizeGameStateFromStartGameEvent (ev : StartGameEvent) =
            {
                GameId= ev.GameId
                NotificationMessages= None
                CurrentPlayer= ev.CurrentPlayer
                OpponentPlayer= ev.OpponentPlayer
                Players= ev.Players
                Boards= ev.Decks
                        |> Seq.map (fun x -> x.Key, takeDeckDealFirstHandAndReturnNewPlayerBoard 7 x.Key x.Value )
                        |> Map.ofSeq
                CurrentStep= ev.CurrentPlayer |> Draw
                TurnNumber= 1
            }

{% endhighlight %}

Then I just have to call intitalizeGameStateFromStartGameEvent from my update match statement.

Using the same draw function I can implement the draw event.

{% highlight FSharp %}
let appendNotificationMessageToListOrCreateList (existingNotifications : Option<Notification list) (newNotification : string) =
    match existingNotifications with
    | Some nl ->
        newNotification
        |> Notification
        |> (fun y -> y :: nl)
        |> Some
    | None ->
        [ (newNotification |> Notification) ]
        |> Some

let modifyGameStateFromDrawCardEvent (ev: DrawCardEvent) (gs: GameState) =
    match gs.Boards.TryGetValue ev.PlayerId with
    | true, pb ->
        let newDeck, newHand =  drawCardsFromDeck 1 pb.Deck pb.Hand
        { gs with Boards = (gs.Boards.Add (ev.PlayerId, { pb with Deck = newDeck; Hand = newHand })  ) }
    | false, _ ->
        { gs with NotificationMessages = appendNotificationMessageToListOrCreateList gs.NotificationMessages "Unable to lookup player board" }
{% endhighlight %}

I realize there has to be a better way to pipe into a list than `|> (fun y -> y :: nl)` but I am just going to leave that for now.

Discard card is next. I noticed a typo in the name `CardInstanceId` so I corrected that. Also, I notice I will similarly have to pull the player board from the state so I should extract that to be a function like:

{% highlight FSharp %}
let getExistingPlayerBoardFromGameState playerId gs =
 match gs.Boards.TryGetValue playerId with
    | true, pb ->
        pb |> Ok
    | false, _ ->
        (sprintf "Unable to locate player board for player id %s" (playerId.ToString())) |> Error
{% endhighlight %}

I am then able to implement a discardCardFromBoard function like:

{% highlight FSharp %}
let discardCardFromBoard (cardInstanceId : CardInstanceId) (playerBoard : PlayerBoard) =
    let cardToDiscard : CardInstance list = List.filter (fun x -> x.CardInstanceId = cardInstanceId) playerBoard.Hand.Cards

    match cardToDiscard with
    | [] ->
        (sprintf "Unable to locate card in hand with card instance id %s" (cardInstanceId.ToString())) |> Error
    | [ x ] ->
            {
                playerBoard
                    with Hand =
                          { playerBoard.Hand with

                                Cards = (List.filter (fun x -> x.CardInstanceId = cardInstanceId) playerBoard.Hand.Cards)

                          };
                         DiscardPile ={playerBoard.DiscardPile with Cards = playerBoard.DiscardPile.Cards @ [ x ] }
            } |> Ok
    | _ ->
        (sprintf "ERROR: located multiple cards in hand with card instance id %s. This shouldn't happen" (cardInstanceId.ToString())) |> Error

let modifyGameStateFromDiscardCardEvent (ev: DiscardCardEvent) (gs: GameState) =
    let newBoard =
        getExistingPlayerBoardFromGameState ev.PlayerId gs
        |> Result.bind (discardCardFromBoard ev.CardInstanceId)

    match newBoard with
    | Ok pb ->
        { gs with Boards = (gs.Boards.Add (ev.PlayerId, pb)  ) }
    | Error e ->
        { gs with NotificationMessages = appendNotificationMessageToListOrCreateList gs.NotificationMessages e }

{% endhighlight %}

The end play step should just move the player to the Attack step like:

{% highlight FSharp %}
    | EndPlayStep ev ->
        { model with CurrentStep = (Attack ev.PlayerId)}, Cmd.none
{% endhighlight %}

Similarly, SkipAttack should just move the player to the Reconcile step.

{% highlight FSharp %}
    | SkipAttack ev ->
        { model with CurrentStep = (Reconcile ev.PlayerId)}, Cmd.none
{% endhighlight %}

EndTurn should just move the game to the draw step of the other player.

{% highlight FSharp %}
    | EndTurn ev ->
        let otherPlayer = getTheOtherPlayer model ev.PlayerId
        { model with CurrentStep = (Draw otherPlayer)}, Cmd.none
{% endhighlight %}

Game Won should transition to the GameOverState, set a winner and a message but I will first have to define a function to format the GameOverMessage like:

{% highlight FSharp %}
let formatGameOverMessage (notifications : Option<Notification list>) =
    match notifications with
    | None ->
        "Game Over for unknown reason"
    | Some [] ->
        "Game Over for unknown reason"
    | Some x ->
        x
        |> Seq.map (fun x -> x.ToString())
        |> String.concat ";"
{% endhighlight %}

then I can

{% highlight FSharp %}
    | GameWon ev ->
        let newStep =  { WinnerId =  ev.Winner; Message = formatGameOverMessage ev.Message } |> GameOver
        { model with CurrentStep = newStep}, Cmd.none
{% endhighlight %}

This involved some copypasta which could be removed and refactored but I am just going to keep driving on for now.

This now just leaves me to complete the play card and attack events.

For the play card action, I will need to implement a variety of functions. These will require refactoring but just to get something down I added

{% highlight FSharp %}
let applyEffectIfDefinied effect gs =
    match effect with
    | Some e -> e.Function.Invoke gs |> Ok
    | None  -> gs |> Ok

let currentResourceAmountFromPool (resourcePool : ResourcePool) (resource : Resource) =
    match resourcePool.TryGetValue(resource) with
    | true, t -> t
    | false, _ -> 0

let rec addResourcesToPool (rp1 : Map<Resource, int>)  rp2 =
    match rp2 with
    | [] ->  rp1 |> Map.toSeq |> ResourcePool
    | [ (x,y) ] ->
        rp1.Add(x, y + (currentResourceAmountFromPool rp1 x))  |> Map.toSeq |> ResourcePool
    | (x,y) :: xs  ->
        addResourcesToPool (rp1.Add(x,  y + (currentResourceAmountFromPool rp1 x))) xs


let createInPlayCreatureFromCardInstance characterCard inPlayCreatureId =
            {
                InPlayCharacterId=  inPlayCreatureId
                Card = characterCard
                CurrentDamage=  0
                SpecialEffect=  None
                AttachedEnergy =  Seq.empty |> ResourcePool
                SpentEnergy = Seq.empty |> ResourcePool
            } |> Ok

let appendCreatureToPlayerBoard inPlayCreature playerBoard =
        match playerBoard.ActiveCreature with
        | None ->
            { playerBoard with ActiveCreature = Some inPlayCreature }
        | Some a ->
            { playerBoard
                with Bench
                    = Some (Option.fold (fun x y -> x @ y)  [ inPlayCreature ]  playerBoard.Bench)  }


let addCreatureToGameState cardInstanceId x playerId gs playerBoard inPlayCreature=
        let playerBoard =
                        {
                            playerBoard
                                with Hand =
                                        {
                                          playerBoard.Hand with
                                            Cards = (List.filter (fun x -> x.CardInstanceId = cardInstanceId) playerBoard.Hand.Cards)
                                        };
                                     DiscardPile ={playerBoard.DiscardPile with Cards = playerBoard.DiscardPile.Cards @ [ x ] }
                        } |> (appendCreatureToPlayerBoard inPlayCreature)

        { gs with Boards = gs.Boards.Add ( playerId, playerBoard) } |> Ok

let buildInPlayCreatureId idStr =
    let res = idStr |> NonEmptyString.build

    match res with
    | Ok r -> InPlayCreatureId r |> Ok
    | Error e -> Error e


let playCardFromBoard (cardInstanceId : CardInstanceId) (playerId : PlayerId) (gs: GameState) (playerBoard : PlayerBoard) =
    let cardToDiscard : CardInstance list = List.filter (fun x -> x.CardInstanceId = cardInstanceId) playerBoard.Hand.Cards

    match cardToDiscard with
    | [] ->
        (sprintf "Unable to locate card in hand with card instance id %s" (cardInstanceId.ToString())) |> Error
    | [ x ] ->
            match x.Card with
            | CharacterCard cc ->

              System.Guid.NewGuid().ToString()
              |> buildInPlayCreatureId
              |> (Result.bind (createInPlayCreatureFromCardInstance x.Card))
              |> (Result.bind (addCreatureToGameState cardInstanceId x playerId gs playerBoard))
              |> (Result.bind  (applyEffectIfDefinied cc.EnterSpecialEffects))

            | ResourceCard rc ->
              let newPb = {
                  playerBoard
                    with Hand =
                          { playerBoard.Hand with

                                Cards = (List.filter (fun x -> x.CardInstanceId = cardInstanceId) playerBoard.Hand.Cards)
                          };
                         TotalResourcePool = addResourcesToPool playerBoard.TotalResourcePool (Map.toList rc.ResourcesAdded)
                         DiscardPile ={playerBoard.DiscardPile with Cards = playerBoard.DiscardPile.Cards @ [ x ] }
                }
              { gs with Boards = (gs.Boards.Add (playerId, newPb)) } |> (applyEffectIfDefinied rc.EnterSpecialEffects) |> Result.bind (applyEffectIfDefinied rc.ExitSpecialEffects)
            | EffectCard ec ->
              let newPb =  {
                  playerBoard
                    with Hand =
                          { playerBoard.Hand with

                                Cards = (List.filter (fun x -> x.CardInstanceId = cardInstanceId) playerBoard.Hand.Cards)
                          };
                         DiscardPile = {playerBoard.DiscardPile with Cards = playerBoard.DiscardPile.Cards @ [ x ] }
                }
              { gs with Boards = (gs.Boards.Add (playerId, newPb) ) } |> (applyEffectIfDefinied ec.EnterSpecialEffects) |> Result.bind (applyEffectIfDefinied ec.ExitSpecialEffects)
    | _ ->
        (sprintf "ERROR: located multiple cards in hand with card instance id %s. This shouldn't happen" (cardInstanceId.ToString())) |> Error


let modifyGameStateFromPlayCardEvent (ev: PlayCardEvent) (gs: GameState) =
    let newBoard =
        getExistingPlayerBoardFromGameState ev.PlayerId gs
        |> Result.bind (playCardFromBoard  ev.CardInstanceId ev.PlayerId gs)

    match newBoard with
    | Ok pb ->
        pb
    | Error e ->
        { gs with NotificationMessages = appendNotificationMessageToListOrCreateList gs.NotificationMessages e }

{% endhighlight %}

I am leaving the attack logic for later and am now focuses on testing to verify what I have done so far is working.

I am now interested in building a more realistic gamestate for testing.

This is the final commit in the branch `step-8-wire-up-events-to-update-game-step`

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-8-wire-up-events-to-update-game-step)