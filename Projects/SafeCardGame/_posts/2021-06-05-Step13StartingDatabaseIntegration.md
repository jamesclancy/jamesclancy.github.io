---
layout: post
title: SafeCardGame - Step 13 - Pushing Decks, Players, and Cards into a Database Part 1
author: James Clancy
tags: fsharp dotnet
---

Next, I would like a better way to come up with more complicated and complete sets of decks, players, and cards.

## Pushing Decks, Players, and Cards into a Database

I am not going to try to start pulling the decks, player info, and cards from a database.

The first thing I am going to do for this is moving the `SampleCardDatabase` to the Shared project from the Client project.

Now I am researching ways to store the information on the server-side.

The first step appears to be mapping to and from a DTO. To start this I added a Dto module to the Shared project.

I am going to be creating DTOs for each of the domain types and writing functions to map from the DTOs to the Domain models.

I started doing this with the idea of documenting the process I was going through. This turned into quite a bit of code but generally, I was creating Dtos for each domain type and adding a module for that class with `fromDomain` and `toDomain` functions which map to the DTO and to a Result<the domain type, string> type. I implemented them like

```
module Player =

    let fromDomain (person:Domain.Player) : PlayerDto =
       {
           PlayerId = person.PlayerId.ToString()
           Name = person.Name
           PlaymatUrl = person.PlaymatUrl.ToString()
           LifePoints = person.RemainingLifePoints
       }: PlayerDto

    let toDomain (dto: PlayerDto) :Result<Domain.Player,string> =
        CollectionManipulation.result {
            let! playerId = dto.PlayerId |> NonEmptyString.build
            let! name = dto.Name |> Ok
            let! playmatUrl = dto.PlaymatUrl |> ImageUrlString.build
            let! lifePoints = dto.LifePoints |> Ok

            return {
               PlayerId = playerId |> PlayerId
               Name = name
               PlaymatUrl = playmatUrl
               RemainingLifePoints = lifePoints
            }
        }
```

You can note that the `toDomain` function utilizes a computational expression I had to added.

the computational expression is defined like:

```
type ResultBuilder() =
    member __.Return(x) = Ok x
    member __.ReturnFrom(m: Result<_, _>) = m
    member __.Bind(m, f) = Result.bind f m


let result = new ResultBuilder()
```

This `ResultBuilder` allows multiple result checks to chained together using a series of `let!` bind operations.


I was able to test serializing these DTOs using System.Text.Json by adding code to the Server project

```
SampleCardDatabase.creatureCardDb |> Seq.map (fun c-> CharacterCard c)
|> Seq.map Card.fromDomain
|> Seq.map (fun c ->
                    System.Console.Write(c)
                    c
)
|> JsonSerializer.Serialize
|> (fun c-> System.IO.File.WriteAllText("test.json", c))
|> ignore
```

It did serialize but I noticed the CardId and other Domain Ids needed to have their `ToString()` methods overloaded like:

```
module Domain =

    type PlayerId = PlayerId of NonEmptyString
        with override this.ToString() = match this with PlayerId s -> s.ToString()
    type CardInstanceId = CardInstanceId of NonEmptyString
        with override this.ToString() = match this with CardInstanceId s -> s.ToString()
    type CardId = CardId of NonEmptyString
        with override this.ToString() = match this with CardId s -> s.ToString()
    type InPlayCreatureId = InPlayCreatureId of NonEmptyString
        with override this.ToString() = match this with InPlayCreatureId s -> s.ToString()
    type GameId = GameId of NonEmptyString
        with override this.ToString() = match this with GameId s -> s.ToString()
        ...
```

I am leaving this as the final commit in the branch `step-13-decks-to-database`. We were unable to actually write to a database but are able to serialize a list of cards to JSON now. This actually took an inordinate amount of time (like a week of share time) so I will continue on with this in step 14.


[Git branch for this step](https://github.com/jamesclancy/SafeCardGame/tree/step-13-decks-to-database)