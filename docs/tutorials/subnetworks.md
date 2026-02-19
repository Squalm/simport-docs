---
title: Creating subnetworks
layout: default
parent: Tutorials
nav_order: 4
---

# Subnetworks and Compound Node

```kotlin
class BoundedSubnetwork<
    ItemT,
    InputChannelT : ChannelType<InputChannelT>,
    OutputChannelT : ChannelType<OutputChannelT>,
>(...) : CompoundNode(...), BoundedContainer<ItemT>
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
            it.thenQueue("First queue in subnetwork").thenService("Service")
        }.thenSink("Sink")
}
```

### Implementation and CompoundNodes
Like a lot of things in life, subnetworks are abstracted versions of several nodes together, defined as:

```kotlin
/* input: Connection<out ItemT, InputChannelT> 
   inner : (NodeBuilder<ItemT, InputChannelT>) -> NodeBuilder<ItemT, OutputChannelT>
*/

val tokenBackEdge = newConnection<Token, _>(ChannelType.Push)
val tokenQueue : RegularNodeBuilder<QueueNode<Token>, Token, ChannelType.Pull> = 
    tokenBackEdge
        .thenQueue("Token Queue", TokenQueuePolicy(capacity))
        .saveNode { tokens = it }

input
    .thenMatch("Token Match", tokenQueue) { input, _ -> input }
    .saveNode { tokenMatch = it }
    .let { inner(it) }
    .thenSplit("Token Split") { output -> output to Token }
    .let { (outputs, tokens) ->
        outputs.saveNode { tokenSplit = it }
    
        tokens.thenConnect(tokenBackEdge)
        outputs.thenOutput(output)
    }
```

Subnetworks are implemented using match nodes and split nodes, and a token buffer. When the token buffer is drained entirely, as all tokens are paired with some vehicle inside the bounded subnetwork, then the match node will refuse any additional vehicles to pass through, until the split node processes (representing a vehicle leaving the subnetwork) a token-paired vehicle. 

The occupancy of the subnetwork is known.