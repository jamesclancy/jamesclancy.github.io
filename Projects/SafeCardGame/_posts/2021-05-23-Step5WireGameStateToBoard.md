---
layout: post
title: SafeCardGame - Step 5 - Wire up board to GameState
author: James Clancy
tags: fsharp dotnet
---


### Wire up Layout to the GameState and BReaking it in to Smaller Parts

The first thing I need to do is alter the init() in the `Index.fs` to return a tuple with a GamesState instead of a Model. I am then going to work through the errors until everything compiles.

In order to alter the init() I will have to come up with what the initial state should be. By the end of this  section I am going to manually add some information (i.e. not leave it in a new game state) so that something is happening on the screen to render.


First I set up my type like:

{% highlight FSharp %}
    let model =
        {
            Players =  Map.empty
            Boards = Map.empty
            CurrentTurn = None
            CurrentStep=  GameStep
            TurnNumber = 1
        }
{% endhighlight %}

I then also deleted the Model type and updated the `Msg` type to consist solely of a GameStarted type.

I then had to update the cmd of the init() like
{% highlight FSharp %} 
let cmd = Cmd.ofMsg GameStarted 
{% endhighlight %}

so that it returned a valid cmd and update the `update` function
to only match on the `GamesStarted` type.

i.e.
{% highlight FSharp %}
let update (msg: Msg) (model: GameState): GameState * Cmd<Msg> =
    match msg with
    | GameStarted ->
        model, Cmd.none
{% endhighlight %}


I was not able to save and it compiled


Next I will need to wire in the model part by part into the layout potentially updating the domain with missing items as I go.

The first thing I noticed is that I didn't have any sort of validation that my Id's were non empty so I added some by defining a `NonEmptyString` type, making the constructor private and adding a NonEmptyString namespace with a build function that creates a NonEmptyString after validating the provided string is not null or whitespace.

{% highlight FSharp %}
type NonEmptyString = private NonEmptyString of string
module NonEmptyString =
    let build str =
        if String.IsNullOrWhiteSpace str then "Value cannot be empty" |> Error
        else str |> NonEmptyString |> Ok

{% endhighlight %}

I then moved the domain types into a `Domain` module and made the various Ids types of NonEmptyStrings.

