---
title: Comparing Simulations
layout: default
parent: Tutorials
nav_order: 2
---

# Comparing Simulations

## Defining a parametric port

Let's define a port which can be customised parametrically:

```kotlin
// This represents an object in our simulation
class Truck

fun examplePort2(
    // We allow the number of lanes to be customised
    numberOfLanes: Int
): Scenario = buildScenario {
    arrivals(
        "Truck Arrivals",
        Generators.constant(::Truck, Delays.exponentialWithMean(4.minutes)),
    )
        // We split the trucks into `numberOfLanes` lanes:
        .thenFork("Truck Split", numLanes = numberOfLanes) { i, lane ->
            // The lanes each have the trucks queue for a long service
            lane.thenQueue(label = "Long Service Queue $i")
                .thenService(label = "Long Service $i", Delays.exponentialWithMean(30.minutes))
        }
        // The trucks then merge back into 1 lane...
        .thenJoin(label = "Truck Join")
        // ...and leave the simulation
        .thenSink(label = "Truck Departures")
}.withMetrics {
    // Track the occupancy of each Queue
    trackAll<Queue<*>>(Occupancy)
}
```

This gives the following port layout:

![img.png](../../assets/img.png)

which largely looks as we'd expect, except for the `Pump` nodes, which we will now discuss.

Connections between nodes are called "channels". We have two types of channels:

- Push channels are driven by the upstream node sending things downstream
- Pull channels are driven by the downstream node requesting things from upstream

| ![push-channel-symbol](../../assets/push-channel-symbol.png) | ![pull-channel-symbol](../../assets/pull-channel-symbol.png) |
|--------------------------------------------------------------|--------------------------------------------------------------|
| Push Channel Symbol                                          | Pull Channel Symbol                                          |

`Pump` nodes simply convert from a pull channel to a push channel, by repeatedly requesting items from its input and
sending them downstream. For convenience, they are inserted automatically when making a connection from a pull output to
a push input.

The inverse concept is a `Queue`, which converts from a push to a pull. Items are pushed into a queue and remain there
until the downstream requests them. `Queue`s are, by contrast, *not* inserted automatically, since we believe they
should be explicit.

## Running a simulation

```kotlin
fun main() {
    // Option 1: Run a simulation for a set duration and inspect its results
    runSimulation(demoPort(), 20.days)
    
    // Option 2: Run the simulation live in the GUI and see results as they happen
    runLiveSimulation(demoPort())
}
```

## Looking at the metrics 



