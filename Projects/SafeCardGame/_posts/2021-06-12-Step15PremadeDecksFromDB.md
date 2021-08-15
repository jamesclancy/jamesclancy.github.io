---
layout: post
title: SafeCardGame - Step 15 - Predecks From Database
author: James Clancy
tags: fsharp dotnet
---
## Pulling premade decks from the database

I first implemented the repositories. While doing this I had to move the JSON parsing from the shared DTOs to the server project (as fable doesn't support Newtonsoft JSON/ it performs weirdly). Additionally, I pulled out the row to DTO mappings into a shared module in the server project.

When implementing the repository for pulling cards from a deck I ordered by random to `shuffle` the deck on load like `select c.* from card c inner join deck_card_association dca on dca.card_id  = c.card_id where dca.deck_id = @deck_id order by RANDOM()`.

I was then able to update the API to include new methods for the premade decks like:

{% highlight FSharp %}
type ICardGameApi =
    {
        getPlayers : unit -> Async<Player seq>
        getPlayer: string -> Async<Result<Player, string>>
        getCards: unit -> Async<Card seq>
        getCard: string -> Async<Result<Card,string>>
        getDecks: unit -> Async<PreCreatedDeckDto seq>
        getCardsForDeck: string ->  Async<Card seq>
    }
{% endhighlight %}

which I then implemented like:
{% highlight FSharp %}
let playerRepository = PlayerRepository()
let cardRepository = CardRepository()
let deckRepository = DeckRepository()

let gameApi : ICardGameApi =
    {
        getPlayers = fun () ->  playerRepository.GetAll()
        getPlayer = fun playerId ->  playerRepository.Get(playerId)
        getCards = fun () -> cardRepository.GetAll()
        getCard = fun cardId -> cardRepository.Get(cardId)
        getDecks = fun () -> deckRepository.GetAll()
        getCardsForDeck = fun deckId -> deckRepository.GetCardsForDeck(deckId)
    }
{% endhighlight %}

I was then able to update the Index on the client to use this API:

{% highlight FSharp %}
let createCardInstanceForCard (card : Card) =
    let cardInstanceId = NonEmptyString.build (System.Guid.NewGuid().ToString()) |> Result.map CardInstanceId

    match cardInstanceId with
    | Ok id ->
        Ok {
                CardInstanceId  =  id
                Card =  card
        }
    | _ ->
        sprintf "Unable to create card instance for %s" (card.ToString()) |> Error

let testDeckSeqGenerator (numberOfCards :int) =
    async {
      let! values = cardGameServer.getDecks ()
      let! cards = values |> CollectionManipulation.shuffleG |> Seq.head |> (fun x -> x.DeckId) |> cardGameServer.getCardsForDeck

      return cards
        |> Seq.map createCardInstanceForCard
        |> CollectionManipulation.selectAllOkayResults
    }

{% endhighlight %}

This now appears to work but some of the cards are rending weird. To fix this I updated the PageLayElements with:

{% highlight FSharp %}
let renderThumbnailResourceCard (card: ResourceCard) =
        figure [ Class "image is-64x64" ]
                            [ img [ Src (card.ImageUrl.ToString())
                                    Alt card.Name
                                    Class "is-fullwidth" ] ]

let renderThumbnailEffectCard (card: EffectCard) =
        figure [ Class "image is-64x64" ]
                            [ img [ Src (card.ImageUrl.ToString())
                                    Alt card.Name
                                    Class "is-fullwidth" ] ]

let renderResourceCard (card: ResourceCard) =
    [
            header [ Class "card-header" ]
                       [ p [ Class "card-header-title" ]
                           [ str card.Name ]
                         p [ Class "card-header-icon" ]
                           [ str (textDescriptionForResourcePool card.ResourceCost) ] ]
            div [ Class "card-image" ]
                       [ figure [ Class "image is-4by3" ]
                           [ img [ Src (card.ImageUrl.ToString())
                                   Alt card.Name
                                   Class "is-fullwidth" ] ] ]
            div [ Class "card-content" ]
                       [ div [ Class "content" ]
                           [ p [ Class "is-italic" ]
                               [ str card.Description ]
                             displayCardSpecialEffectDetailIfPresent "On Enter Playing Field" card.EnterSpecialEffects
                             displayCardSpecialEffectDetailIfPresent "On Exit Playing Field" card.ExitSpecialEffects
                           ] ] ]

let renderEffectCard (card: EffectCard) =
    [
            header [ Class "card-header" ]
                       [ p [ Class "card-header-title" ]
                           [ str card.Name ]
                         p [ Class "card-header-icon" ]
                           [ str (textDescriptionForResourcePool card.ResourceCost) ] ]
            div [ Class "card-image" ]
                       [ figure [ Class "image is-4by3" ]
                           [ img [ Src (card.ImageUrl.ToString())
                                   Alt card.Name
                                   Class "is-fullwidth" ] ] ]
            div [ Class "card-content" ]
                       [ div [ Class "content" ]
                           [ p [ Class "is-italic" ]
                               [ str card.Description ]
                             displayCardSpecialEffectDetailIfPresent "On Enter Playing Field" card.EnterSpecialEffects
                             displayCardSpecialEffectDetailIfPresent "On Exit Playing Field" card.ExitSpecialEffects
                           ] ] ]

let renderCardDetailForHand (card: Card) : ReactElement list=
    match card with
    | CharacterCard c -> renderCharacterCard c
    | ResourceCard rc -> renderResourceCard rc
    | EffectCard ec -> renderEffectCard ec

let renderCardThumbnailForHand (card: Card) : ReactElement =
    match card with
    | CharacterCard c -> renderThumbnailCharacterCard c
    | ResourceCard rc -> renderThumbnailResourceCard rc
    | EffectCard ec -> renderThumbnailEffectCard ec
{% endhighlight %}

I am now able to play the game entirely on the client it seems. I do notice an issue where my attacks are not decrementing my available resources and I don't think my resources are resetting every turn.

I was able to update the resource resetting but altering the `EndTurnEvent` to reset the user's resources when the draw step is set like:

{% highlight FSharp %}
let modifyGameStateFromPerformAttackEvent (ev: PerformAttackEvent) (gs: GameState) =
        tryToMoodifyGameStateFromPerformAttackEvent ev gs
        |> applyErrorResultToGamesState gs

let tryToMoodifyGameStateTurnToOtherPlayer model otherPlayer  =
         CollectionManipulation.result {
           let! pb = getExistingPlayerBoardFromGameState otherPlayer model

           return { model with
                            CurrentStep = (Draw otherPlayer)
                            Boards = model.Boards.Add(otherPlayer, { pb with AvailableResourcePool = pb.TotalResourcePool})
                  }
         }

let moodifyGameStateTurnToOtherPlayer playerId model =
    getTheOtherPlayer model playerId
    |> tryToMoodifyGameStateTurnToOtherPlayer model
    |> applyErrorResultToGamesState model
{% endhighlight %}

I was able to update the attack to remove resources but updating the `modifyGameStateFromPerformAttackEvent` to also remove the resources by modifying it like:

{% highlight FSharp %}
let tryToMoodifyGameStateFromPerformAttackEvent (ev: PerformAttackEvent) (gs: GameState) =
    CollectionManipulation.result {
           let! pb = getExistingPlayerBoardFromGameState ev.PlayerId gs
           let! newGs = decrementRequiredResourcePoolFromModel ev.Attack.Cost ev.PlayerId gs pb
           let! postAttackGs = playAttackFromBoard ev.Attack ev.PlayerId newGs pb

           return! postAttackGs |> migrateGameStateToNewStep (ev.PlayerId |> Reconcile )

    }

let modifyGameStateFromPerformAttackEvent (ev: PerformAttackEvent) (gs: GameState) =
        tryToMoodifyGameStateFromPerformAttackEvent ev gs
        |> applyErrorResultToGamesState gs
{% endhighlight %}

After testing this I have noticed that the attacks should use any available colored resources for colorless resources with no more colorless resources are available.

I was able to add this functionality by updating `tryRemoveResourceFromPlayerBoard` to be a recursive function that will try to remove available resources from colored types if colorless is needed and not available.

{% highlight FSharp %}
    let rec tryRemoveResourceFromPlayerBoard (playerBoard:PlayerBoard) x y =
        match playerBoard.AvailableResourcePool.TryGetValue(x) with
        | true, z when z >= y -> Ok {playerBoard with AvailableResourcePool = (addResourcesToPool playerBoard.AvailableResourcePool  [ (x, 0-y) ])  }
        | _, _ ->
                if x = Colorless then
                    let letBestMatch = playerBoard.AvailableResourcePool
                                        |> Map.toList
                                        |> List.filter (fun (k,v) -> v > 0)
                                        |> List.tryHead

                    match letBestMatch with
                    | None -> sprintf "Not enough %s" (getSymbolForResource x) |> Error
                    | Some (k, v) when v >= y  -> Ok {playerBoard with AvailableResourcePool = (addResourcesToPool playerBoard.AvailableResourcePool  [ (k, v-y) ])  }
                    | Some (k, v) ->
                        tryRemoveResourceFromPlayerBoard
                            {playerBoard with AvailableResourcePool = (addResourcesToPool playerBoard.AvailableResourcePool  [ (k, 0) ])  }
                            x (y - v)
                else
                    sprintf "Not enough %s" (getSymbolForResource x) |> Error
{% endhighlight %}


Additionally, to make this give preference to decrementing the colored before the colorless energy I had to order the application but sorting the resource pools like:

{% highlight FSharp %}
let rec decrementResourcesFromPlayerBoard playerBoard resourcePool =

        let sortedPool = resourcePool |> List.sortBy (fun (x : Resource * int) ->
                                                                    match x with
                                                                    | Colorless, _ -> 2
                                                                    | _, _ -> 1
                                                                )
...
{% endhighlight %}

This now appears to be a low quality but playable game besides not having a win condition. I need to add the logic to actually cause a player to lose when they have zero or less health. (obviously currently you can do that first turn depending on the cards you draw which makes the game more exciting?)

I am leaving this as the final commit in the branch `step-15-deck-repository`.

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-15-deck-repository)