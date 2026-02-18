---
title: Visualisation
layout: default
parent: Documentation
nav_order: 8
---

# Visualisations
There are 3 types of visualisations available to use with SimPort: StaticVisualisation, MultiVisualisation and LiveVisualisation. Run any of these visualisations using the `runVisualisation` function.
```kotlin
fun runVisualisation(body: @Composable () -> Unit) 
```

## StaticVisualisation
A StaticVisualisation renders a single run of a port layout, and can be created using the `runSimulation` function:

```kotlin
fun runSimulation(scenario: Scenario, duration: Duration, logger: EventLog = EventLog.noop())
```
`runSimulation` requires a `Scenario`, a `Duration` to simulate these scenarios for, and optionally an `EventLog` to print output. Internally, it creates a `MetricsPanelState` with the `Scenario`, a `Simulator`, and then runs the simulation for the `Duration` provided. Then it creates a `StaticVisualisation` using the result.

## MultiVisualisation
A MultiVisualisation renders multiple runs of different scenarios, and can be created using the `runSimulations` function:

```kotlin
fun runSimulations(
    scenarios: Map<String, Scenario>,
    duration: Duration,
    logger: (String) -> EventLog = { EventLog.noop() },
)
```

`runSimulations` requires a map of scenario names (`String`s) to `Scenario`s, a `Duration` to simulate these scenarios for, and optionally a `(String) -> EventLog` function to create loggers by scenario name. Internally, it creates a `MetricsPanelState`, `Simulator` and `EventLog` for each `Scenario` provided, and runs each of those simulations. Then it creates a `MultiVisualisation` using the results.

## LiveVisualisation
A LiveVisualisation allows the user to run a simulation live for a specified port configuration using a GUI, and can be created using the `runLiveSimulation` function:

```kotlin
fun runLiveSimulation(scenario: Scenario, logger: EventLog = EventLog.noop())
```

`runLiveSimulation` requires a `Scenario` and an `EventLog`. Note that it does not require a `Duration`, as it can be run for arbitrarily long via the GUI. Internally, it simply creates a `LiveVisualisation` using the provided `Scenario` and `EventLog`.



