---
title: Metrics
layout: default
parent: Tutorials
nav_order: 2
---

# Metrics

## Enabling Metrics

Metrics can be enabled in 3 ways:

```kotlin
// 1. Track a metric on a specific node:
input
    .thenQueue("Some Queue") // for example
    .track(Occupancy) // for example

// 2. Track metrics for every applicable node:
buildScenario { ... }
    .withMetrics {
        trackAll(Occupancy) // for example
    }

// 3. Track global metrics:
buildScenario { ... }
    .withMetrics {
        trackGlobal(Occupancy) // for example
    }
```

Some metrics provide extra configuration options, such as a duration unit for time-based metrics. These can be provided like so:

```kotlin
input
    .thenQueue("Some Queue") // for example
    .track { ResidenceTime.create(it, DurationUnit.HOURS) } // for example

buildScenario { ... }
    .withMetrics {
        trackAll { ResidenceTime.create(it, DurationUnit.HOURS) } // for example
    }

buildScenario { ... }
    .withMetrics {
        trackGlobal { ResidenceTime.create(it, DurationUnit.HOURS) } // for example
    }
```

Most provided metrics can be used both globally and per node, though for custom metrics you need not implement both
varieties.

## Metric Types

There are 2 types of metrics: Continuous (measured over time) and Instantaneous (triggered by specific events).

There are several built-in metrics:

### `Occupancy`

Continuous – tracks how many occupants the node or simulation has.

### `ResidenceTime`

Instantaneous – tracks how long items spend in each node, or in the whole simulation.

### `Latency`

Instantaneous – tracks the time between items leaving a node or the whole simulation.

## Custom Metrics

Custom metrics can be implemented by extending either `ContinuousMetric` or `InstantaneousMetric`.

### Continuous Metrics

```kotlin
class Occupancy(private val container: Container<*>) : ContinuousMetric() {
    // Report the current value of the metric, given the time interval
    override fun reportImpl(previousTime: Instant, currentTime: Instant) = container.occupants.toDouble()
}
```

### Instantaneous Metrics

```kotlin
class ServiceTime(service: Service<*>) : InstantaneousMetric() {
    private lateinit var startTime: Instant

    init {
        service.onEnter {
            // Record the start time when the service is entered
            startTime = contextOf<Simulator>().currentTime
        }

        service.onLeave {
            val currentTime = contextOf<Simulator>().currentTime
            // Report the time difference as a sample of this metric
            notify(currentTime, (currentTime - startTime).toDouble(DurationUnit.SECONDS))
        }
    }
}
```

### Metric Factories

Node-specific metrics should have a companion object extending `MetricFactory`:

```kotlin
class Occupancy(container: Container<*>) : ContinuousMetric() {
    ...
    
    // This metric can be attached to only Container nodes
    companion object : MetricFactory<Container<*>> {
        override fun create(node: Container<*>): MetricGroup {
            // Create the metric itself
            val raw = Occupancy(node)
            // Helpers are provided to compute moments and confidence intervals for both continuous and instantaneous metrics
            val cis = ContinuousConfidenceIntervals(raw)
            return MetricGroup(
                name = "Occupancy", 
                associatedNode = node as NodeGroup, 
                raw = raw, 
                moments = cis.moments(),
            )
        }
    }
}
```

Global metrics should have a companion object extending `GlobalMetricFactory`:

```kotlin
class GlobalOccupancy(scenario: Scenario) : ContinuousMetric() {
    private var currentOccupants = 0
    
    override fun reportImpl(previousTime: Instant, currentTime: Instant) = currentOccupants.toDouble()

    init {
        for (source in scenario.allNodes.asSequence().filterIsInstance<Source<*>>()) {
            source.onEmit { 
                // A new object has entered the simulation
                currentOccupants++
            }
        }

        for (sink in scenario.allNodes.asSequence().filterIsInstance<Sink<*>>()) {
            sink.onEnter { 
                // An object has left the simulation
                currentOccupants--
            }
        }
    }

    companion object : GlobalMetricFactory {
        override fun create(scenario: Scenario): MetricGroup {
            val raw = GlobalOccupancy(scenario)
            val cis = ContinuousConfidenceIntervals(raw)
            return MetricGroup(
                name = "Occupancy", 
                associatedNode = null, 
                raw = raw, 
                moments = cis.moments(),
            )
        }
    }
}
```