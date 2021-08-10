---
layout: post
title: SafeCardGame - Step 2 - Domain Model
author: James Clancy
tags: fsharp dotnet
---

## Game Rules
---

My idea would be that scoping out the rules for the game might be the best first step.

### Defining the basic Domain Model and Terms
---

I think defining the domain models might be the best way to start documenting the rules.  The domain model should correlate to the object defined in the shared path of the project (i.e. src/shared). For now, I am just defining the types in the existing `Shared` module found in the Shared.fs.


The game starts with two players.

Each `Player` will start with a name and an amount of health.

```
type PlayerId = string

type Player =
    {
        PlayerId: PlayerId
        Name: string
        RemainingLifePoints: int
    }
```

Each player will have a `board` which consists of a Deck of cards, a hand, a pile of discarded, an optional active creature, a list of in play but not active creatures (i.e. a bench), a total resource pool and an available resource pool.

```
type PlayerBoard =
    {
        PlayerId: PlayerId
        Deck: Deck
        Hand: Hand
        ActiveCreature: Option<InPlayCreature>
        Bench:  Option<InPlayCreature list>
        DiscardPile: Deck
        TotalResourcePool: ResourcePool
        AvailableResourcePool: ResourcePool
    }

```

`Resource`s are typed from a finite list, i.e.:
```
type Resource =
    Grass | Fire |
    Water | Lightning |
    Psychic | Fighting |
    Colorless
```

A `ResourcePool` is simply a list of the different types of resources along with a quantity value.

```
type ResourcePool = Map<Resource, int>
```

A `Deck` and a `Hand` are both lists of `CardInstance`s, the main difference being that potentially a deck could have a number of cards exposed or visible.
```
type Hand =
    {
        Cards: CardInstance list
    }
type Deck =
    {
        Cards: CardInstance list
        TopCardsExposed: int
    }
```

Each `CardInstance` will contain a unique CardInstanceId and refers to a generic card referenced via a CardId. I am also placing the Card Type on the CardInstance, this seems pretty memory inefficient but probably doesn't matter due to the limited size of decks. For the moment it makes it much easier to manipulate the decks and pull information.
The `Card` referenced by the `CardId` will have:
* a name
* a resource cost (a `ResourcePool`)
* a Primary resource type for determining weaknesses
* a classification under one of three different types
    * `ResourceCard` - Also contains a `ResourcePool` for `AddedResource` and a bool for `ResourceAvailableOnFirstTurn`
    * `CreatureCard` - Also contains a `Creature`
    * `EffectCard`
* a `GameStateSpecialEffect` for when the card enters and leaves the game
    * the `GameStateSpecialEffect` will consist of a function which takes an arguement of the `GameState` and returns a new manipulated `GameState`

```
type CardInstanceId = string
type CardId = string
type GameStateSpecialEffect = delegate of GameState -> GameState
type CardInstance =
    {
        CardIntanceId : CardInstanceId
        CardId: CardId
        Card: Card
    }
type Card =
   CharacterCard of { CardId: CardId; Name: string; Creature: Creature; ResourceCost: ResourcePool; PrimaryResource: Resource; EnterSpecialEffects: GameStateSpecialEffect; ExitSpecialEffects: GameStateSpecialEffect}
   | EffectCard of { CardId: CardId; Name: string; ResourceCost: ResourcePool; PrimaryResource: Resource; EnterSpecialEffects: GameStateSpecialEffect; ExitSpecialEffects: GameStateSpecialEffect; }
   | ResourceCard of { CardId: CardId; Name: string; ResourceCost: ResourcePool; PrimaryResource: Resource; EnterSpecialEffects: GameStateSpecialEffect; ExitSpecialEffects: GameStateSpecialEffect; ResourceAvailableOnFirstTurn: bool; ResourcesAdded: ResourcePool}
```

A `Creature` will have integer health for total health associated with the card, a list of `Resource`s the creature is weak to, and a list of `Attacks.
An `Attack` will have an integer for the damage it inflicts, a `ResourcePool` representing the cost to use, and a `GameStateSpecialEffect` defining any additional effects which may be triggered by the attack.
```
type Creature =
    {
        Health: int
        Weaknesses: Resource list
        Attacks: Attack list
    }
type Attack =
    {
        Damage: int
        Cost: ResourcePool
        SpecialEffect: GameStateSpecialEffect
    }
