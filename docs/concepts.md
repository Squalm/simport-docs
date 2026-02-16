---
title: Documentation
layout: default
nav_order: 4
---
# Core Concepts

For this project, we designed a somewhat non-standard simulation layout. Our models differ slightly from a petri-net.

In our model, everything is a node. A road is a node that takes cars as inputs and outputs cars after a delay to
represent the travel time. A loading crane is a node that has a container input, an empty truck input, and an loaded
truck output.

A node's behaviour is customised similar to a state machine: it's a class with callbacks that get called when events
happen. The class's data corresponds to the state of a state machine and the behaviour of the callbacks define the
transitions of a state machine. These callbacks can schedule additional callbacks that will be called after a specified delay. 

## Our first port

To represent a road node, we create a node that schedules a send event on its output channel after a delay.

```kotlin 

```

We can simulate a port by connecting a source node, our delay node, and a sink node.

```kotlin 

```

Additionally, our library is built with a visualisation component that shows a port layout with a simulation.

```kotlin 

```

You should be able to see the following port. Click run to play the simulation. If the simulation is taking too long,
click the step button to jump the simulation forward. Stepping simulates the port as fast as possible without slowing
down to render events at the play speed.

## Push and pull channels

We have two types of channels: push channels allows the source node to push objects to the destination node, pull
channels allows the destination node to pull objects from the source node.

Remark: We initially only had push channels. However, we realised that it was not enough to represent match nodes, our
equivalent to a petri-net transitions. A petri-net transition waits for all its input channels to have an object before
simultaneously pulling an object from every input channel. Moreover, push forks are nicely symmetric to pull joins. A
push fork takes in one input and chooses one output to push into. A pull join takes in a number of pull inputs and one
output. It can decide from which channel to pull from.

```kotlin
// TODO put code for push channel interface
```

The source node of a push channel can ask if a push channel is open and push into an open channel. If the channel is not
open during a push, an exception is thrown (and the simulator crashes if the exception is not caught). Additionally, it
can register a callback for when the channel is opened or closed.

The destination node of a push channel can open or close the channel. It must register a callback to handle when the
source node sends an object through the channel.

```
// TODO put code for pull channel interface 
```

// TODO 

> **Caution:** Because the events are based on callbacks, it could be the case that there is a cycle in the graph resulting in the callback being called before it has returned. Re-entry is complicated because the object's internal state might be in a weird state so we strongly recommend that nodes do not send objects immediately upon being notified. When a node wants to send an object immediately after receiving a notification it should schedule a callback with 0 delay. That callback should be the one to call `channel.send`. 

## TODOs

// TODO error when they try to push an event when they get notified ???

