---
title: More Complex Port Layout
layout: default
parent: Tutorials
nav_order: 3
---

# Node Types

There are many kinds of nodes in SimPort, here are examples of each of them:

## Arrivals

Objects are generated at given intervals:

```kotlin
val pushOutput = arrivals("Truck Arrivals", Generators.constant(::Truck, Delays.exponentialWithMean(1.hours)))
```

## Sinks

Objects "leave" the network:

```kotlin
input
    .thenSink("Truck Departures")
```

## Delays

Objects coming in are emitted after a given delay:

```kotlin
val pushOutput =
    input
        .thenDelay("Road", Delays.exponentialWithMean(10.minutes))
```

## Services

Objects are delayed but can only be served 1 at a time:

```kotlin
val pushOutput =
    input
        .thenService("Passport Check", Delays.exponentialWithMean(10.minutes))
```

## Forks

Objects are forwarded to one of multiple outputs:

```kotlin
// Push forks choose their destination via a policy, random by default
val pushOutputs =
    pushInput
        .thenFork("Truck Split", numLanes = 5, policy = forkPolicy(RandomPolicy())) { i, lane ->
            lane.thenDelay("Road $i", Delays.fixed(i.seconds))
        }
// Instead of a number of lanes, you can also provide a list of lanes, each of which is a lambda to build the lane
val pushOutputs =
    pushInput
        .thenFork(
            "Truck Split",
            lanes = listOf(
                { it.thenService("Short", Delays.fixed(1.minutes)) },
                { it.thenService("Long", Delays.fixed(10.minutes)) }
            ),
            policy = ...
)

// Pull forks instead forward objects as they are requested by lanes, and as such have no policy
// In this example, each service will pull from the input when it is ready
val pullOutputs =
    pullInput
        .thenFork("Truck Split", numLanes = 5) { i, lane ->
            lane.thenService("Service $i", Delays.fixed(i.seconds))
        }
```

## Joins

Objects are combined back into a single output:

```kotlin
// Push joins simply accept items as they arrive and as such have no policy
val pushOutput =
    pushInputs
        .thenJoin("Truck Join")

// Pull joins choose from the available inputs when their downstream requests an item, and as such do have a policy, 
// random by default
val pullOutput =
    pullInputs
        .thenJoin("Truck Join", policy = joinPolicy(RandomPolicy()))
```

## Matches

Objects are combined as per some function when both ready:

```kotlin
val output =
    // The main input can be either a push or a pull, and the output will match
    input
        .thenMatch(
            "Container Mount",
            // The side input must be a pull
            pullInput
        ) { vehicle, container ->
            // Somehow produce a new object from the two objects
            VehicleWithContainer(vehicle, container)
        }
```

## Splits

Objects are split into 2 outputs as per some function:

```kotlin
val (output, pushOutput) =
    // The main input can be either a push or a pull, and the output will match
    input
        .thenSplit("Container Unmount") { vehicleWithContainer ->
            // Somehow produce a Pair of objects from the object
            Pair(vehicleWithContainer.vehicle, vehicleWithContainer.container)
        }
```

## Queues

Objects are queued and can be extracted by the downstream in an order specified by the policy (FIFO by default):

```kotlin
val pullOutput =
    pushInput
        .thenQueue("Queue", policy = FIFOQueuePolicy())
```

## Subnetworks

Subnetworks have a fixed capacity and are an example of a *compound node*, which you can make yourself (LINK???).
Internally, they use a queue of Tokens, along with Match and Split nodes, to implement their behaviour.

```kotlin
val output =
    input
        .thenSubnetwork("Car Park", capacity = 10) { inner ->
            // Only 10 trucks can be in the park at once
            inner
                .thenQueue("Leaving Queue")
                .thenService("Ticket Gate", Delays.exponentialWithMean(10.seconds))
        }
```

# Helpers

Slightly simpler than compound nodes, you can make regular helper functions to extract new components:

```kotlin
// Here, we define that we accept either a pull or push input of a generic item
// type, and we will give back a Push output of the same type.
fun <T> NodeBuilder<T, *>.thenCarPark(capacity: Int): NodeBuilder<T, ChannelType.Push> =
    this
        .thenSubnetwork("Car Park", capacity = capacity) { inner ->
            // Only 10 trucks can be in the park at once
            inner
                .thenQueue("Leaving Queue")
                .thenService("Ticket Gate", Delays.exponentialWithMean(10.seconds))
        }
```

which can be used as normal:

```kotlin
val pushOutput = 
    pushInput
        .thenCarPark(capacity = 10)
```