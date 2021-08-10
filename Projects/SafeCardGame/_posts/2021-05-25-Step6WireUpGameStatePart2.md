---
layout: post
title: SafeCardGame - Step 6 - Wiring up the Layout to the GameState Part 2
author: James Clancy
tags: fsharp dotnet
---


### Wiring up the Layout to the GameState Part 2
---

Now I am going to continue to wire up the layout to the GameState. Previously, I was able to generate some cards and decks. Doing that I realize I can pull out a function to map Resources to a symbol.

In order to facilitate this mapping I created a function:

```
let getSymbolForResource resource =
    match resource with
    | Grass -> "🍂"
    | Fire -> "🔥"
    | Water -> "💧"
    | Lightning -> "⚡"
    | Psychic -> "🧠"
    | Fighting -> "👊"
    | Colorless -> "□"
```

Using this I can create a function mapping a `ResourcePool' to a string like:

```
let textDescriptionForResourcePool (resourcePool : ResourcePool) =
    resourcePool
    |> Seq.map (fun x -> sprintf "%s x%i" (getSymbolForResource x.Key) x.Value)
    |> String.concat ";"
```

Using this function I can break out a renderAttackRow function. To do this I had to add a Name property to the `Attack` type. After adding the `Name` I can create a method like:

```

let renderAttackRow (attack: Attack) =
    match attack.SpecialEffect with
    | Some se ->
        tr [ ]
            [ td [ ]
                [ str (textDescriptionForResourcePool attack.Cost) ]
              td [ ]
                [ str attack.Name ]
              td [ ]
                [
                    p [] [ str (sprintf "%i" attack.Damage) ]
                    p [] [ str se.Description ]
                ] ]
    | None ->
        tr [ ]
            [ td [ ]
                [ str (textDescriptionForResourcePool attack.Cost) ]
              td [ ]
                [ str attack.Name ]
              td [ ]
                [
                    p [] [ str (sprintf "%i" attack.Damage) ]
                ] ]
```

and reference it like

```
let renderCharacterCard (card: CharacterCard) =
      div [ Class "column is-4" ]
                [ div [ Class "card" ]
                    [ header [ Class "card-header" ]
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
                              h5 [ Class "IsTitle is5" ]
                                [ str "Attacks" ]
                              table [ ]
                                [
                                  yield! seq {
                                    for a in card.Creature.Attach do
                                      (renderAttackRow a)
                                  }
                                ] ] ]
                      footer [ Class "card-footer" ]
                        [ a [ Href "#"
                              Class "card-footer-item" ]
                            [ str "Play" ]
                          a [ Href "#"
                              Class "card-footer-item" ]
                            [ str "Discard" ] ] ] ]
```

Now I have to plug in the information into the player and enemy creature hopefully reusing much of the previously created functions.

First, I can plug in the playmat backgrounds.

Then, I see I can use a mapping function for the `SpecialCondition`s to status symbols similar to what was done for the Resource symbols as well a function which takes an optional list of these conditions and appends them together. *These mappings do rely on emojis, this is not a great idea but I am sticking with it for now*.

Like:
```
let getSymbolForSpecialCondition status =
    match status with
    | Asleep -> "💤"
    | Burned -> "♨"
    | Confused -> "❓"
    | Paralyzed -> "🧊"
    | Poisoned -> "☠️"

let textDescriptionForListOfSpecialConditions specialConditions =
    match specialConditions with
    | Some sc -> sc |> Seq.map getSymbolForSpecialCondition |> String.concat ";"
    | None -> ""
```

I am starting to have a number of UI helper functions so I am going to move them into another file/module `GeneralUIHelpers`. I will have to register this file in Client.fsproj making sure it is compiled before the PageLayoutElement.fs.


I can now pull out a `renderEnemyActiveCreature` function which takes an optional `InPlayCreature` as well as a `renderEnemyBench` which takes an optional list of `InPlayCreature`s.

I wrote these and modified the enemyCreatures function to be like:

```

let renderEnemyActiveCreature (inPlayCreature : Option<InPlayCreature>) =
  match (Option.map (fun x -> (x, x.Card)) inPlayCreature) with
  | None -> strong [] [ str "No creature in active play" ]
  | Some (creature, EffectCard card) ->  strong [] [ str "Active creature is not a character?" ]
  | Some (creature, ResourceCard card) -> strong [] [ str "Active creature is not a character?" ]
  | Some (creature, CharacterCard card) ->
        a [ Href "#" ]
                    [ div [ Class "card" ]
                        [ header [ Class "card-header" ]
                            [ p [ Class "card-header-title" ]
                                [ str (sprintf "✫ %s" card.Name) ]
                              p [ Class "card-header-icon" ]
                                [ str (sprintf "💓 %i/%i" (card.Creature.Health - creature.CurrentDamage) card.Creature.Health)
                                  str (textDescriptionForListOfSpecialConditions creature.SpecialEffect) ] ]
                          div [ Class "card-image" ]
                            [ figure [ Class "image is-4by3" ]
                                [ img [ Src (card.ImageUrl.ToString())
                                        Alt card.Name
                                        Class "is-fullwidth" ] ] ]
                          div [ Class "card-content" ]
                            [ div [ Class "content" ]
                                [ p [ Class "is-italic" ]
                                    [ str card.Description ]
                                  h5 [ Class "IsTitle is5" ]
                                    [ str "Attacks" ]
                                  table [ ]
                                    [
                                        yield! seq {
                                            for a in card.Creature.Attach do
                                                (renderAttackRow a)
                                        } ] ] ] ] ]

