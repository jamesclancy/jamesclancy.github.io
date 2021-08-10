---
layout: post
title: SafeCardGame - Step 12 - Implementing Attack
author: James Clancy
tags: fsharp dotnet
---

## Implementing Attack

The next step will be to implement the attack functionality.

To do this I implemented several new functions:

```
let getPlayBoardToTargetAttack (playerId : PlayerId) gs =
    playerId
    |> gs.Boards.TryGetValue
    |> function
        | true, pb -> pb |> Ok
        | _, _ -> "Unable to locate target for attack" |> Error

let activeCreatureKilledFromPlayerBoard playBoard :PlayerBoard =
    match playBoard.Bench with
    | None | Some [] -> { playBoard with ActiveCreature = None}
    | Some [ x ] ->  { playBoard with ActiveCreature = Some x; Bench = None}
    | Some (x :: xs) -> { playBoard with ActiveCreature = Some x; Bench = Some xs}

let applyBasicAttackToPlayBoard (attack : Attack) (playBoard) gs =
        match playBoard.ActiveCreature with
        | Some cre when (cre.CurrentDamage + attack.Damage) < cre.TotalHealth ->
              ({
                playBoard with ActiveCreature =
                                    Some { cre with CurrentDamage = cre.CurrentDamage  + attack.Damage }
              }, 0, sprintf "%i damage dealt to %s" attack.Damage cre.Name)
        | Some cre ->
            ((activeCreatureKilledFromPlayerBoard playBoard), 0, sprintf "%i damage dealt to %s. It died." attack.Damage cre.Name)
        | None ->
            (playBoard, attack.Damage, sprintf "%i damage dealt to player" attack.Damage)


let applyPlayerDamageToPlayer (playerId : PlayerId) damage (gs: GameState) =
    let player = gs.Players.TryGetValue playerId
    match player with
    | true, p -> { gs with Players = gs.Players.Add(playerId, {p with RemainingLifePoints = p.RemainingLifePoints - damage})}
    | _,_ -> gs


let playAttackFromBoard (attack : Attack) (playerId : PlayerId) (gs: GameState) (playerBoard : PlayerBoard) =

    let target = getTheOtherPlayer gs playerId
    let otherBoard = getPlayBoardToTargetAttack target gs

    match otherBoard with
    | Ok playBoard ->

        let (newPb, playerDamage, messages) =  (applyBasicAttackToPlayBoard attack playBoard gs)

        { gs with Boards = (gs.Boards.Add (target, newPb)  ) }
        |> applyPlayerDamageToPlayer target playerDamage
        |> appendMessagesToGameState messages |>Ok
    | Error e ->
        Error e



let modifyGameStateFromPerformAttackEvent (ev: PerformAttackEvent) (gs: GameState) =
        getExistingPlayerBoardFromGameState ev.PlayerId gs
        >>= (playAttackFromBoard ev.Attack ev.PlayerId gs)
        >>= migrateGameStateToNewStep (ev.PlayerId |> Reconcile )
        |> applyErrorResultToGamesState gs
```

I then reference this `modifyGameStateFromPerformAttackEvent` in the update function.

This builds. I will now have to make sure my sample data has attacks and that the UI is wired up to trigger these events.


To wire up the UI I am able to modify the `renderAttackRow` to also include a button to exec an attack if it is available like:

```
let renderDamageInformationForAttack (attack: Attack) =
    match attack.SpecialEffect with
    | Some se ->
        td [ ]
                [
                    p [] [ str (sprintf "%i" attack.Damage) ]
                    p [] [ str se.Description ]
                ]
    | None ->
        td [ ]
                [
                    p [] [ str (sprintf "%i" attack.Damage) ]
                ]


let renderAttackRowWithoutActions (attack: Attack) =
        tr [ ]
            [ td [ ]
                [ str (textDescriptionForResourcePool attack.Cost) ]
              td [ ]
                [ str attack.Name ]
              renderDamageInformationForAttack attack ]

let renderAttackRow displayAttackButton canAttack availableResources gameId playerId inPlayCreatureId (attack: Attack) dispatch = // Lol this needs to be refactored
    let execAttack =  (fun _ ->
                                                    ({
                                                        GameId = gameId
                                                        PlayerId = playerId
                                                        InPlayCreatureId = inPlayCreatureId
                                                        Attack = attack
                                                    } :  PerformAttackEvent) |> PerformAttack |>  dispatch)

    let displayAttackButton = displayAttackButton && canAttack && (hasEnoughResources availableResources (Map.toList attack.Cost))

    tr [ ]
            [
              td [ ]
                [ str (textDescriptionForResourcePool attack.Cost) ]
              td [ ]
                [ str attack.Name ]
              renderDamageInformationForAttack attack
              td [ ]
                [
                    if displayAttackButton then
                        button [
                            Class "is-danger"
                            OnClick execAttack
                        ]
                            [
                                str "Exec"
                            ]
                    else
                        str ""
                ] ]
```

To populate the test database I am adding a "tackle" attack to all generated creatures in the `SampleCardDatabase`

```
let creatureCreatureConstructor creatureId name description primaryResource resourceCost health weaknesses =
    let cardId = NonEmptyString.build creatureId |> Result.map CardId
    let cardImageUrl = ImageUrlString.build (sprintf "/images/full/%s.png" creatureId)

    match  cardId, cardImageUrl with
    |  Ok cid, Ok imgUrl ->
        Ok {
                CardId = cid
                ResourceCost =  resourceCost
                Name = name
                EnterSpecialEffects = None
                ExitSpecialEffects = None
                PrimaryResource = primaryResource
                Creature =
                      {
                        Health= health
                        Weaknesses=  weaknesses
                        Attack = [
                                    {
                                        Damage = 5
                                        Name = "Tackle"
                                        Cost = Seq.empty |> ResourcePool
                                        SpecialEffect = None
                                    }
                                 ]
                      }
                ImageUrl = imgUrl
                Description =description
            }
    | _,_ -> Error "No"
```
I am now able to test the game. I can deal damage to creatures and players. It works in terms of basic functionality.

I am leaving this as the last commit in the branch `step-12-implement-attack`.

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-12-implement-attack)