i.e.
{% highlight FSharp %}
module Domain =

    type PlayerId = NonEmptyString
    type CardInstanceId = NonEmptyString
    type CardId = NonEmptyString
    type InPlayCreatureId = NonEmptyString

    type Player =
        { ...
{% endhighlight %}

I then added the arguments `model` and `dispatch` to pass through from the `view` to the `mainLayout`.

Inside the mainLayout the enemyStats and enemyCreatures will depend on the opponent `Player` and opponent `PlayerBoard`
and  `playerCreatures` and `playerHand` will depend on your `PlayerBoard` and the `playerControlCenter` will depend on the entire `GameState`.

First, we will need to pull out the opponent and player board and player but I have noticed that the GamesState does not define the current player vs the opponent so I am adding a property to the GameState to store this information.

i.e.
{% highlight FSharp %}
    and GameState =
        {
            CurrentPlayer: PlayerId
            OpponentPlayer: PlayerId
            ...
{% endhighlight %}

I then wrote functions to extract the needed information, wrapped them in results and a single function to extract all the data.

I then wrapped the main layout to verify no errors were encountered in building the models and changed the function to return an error message if any issues are encountered.

i.e.

{% highlight FSharp %}

let opponentPlayer (model : GameState) =
    match model.Players.TryGetValue model.OpponentPlayer with
    | true, p -> p |> Ok
    | false, _ -> "Unable to locate oppponent in player list" |> Error


let opponentPlayerBoard (model : GameState) =
    match model.Boards.TryGetValue model.OpponentPlayer with
    | true, p -> p |> Ok
    | false, _ -> "Unable to locate oppponent in board list" |> Error

let currentPlayer (model : GameState) =
    match model.Players.TryGetValue model.CurrentPlayer with
    | true, p -> p |> Ok
    | false, _ -> "Unable to locate current player in player list" |> Error


let currentPlayerBoard (model : GameState) =
    match model.Boards.TryGetValue model.CurrentPlayer with
    | true, p -> p |> Ok
    | false, _ -> "Unable to locate current player in board list" |> Error

let extractNeededModelsFromState (model: GameState) =
    opponentPlayer model, opponentPlayerBoard model, currentPlayer model, currentPlayerBoard model


let mainLayout  model dispatch =
  match extractNeededModelsFromState model with
  | Ok op, Ok opb, Ok cp, Ok cpb ->
      div [ Class "container is-fluid" ]
        [ topNavigation
          br [ ]
          br [ ]
          enemyStats
          enemyCreatures
          playerControlCenter
          playerCreatures
          playerHand
          footerBand
        ]
  | _ -> strong [] [ str "Error in GameState encountered." ]
{% endhighlight %}

Looking further at the `Player` definition I see that it is missing a playmat URL for their background graphic. I added that as well as definitions for two new primates domain types which should only hold valid URLs:

{% highlight FSharp %}

type UrlString = private UrlString of NonEmptyString
module UrlString =
    let build str =
        if Uri.IsWellFormedUriString(str, UriKind.RelativeOrAbsolute) then "Value must be a url." |> Error
        else str |> NonEmptyString.build |> Result.map UrlString


type ImageUrlString = private ImageUrlString of UrlString
module ImageUrlString =

    let isUrlImage (str : string) =
        let ext  = Path.GetExtension(str)
        if String.IsNullOrWhiteSpace ext then false
        else
            match str with
            | "png" | "jpg" ->
                true
            | _ -> false

    let build str =
        if isUrlImage str then "Value must be a url pointing to iamge." |> Error
        else str |> UrlString.build |> Result.map ImageUrlString

{% endhighlight %}

and

{% highlight FSharp %}

    type Player =
        {
            PlayerId: PlayerId
            Name: string
            PlaymatUrl: ImageUrlString
            RemainingLifePoints: int
        }

{% endhighlight %}

In the Init I now have to add PlayerIds for the opponent and and current player. To do this I created a createPlayer function which takes a playerIdStr playerName playerCurrentLife playerPlaymatUrl and returns a `Player`.

{% highlight FSharp %}
let createPlayer playerIdStr playerName playerCurrentLife playerPlaymatUrl =
    let playerId = NonEmptyString.build playerIdStr |> Result.map PlayerId
    let playerPlaymatUrl = ImageUrlString.build playerPlaymatUrl

    match playerId, playerPlaymatUrl with
    | Ok s, Ok pm ->
        Ok {
           PlayerId = s
           Name = playerName
           RemainingLifePoints = playerCurrentLife
           PlaymatUrl = pm
        }
    | _ -> Error "Unable to create player"
{% endhighlight %}

I then modified the init to create two players and plug them into the GameState. In doing this modification I had to switch the init to return a Result type.

i.e.

{% highlight FSharp %}
let init =
    let player1 = createPlayer "Player1" "Player1" 10 "https://picsum.photos/id/1000/2500/1667?blur=5"
    let player2 = createPlayer "Player2" "Player2" 10 "https://picsum.photos/id/10/2500/1667?blur=5'"
    match player1, player2 with
    | Ok p1, Ok p2 ->
        let model =
            {
                Players =  Map.empty
                Boards = Map.empty
                CurrentTurn = None
                CurrentStep=  NotCurrentlyPlaying
                TurnNumber = 1
                CurrentPlayer = p1.PlayerId
                OpponentPlayer = p2.PlayerId
            }
        let cmd = Cmd.ofMsg GameStarted
        Ok (model, cmd)
    | _ -> "Failed to create players" |> Error
{% endhighlight %}


I then had to modify the App.fs in the client to accept a result set back from the init like

{% highlight FSharp %}

match Index.init with
| Ok (gamesState, dispatch) ->
    let placeHolderFunc () =
        gamesState, dispatch
    Program.mkProgram placeHolderFunc Index.update Index.view
    ...
{% endhighlight %}

At this point, I ran into an error where the methods I was using from Path and Uri to validate the URLs are not implemented in FABLE. While FABLE implements a large part of the .NET framework some parts are not implemented. for now, I basically made the functions just to validate that the strings not null.


It now builds and displays the `Error in GameState encountered.` message from the view definition (since the players and boards are not registered in their respective maps).

Now I will have to update the init to populate these values.

I updated the Players value to
{% highlight FSharp %}

                Players =  [
                            p1.PlayerId, p1;
                            p2.PlayerId, p2
                           ] |> Map.ofList
{% endhighlight %}

Additionally, I will need to create a function to generate a player board.

{% highlight FSharp %}
let playerBoard player =
    {
            PlayerId=  player.PlayerId
            Deck= {
                TopCardsExposed = 0
                Cards =  List.empty
            }
            Hand= { Cards = List.empty }
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

This was not working for me. I spent a long time trying to figure out why. The end answer was that my ImageUrlString build method needed a `not` and was actually verifying that the image URL was invalid.

Now it is once again loading the page and I am able to start padding these values to my layout parts.

First I will pass the opponent `Player` and `PlayerBoard` to the `enemyStats` and plug those in.


Similarly, I will do the same for the player control center.


Here I can notice that the health, hand, deck and discard items are shared between the two nav bars and extract that as a shared function.

I, therefore, pulled out a playerStats function:

{% highlight FSharp %}
let playerStats  (player: Player) (playerBoard: PlayerBoard) =
    [             div [ Class "navbar-item" ]
                    [ a [ Class "button is-primary"
                          Href "#" ]
                        [ str (sprintf "💓 %i/10" player.RemainingLifePoints) ] ]
                  div [ Class "navbar-item" ]
                    [ a [ Class "button is-primary"
                          Href "#" ]
                        [ str (sprintf "🤚 %i" playerBoard.Hand.Cards.Length) ] ]
                  div [ Class "navbar-item" ]
                    [ a [ Class "button is-primary"
                          Href "#" ]
                        [ str (sprintf "🂠 %i" playerBoard.Deck.Cards.Length) ] ]
                  div [ Class "navbar-item" ]
                    [ a [ Class "button is-primary"
                          Href "#" ]
                        [ str (sprintf "🗑️ %i" playerBoard.DiscardPile.Cards.Length)] ] ]

{% endhighlight %}

I can also pull out the step information into a `currentStepInformation` function. I can then utilize it if it is the player's turn and the GameState.CurrentStep to select the classes for the steps like:

{% highlight FSharp %}
let yourCurrentStepClasses (player: Player) (gameState : GameState) (gamesStep: GameStep) =
    if not (player.PlayerId = gameState.CurrentPlayer) then "button is-primary"
    else
        if gameState.CurrentStep = gamesStep then "button is-danger"
        else "button is-primary"

let currentStepInformation (player: Player) (playerBoard: PlayerBoard) (gameState : GameState) =
    div [ Class "navbar-item" ]
                    [ div [ Class "field is-grouped has-addons is-grouped-right" ]
                        [ p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses player gameState GameStep.Draw)
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Draw" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses player gameState GameStep.Play)
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Play" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses player gameState GameStep.Attack)
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Attack" ] ] ]
                          p [ Class "control" ]
                            [ button [ Class (yourCurrentStepClasses player gameState GameStep.Reconcile)
                                       Disabled true ]
                                [ span [ ]
                                    [ str "Reconcile" ] ] ] ] ]
{% endhighlight %}

All this builds so I am not able to try to populate the Player Hand from the PlayerBoard.Hand. This can be rendered by rendering each card from the list in a yield.

I can do this like
{% highlight FSharp %}
let renderCardForHand (card: Card) =
  div [ Class "column is-4" ]
                [ div [ Class "card" ]
                    [ header [ Class "card-header" ]
                        [ p [ Class "card-header-title" ]
                            [ str "Card Name" ]
                          p [ Class "card-header-icon" ]
                            [ str "🍂 x4" ] ]
                      div [ Class "card-image" ]
                        [ figure [ Class "image is-4by3" ]
                            [ img [ Src "https://picsum.photos/320/200?1"
                                    Alt "Placeholder image"
                                    Class "is-fullwidth" ] ] ]
                      div [ Class "card-content" ]
                        [ div [ Class "content" ]
                            [ p [ Class "is-italic" ]
                                [ str "This is a sweet description." ]
                              p [ Class "is-italic" ]
                                [ strong [ ]
                                    [ str "On Enter Playing Field" ]
                                  str ": Effect description." ]
                              p [ Class "is-italic" ]
                                [ strong [ ]
                                    [ str "On Exit Playing Field" ]
                                  str ": Effect description." ]
                              h5 [ Class "IsTitle is5" ]
                                [ str "Attacks" ]
                              table [ ]
                                [ tr [ ]
                                    [ td [ ]
                                        [ str "🍂 x1" ]
                                      td [ ]
                                        [ str "Leaf Cut" ]
                                      td [ ]
                                        [ str "10" ] ]
                                  tr [ ]
                                    [ td [ ]
                                        [ str "🍂 x2" ]
                                      td [ ]
                                        [ str "Vine Whip" ]
                                      td [ ]
                                        [ str "30" ] ] ] ] ]
                      footer [ Class "card-footer" ]
                        [ a [ Href "#"
                              Class "card-footer-item" ]
                            [ str "Play" ]
                          a [ Href "#"
                              Class "card-footer-item" ]
                            [ str "Discard" ] ] ] ]

let renderCardInstanceForHand (card: CardInstance) =
    renderCardForHand card.Card

let playerHand (hand : Hand) =
  section [ Class "section" ]
    [ div [ Class "container py-4" ]
        [ h3 [ Class "title is-spaced is-4" ]
            [ str "Hand" ]
          div [ Class "columns is-mobile mb-5" ]
            [ yield! Seq.map renderCardInstanceForHand hand.Cards ] ] ]

{% endhighlight %}

Next to see some data I will have to populate some hand data into the init object.

To do this I am going to create a `testCardSeqGenerator` function. This will take a number and return that number of randomly generated cards.

While doing this I realized the all the `GameStateSpecialEffect` in the cards and attacks should be Optional and updated the domain like
{% highlight FSharp %}
    and CharacterCard
        = { CardId: CardId;
            Name: string;
            Creature: Creature;
            ResourceCost: ResourcePool;
            PrimaryResource: Resource;
            EnterSpecialEffects: Option<GameStateSpecialEffect>;
            ExitSpecialEffects: Option<GameStateSpecialEffect>
          }
    and EffectCard
        = {
            CardId: CardId;
            Name: string;
            ResourceCost: ResourcePool;
            PrimaryResource: Resource;
            EnterSpecialEffects: Option<GameStateSpecialEffect>;
            ExitSpecialEffects: Option<GameStateSpecialEffect>;
        }
    and ResourceCard
        = {
            CardId: CardId;
            Name: string;
            ResourceCost: ResourcePool;
            PrimaryResource: Resource;
            EnterSpecialEffects: Option<GameStateSpecialEffect>;
            ExitSpecialEffects: Option<GameStateSpecialEffect>;
            ResourceAvailableOnFirstTurn: bool;
            ResourcesAdded: ResourcePool
        }
    ...
    and Attack =
        {
            Damage: int
            Cost: ResourcePool
            SpecialEffect: Option<GameStateSpecialEffect>
        }

{% endhighlight %}

After playing around for a while I was eventually able to create functions to generate test data:

{% highlight FSharp %}

let testCardGenerator cardInstanceIdStr cardIdStr =

    let cardInstanceId = NonEmptyString.build cardInstanceIdStr |> Result.map CardInstanceId

    let cardId = NonEmptyString.build cardInstanceIdStr |> Result.map CardId

    match cardInstanceId, cardId with
    | Ok id, Ok cid ->
        let creature =
          {
            Health= 95
            Weaknesses=  List.empty
            Attach = List.empty
          }
        let card =
            {
                CardId = cid
                ResourceCost = [ Resource.Grass, 4;
                                 Resource.Colorless, 1 ] |> Seq.ofList |> ResourcePool
                Name = cardIdStr
                EnterSpecialEffects = None
                ExitSpecialEffects = None
                PrimaryResource = Resource.Grass
                Creature = creature
            }
        Ok  {
                CardIntanceId  =  id
                Card =  card |> CharacterCard
            }
    | _, _ ->
        sprintf "Unable to create card instance for %s\t%s" cardInstanceIdStr cardIdStr
        |> Error

let testCardSeqGenerator (numberOfCards : int) =
    seq { 0 .. (numberOfCards - 1) }
    |> Seq.map (sprintf "Exciting Character #%i")
    |> Seq.map (fun x -> testCardGenerator x x)
    |> Seq.map (fun x ->
                        match x with
                        | Ok s -> [ s ] |> Ok
                        | Error e -> e |> Error)
    |> Seq.fold (fun x y ->
                    match x, y with
                    | Ok accum, Ok curr -> curr @ accum |> Ok
                    | _,_ -> "Eating Errors lol" |> Error
                ) (Ok List.empty)
{% endhighlight %}

One of the major issues I encountered was transforming the Sequence of results into a Result of a sequence. I ended up having to map the Seq<Result<Card, string>> to a Seq<Result<Card list, string>> and fold across that.

I was then able to update the `playerBoard` function to

{% highlight FSharp %}
let playerBoard (player : Player) =
    let deckTemp =  testCardSeqGenerator 35
    let handTemp = testCardSeqGenerator 3

    match deckTemp, handTemp with
    | Ok deck, Ok hand ->
        Ok  {
                PlayerId=  player.PlayerId
                Deck= {
                    TopCardsExposed = 0
                    Cards =  deck
                }
                Hand=
                    {
                        Cards = hand
                    }
                ActiveCreature= None
                Bench=  None
                DiscardPile= {
                    TopCardsExposed = 0
                    Cards = List.empty
                }
                TotalResourcePool= ResourcePool Seq.empty
                AvailableResourcePool =  ResourcePool Seq.empty
            }
    | _,_ -> "Error creating deck or hand" |> Error
{% endhighlight %}

I feel like there has to be a much better way to deal with these Result types but I am going to keep driving on and revisit later.

In building out this generator I realized I needed to add an image URL for the cards so I added an `ImageUrl` to the types of type `ImageUrlString` and added this to the builder.

Now I need to update the `renderCardForHand`. First I switch the card type and push the character cards into a `renderCharacterCard`.

Apparently, a description is also needed on the card types.

Also, a description is needed on the `GameStateSpecialEffect`.

Also, a `ToString` override is needed for the `ImageUrlString` type. I set this by overriding all the ToString methods on the `NonEmptyString`, `UrlString`, and `ImageUrlString`.

i.e.

{% highlight FSharp %}
type NonEmptyString = private NonEmptyString of string
    with override this.ToString() = match this with NonEmptyString s -> s
...

type UrlString = private UrlString of NonEmptyString
    with override this.ToString() = match this with UrlString s -> s.ToString()
...

type ImageUrlString = private ImageUrlString of UrlString
    with override this.ToString() = match this with ImageUrlString s -> s.ToString()

{% endhighlight %}

This has proven to be extremely time-consuming and I am calling it quits for the day.

This is the last commit in the branch `step-5-wire-up-layout-to-gamestate`

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-5-wire-up-layout-to-gamestate)