let renderEnemyBenchRow inPlayCreature =
    match inPlayCreature.Card with
    | EffectCard ec -> strong [] [ str "Bench creature is not a character code" ]
    | ResourceCard rc -> strong [] [ str "Bench creature is not a character code" ]
    | CharacterCard card ->
        tr [ ]
                            [ td [ ]
                                [ str card.Name ]
                              td [ ]
                                [ str (textDescriptionForListOfSpecialConditions inPlayCreature.SpecialEffect) ]
                              td [ ]
                                [ str (sprintf "💓 %i/%i" (card.Creature.Health - inPlayCreature.CurrentDamage) card.Creature.Health)  ] ]



let renderEnemyBench bench  =
    match bench with
    | None -> strong [] [ str "No creatures on bench" ]
    | Some  l->
                      table [ Class "table is-fullwidth" ]
                        [ tr [ ]
                            [ th [ ]
                                [ str "Creature Name" ]
                              th [ ]
                                [ str "Status" ]
                              th [ ]
                                [ str "Health" ] ]
                          yield! seq {
                                for b in l do
                                renderEnemyBenchRow b
                          }
                        ]

let enemyCreatures  (player: Player) (playerBoard: PlayerBoard) =
  section [
          Class "section"
          Style [ Background (sprintf "url(%s')" (player.PlaymatUrl.ToString()))
                  BackgroundSize "cover"
                  BackgroundRepeat "no-repeat"
                  BackgroundPosition "center center" ] ]
    [ div [ Class "container py-r" ]
        [ div [ Class "columns" ]
            [ div [ Class "column is-1" ]
                [ ]
              div [ Class "column is-3" ]
                [ renderEnemyActiveCreature playerBoard.ActiveCreature ]
              div [ Class "column is-7" ]
                [ div [ Class "columns is-mobile is-multiline" ]
                    [ h2 [ Class "title is-4" ]
                        [ str "Bench" ]
                      renderEnemyBench playerBoard.Bench
                    ] ] ] ] ]
```

This now builds (and looks very weird with no content).

I now need to populate some test data for the enemy creatures.

Similar to the cards I am able to generate the data like:

```

let inPlayCreatureGenerator inPlayCreatureIdStr cardInstanceIdStr cardIdStr cardImageUrlStr =
        let inPlayCreatureId = NonEmptyString.build inPlayCreatureIdStr |> Result.map InPlayCreatureId
        let card = testCardGenerator cardInstanceIdStr cardIdStr cardImageUrlStr

        match inPlayCreatureId, card with
        | Ok id, Ok c ->
            Ok {
                InPlayCharacterId=  id
                Card = c.Card
                CurrentDamage=  0
                SpecialEffect=  None
                AttachedEnergy = [ Resource.Grass, 4;
                                     Resource.Colorless, 1 ] |> Seq.ofList |> ResourcePool
                SpentEnergy = [ Resource.Grass, 4;
                                     Resource.Colorless, 1 ] |> Seq.ofList |> ResourcePool
            }
        | _, _ -> "Unable to create in play creature." |> Error


let inPlayCreatureSeqGenerator (numberOfCards : int) =
    seq { 0 .. (numberOfCards - 1) }
    |> Seq.map (fun x -> inPlayCreatureGenerator
                            (sprintf "ExcitingCharacter%i" x)
                            (sprintf "ExcitingCharacter%i" x)
                            (sprintf "Exciting Character #%i" x)
                            (sprintf "https://picsum.photos/320/200?%i" x))
    |> Seq.map (fun x ->
                        match x with
                        | Ok s -> [ s ] |> Ok
                        | Error e -> e |> Error)
    |> Seq.fold (fun x y ->
                    match x, y with
                    | Ok accum, Ok curr -> curr @ accum |> Ok
                    | _,_ -> "Eating Errors lol" |> Error
                ) (Ok List.empty)
```

Next, I need to modify the player creatures to also reference the state.

To do this I created three functions:

```

