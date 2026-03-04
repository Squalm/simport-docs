---
title: Metrics
layout: default
parent: Tutorials
nav_order: 30
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

There are 3 types of metrics: Continuous (measured over time), Instantaneous (triggered by specific events), and Rates (events averaged over time).

There are several built-in metrics:

### Arrival rate `ArrivalRate`
Rate metric: average rate of arrivals.

### Inter-arrival time `InterArrivalTime`
Instantaneous: time between arrivals.

### Inter-departure time `InterDepartureTime`
Instantaneous: time between departures.

### Occupancy `Occupancy`
Continuous: objects in a node.

### Residence time `ResidenceTime`
Instantaneous: total time any object spends in a node before going to a sink

{: .important }
Residence time counts objects that *never* enter a node (as `0`s), whereas response time will not. 

### Response time `ResponseTime` 
Instantaneous: time an object spends in a node before leaving the node

### Throughput `Throughput`
Rate: average rate of departures.

### Utilisation `Utilisation`
Continuous: objects inside doing work as a fraction of capacity.

Utilisation is a Local metric only.

## Custom Metrics

Custom metrics can be implemented by extending `ContinuousMetric`, `InstantaneousMetric`, or `RateMetric`.

### Continuous Metrics

```kotlin
sealed class Occupancy : ContinuousMetric() {

    protected abstract val current: Int

    override fun reportImpl(previousTime: Instant, currentTime: Instant) = current.toDouble()

    class Local(private val container: Container<*>) : Occupancy() {
        override val current
            get() = container.occupants
    }
}
```

### Instantaneous Metrics

```kotlin
sealed class InterArrivalTime(private val unit: DurationUnit) : InstantaneousMetric() {
    private var lastSeen: Instant? = null

    context(sim: Simulator)
    protected fun notifySeen() {
        val currentTime = sim.currentTime
        val lastSeen = lastSeen
        if (lastSeen != null) {
            notify(currentTime, (currentTime - lastSeen).toDouble(unit))
        }
        this.lastSeen = currentTime
    }

    class Local(container: Container<*>, unit: DurationUnit = DurationUnit.SECONDS) : InterArrivalTime(unit) {
        init {
            container.onEnter { notifySeen() }
        }
    }
}
```

### Rate Metrics
Rate metrics handle calculating the rates entirely in their backend. All the metric implementation needs to do is forward the correct events.

```kotlin
sealed class Throughput(unit: DurationUnit) : RateMetric(unit) {

    context(sim: Simulator)
    protected fun notify() {
        notify(sim.currentTime)
    }

    class Local(container: Container<*>, unit: DurationUnit) : Throughput(unit) {
        init {
            container.onLeave { notify() }
        }
    }
}
```

### Metric Factories

Node-specific Metrics should have a companion object extending `MetricFactory`:

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