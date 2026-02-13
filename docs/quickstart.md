---
title: Getting Started
layout: default
nav_order: 2
---
# Getting Started

This is a SimPort quickstart for developing an intuition for SimPort and setting your first port up and running. For more details, see the rest of these docs and our [tutorials](/docs/tutorials).

## Setup

SimPort is a [Kotlin](https://kotlinlang.org) library, so you'll need to know some Kotlin to use this.

Once you have a kotlin project created, simply depend on SimPort to start using it in your project.

```kotlin
TKTK
```

### Project flow

In SimPort, you design and write simulations for your ports in code, then, you can check and inspect the layout as well as see the results of simulations in our visualisations.

This means you get all the benefits of the power and flexibility of designing ports with code *and* all the ease of inspecting a simulations and data with visualisations.

But be prepared: if you were expecting a pretty visual editor for laying out ports, we don't have one!

## Port Layout

A designed port in Simport is called a `Scenario`. Every port needs a source node (arrivals) where objects can enter the port, and a sink where objects can end their journey.

The simplest way to make a scenario is with `buildScenario`:

```kotlin
// imports TKTK

val Scenario = buildScenario {
    // Port specs...
}
```

SimPort has a bunch of builder and helper functions you can use which form our [Domain Specific Language](/docs/api/dsl) (DSL). All of the building blocks of a queue network are available using from functions like `thenNodeName`.

To begin with let's make a port with just an arrivals node, a queue, and then a sink:

```kotlin
// imports TKTK

data object Truck

val Scenario = buildScenario {
    arrivals(
        "Truck arrivals",
        // A Generator to determine when things arrive
        Generators.constant(Truck, Delays.exponential(50.0, DurationUnit.HOURS))
    )
        .thenQueue("Truck arrival queue")
        .thenSink("Truck departures")
}
```

Notice the arrivals node needs a [Generator](/docs/api/generators), which defines what arrives and how often. 

## Running a simulation

Now we've designed our port, we want to run a simulation. To do this we need to create a `Simulator`. We also need a logger to log events and a sampler to sample metrics. 

For now, we don't need to log events so we can use `EventLog.noop()`. `MetricsPanelState` is a sampler that works with our visualisation, so let's use that.

```kotlin
// imports TKTK

val sampler = MetricsPanelState(scenario)
val simulator = Simulator(EventLog.noop(), scenario, sampler)
```

To run the simulation, we can use `simulator.nextStep()` to progress one step (one event) or `simulator.runFor(duration)` to run for a set duration.

```kotlin
// imports TKTK

sampler.beginBatch()
simulator.runFor(7.days)
sampler.endBatch()
```

Since we're not visualising the simulation as it runs, telling the sampler to run the simulation inside one big batch significantly improves performance, and is thus good practice.

## Visualising results

To visualise your port or completed simulation, SimPort provides three options, but for now let's stick with a static visualisation of the port once it has finished running. Running the visualiser is as simple as calling `StaticVisualisation(sampler)`

```kotlin
// imports TKTK

StaticVisualisation(sampler)
```

**And that's it!**

## Complete process

```kotlin
data object Truck

val Scenario = buildScenario {
    arrivals(
        "Truck arrivals",
        Generators.constant(Truck, Delays.exponential(50.0, DurationUnit.HOURS))
    )
        .thenQueue("Truck arrival queue")
        .thenSink("Truck departures")
}

val sampler = MetricsPanelState(scenario)
val simulator = Simulator(EventLog.noop(), scenario, sampler)
sampler.beginBatch()
simulator.runFor(7.days)
sampler.endBatch()

StaticVisualisation(sampler)
```