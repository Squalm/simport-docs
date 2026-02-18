---
title: Creating subnetworks
layout: default
parent: Tutorials
nav_order: 2
---

# Subnetworks

```kotlin
class BoundedSubnetwork<
    ItemT,
    InputChannelT : ChannelType<InputChannelT>,
    OutputChannelT : ChannelType<OutputChannelT>,
>(...) : CompoundNode, BoundedContainer<ItemT>
```

Subnetworks represents a region of the port where there is limited capacity. 

When a subnetwork has reached maximum capacity, the subnetwork's input channel will be closed or will be marked not ready, until at least one item has left the subnetwork.


### Construction
A bounded subnetwork can be created using the DSL semantics as shown below, creating a subnetwork inline with the rest of the queue network, with a capacity of 100 vehicles. 
```kotlin
buildScenario {
    arrivals(
        "Arrival",
        generator = Generators.constant(Vehicle, Delays.fixed(5.seconds)).take(numVehicles),
    )
        .thenQueue("Buffer into subnetwork")
        .thenSubnetwork("Bounded subnetwork 1", capacity = 100) {
            it.thenQueue("First queue in subnetwork").thenService
        }
}
```