```
An `InPlayCreature` has an id, is associated with a `CardInstance`, a current amount of damage, an optional list of `SpecialCondition`s currently applied as well as attached and spent resource pools.
The `InPlayCreatureId` is a string and available `SpecialCondition`s are `Asleep | Burned | Confused | Paralyzed | Poisoned`

```
type SpecialCondition = Asleep | Burned | Confused | Paralyzed | Poisoned
type InPlayCreatureId = string
type InPlayCreature =
    {
        InPlayCharacterId: InPlayCharacterId
        Card: Card
        CurrentDamage: int
        SpecialEffect: Option<SpecialCondition list>
        AttachedEnergy: ResourcePool
        SpentEnergy: ResourcePool
    }
```

Lastly, the `GameState` contains a:
* Map of `PlayerId`s to `Player`s
* Map of boards for each player, a map from `PlayerId` to `PlayerBoard`
* A current turn which is an optional `PlayerId representing the player whose turn it currently is.
* A current step which is a `GameStep` representing the current step the player is on in their turn.
* Finally, it contains a turn number representing the number of turns that have passed since the beginning of the game.

The GameStep is just an enum represented by `NotCurrentlyPlaying | Draw | Play | Attack | Reconcile`

```

type GameStep =
    NotCurrentlyPlaying | Draw | Play | Attack | Reconcile

type GameState =
    {
        Players: Map<PlayerId, Player>
        Boards: Map<PlayerId, PlayerBoard>
        CurrentTurn: Option<PlayerId>
        CurrentStep:GameStep
        TurnNumber: int
    }
```


If you look at the current Shared.fs file you can see that many of the `type` declarations have been transformed into `and`s this is because F# cares about the order in which things are declared. The `type` declares a thing in a subsequent order and the `and` keyword forces the items to be declared at the same time.

Running `dotnet fake build --target run` the application **FAILED to Build** with an error message of
```
  Restored C:\tests\SafeCardGame\src\Shared\Shared.fsproj (in 197 ms).
C:\tests\SafeCardGame\src\Shared\Shared.fs(51,21): error FS0035: This construct is deprecated: Consider using a separate record type instead [C:\tests\SafeCardGame\src\Shared\Shared.fsproj]
C:\tests\SafeCardGame\src\Shared\Shared.fs(52,20): error FS0035: This construct is deprecated: Consider using a separate record type instead [C:\tests\SafeCardGame\src\Shared\Shared.fsproj]
C:\tests\SafeCardGame\src\Shared\Shared.fs(53,22): error FS0035: This construct is deprecated: Consider using a separate record type instead [C:\tests\SafeCardGame\src\Shared\Shared.fsproj]

Build FAILED.

C:\tests\SafeCardGame\src\Shared\Shared.fs(51,21): error FS0035: This construct is deprecated: Consider using a separate record type instead [C:\tests\SafeCardGame\src\Shared\Shared.fsproj]
C:\tests\SafeCardGame\src\Shared\Shared.fs(52,20): error FS0035: This construct is deprecated: Consider using a separate record type instead [C:\tests\SafeCardGame\src\Shared\Shared.fsproj]
C:\tests\SafeCardGame\src\Shared\Shared.fs(53,22): error FS0035: This construct is deprecated: Consider using a separate record type instead [C:\tests\SafeCardGame\src\Shared\Shared.fsproj]
    0 Warning(s)
    3 Error(s)
```

It appears that newer versions of F# no longer allow anonymous types in discriminated unions so I had to break out those types like:
```
type Card =
   CharacterCard of CharacterCard
   | EffectCard of EffectCard
   | ResourceCard of ResourceCard
and CharacterCard
    = { CardId: CardId; Name: string; Creature: Creature; ResourceCost: ResourcePool; PrimaryResource: Resource; EnterSpecialEffects: GameStateSpecialEffect; ExitSpecialEffects: GameStateSpecialEffect}
and EffectCard
    = { CardId: CardId; Name: string; ResourceCost: ResourcePool; PrimaryResource: Resource; EnterSpecialEffects: GameStateSpecialEffect; ExitSpecialEffects: GameStateSpecialEffect; }
and ResourceCard
    = { CardId: CardId; Name: string; ResourceCost: ResourcePool; PrimaryResource: Resource; EnterSpecialEffects: GameStateSpecialEffect; ExitSpecialEffects: GameStateSpecialEffect; ResourceAvailableOnFirstTurn: bool; ResourcesAdded: ResourcePool}

```

Now running `dotnet fake build --target run` builds.

[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-2-domain-model)