let playerActiveCreature (inPlayCreature : Option<InPlayCreature>) =
  match (Option.map (fun x -> (x, x.Card)) inPlayCreature) with
  | None -> strong [] [ str "No creature in active play" ]
  | Some (creature, EffectCard card) ->  strong [] [ str "Active creature is not a character?" ]
  | Some (creature, ResourceCard card) -> strong [] [ str "Active creature is not a character?" ]
  | Some (creature, CharacterCard card) ->
        a [ Href "#" ]
                    [ div [ Class "card" ]
                        [ header [ Class "card-header" ]
                            [ p [ Class "card-header-title" ]
                                [ str (sprintf "✫ %s" card.Name) ]
                              p [ Class "card-header-icon" ]
                                [ str (sprintf "💓 %i/%i" (card.Creature.Health - creature.CurrentDamage) card.Creature.Health)
                                  str (textDescriptionForListOfSpecialConditions creature.SpecialEffect) ] ]
                          div [ Class "card-image" ]
                            [ figure [ Class "image is-4by3" ]
                                [ img [ Src (card.ImageUrl.ToString())
                                        Alt card.Name
                                        Class "is-fullwidth" ] ] ]
                          div [ Class "card-content" ]
                            [ div [ Class "content" ]
                                [ p [ Class "is-italic" ]
                                    [ str card.Description ]
                                  h5 [ Class "IsTitle is5" ]
                                    [ str "Attacks" ]
                                  table [ ]
                                    [
                                        yield! seq {
                                            for a in card.Creature.Attach do
                                                (renderAttackRow a)
                                        } ] ] ]
                          footer [ Class "card-footer" ]
                            [ a [ Href "#"
                                  Class "card-footer-item" ]
                                [ str "Tap Out" ] ] ] ]

let playerBenchCreature (inPlayCreature : InPlayCreature)=
  match (inPlayCreature, inPlayCreature.Card) with
  | (creature, EffectCard card) ->  strong [] [ str "Active creature is not a character?" ]
  | (creature, ResourceCard card) -> strong [] [ str "Active creature is not a character?" ]
  | (creature, CharacterCard card) ->
         div [ Class "column is-4-mobile is-12-tablet" ]
                    [ div [ Class "card" ]
                        [ header [ Class "card-header" ]
                            [ p [ Class "card-header-title" ]
                                [ str card.Name ]
                              p [ Class "card-header-icon" ]
                                [ str (sprintf "💓 %i/%i" (card.Creature.Health - creature.CurrentDamage) card.Creature.Health)
                                  str (textDescriptionForListOfSpecialConditions creature.SpecialEffect) ] ]
                          div [ Class "card-image" ]
                            [ figure [ Class "image is-4by3" ]
                                [ img [ Src (card.ImageUrl.ToString())
                                        Alt card.Name
                                        Class "is-fullwidth" ] ] ]
                          div [ Class "card-content" ]
                            [ div [ Class "content" ]
                                [ p [ Class "is-italic" ]
                                    [ str card.Description ]
                                  h5 [ Class "IsTitle is5" ]
                                    [ str "Attacks" ]
                                  table [ ]
                                    [
                                        yield! seq {
                                            for a in card.Creature.Attach do
                                                (renderAttackRow a)
                                        } ] ] ] ] ]

let playerBench (bench : Option<InPlayCreature list>) columnsInBench =
    match bench with
    | None -> strong [] [ str "No creatures on bench."]
    | Some l ->
      div [ Class "column is-9"] [
        div [ Class "container"] [
          h3 [Class "title is-3"] [str "Bench"]
          yield! seq {
            let numberOfRows = (l.Length / columnsInBench)
            for row in {0 .. numberOfRows }  do
                let innerl = Seq.truncate columnsInBench (Seq.skip (row * columnsInBench) l)

                div [ Class "columns is-mobile is-multiline" ]
                    [
                       yield! seq {
                           for creature in innerl do
                            div [ Class (sprintf "column is-%i" (12/columnsInBench)) ] [
                                playerBenchCreature creature
                            ]
                       }
                    ]
        } ] ]


let playerCreatures  (player: Player) (playerBoard: PlayerBoard) =
  section [
          Class "section"
          Style [ Background (sprintf "url('%s')" (player.PlaymatUrl.ToString()))
                  BackgroundSize "cover"
                  BackgroundRepeat "no-repeat"
                  BackgroundPosition "center center" ] ]
    [ div [ Class "container py-r" ] [
          div [ Class "columns" ]
            [ div [ Class "column is-3" ]
                [ playerActiveCreature playerBoard.ActiveCreature ]
              playerBench playerBoard.Bench 4

               ] ] ]
```

This compiles and displays now. Here note that the playerBench takes an argument for the number of columns to display. I wasn't sure if it should be 3 or 4 cards per column so I made it a variable.

The playerCreatures can then be changed to

```

let playerCreatures  (player: Player) (playerBoard: PlayerBoard) =
  section [
          Class "section"
          Style [ Background (sprintf "url('%s')" (player.PlaymatUrl.ToString()))
                  BackgroundSize "cover"
                  BackgroundRepeat "no-repeat"
                  BackgroundPosition "center center" ] ]
    [ div [ Class "container py-r" ] [
          div [ Class "columns" ]
            [ div [ Class "column is-3" ]
                [ playerActiveCreature playerBoard.ActiveCreature ]
              playerBench playerBoard.Bench 4

               ] ] ]
```

Additionally, at this stage, I noticed that the playmat background on the creatures was missing a `'` around the URL so I added that.

Now our layout is largely wired up to the GameState. Next, we will need to define some events which modify the GameState and attach those events to the UI.

This is the final commit in the branch `step-6-wire-up-layout-to-gamestate-part-2`

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-6-wire-up-layout-to-gamestate-part-2)