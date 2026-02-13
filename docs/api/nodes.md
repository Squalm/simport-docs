---
title: Nodes
layout: default
parent: Documentation
nav_order: 1
---
# Nodes

SimPort aims to provide a simple set of basic nodes that can be easily combined to produce arbitrary behaviour. Nodes are not 'a road' or 'a crane' but instead, a delay or a fork, which should be familiar to queue theorists.

This is useful because there can be multiple kinds of objects flowing through ports, and it's simple to give your ports and objects names and make the intended real-world-relation clear.

## Channels

Nodes in SimPort are connected by Channels, which represent graph edges. Channels can be **push** or **pull** channels.

For your intuition, in a **push** channel, the incoming node pushes an object into the channel and the outgoing node must receive it immediately or crash. In a **pull** channel, the incoming node registers an object as ready to be emitted, and the outgoing node pulls it in whenever they want.

some nodes need certain connections to be specifically a push channel or a pull channel. This is enforced through type safety and usually shouldn't be a problem. However, you may sometimes need to insert a pump node inbetween two nodes to connect them.

## Node types

A node is:

```kotlin
abstract class Node(
    label: String,
    final override val incoming: List<InputChannel<*, *>>,
    final override val outgoing: List<OutputChannel<*, *>>,
) : NodeGroup(label)
```

Source nodes are a special kind of node with no incoming channels:

```kotlin
abstract class SourceNode(
    label: String, 
    outgoing: List<PushOutputChannel<*>>
) : Node(label, emptyList(), outgoing)
```

### Arrival node
```kotlin
class ArrivalNode<OutputT>(
    label: String,
    destination: PushOutputChannel<OutputT>,
    generator: Generator<OutputT>,
) : SourceNode(label, listOf(destination)), Source<OutputT>
```

A source node where objects arrive according to a generator.

### Container node
```kotlin
abstract class ContainerNode<T>(
    label: String,
    incoming: List<InputChannel<*, *>>,
    outgoing: List<OutputChannel<*, *>>,
) : Node(label, incoming, outgoing), Container<T>
```

An abstraction for any node that can hold things.

### Dead end node
```kotlin
class DeadEndNode<InputT>(
    label: String, 
    private val inputChannel: InputChannel<InputT, *>
) : Node(label, listOf(inputChannel), emptyList())
```

Dead end node closes its input channel immediately, representing a queue node that is 'closed for maintenance' or equivalent, and will crash the simulator if a vehicle is dispatched to this node.

### Delay node
```kotlin
class DelayNode<T>(
    label: String,
    source: PushInputChannel<T>,
    destination: PushOutputChannel<T>,
    delayProvider: DelayProvider,
) : ContainerNode<T>(label, listOf(source), listOf(destination)), Delay<T>
```

Takes in a object, and sends it out through the designated destination output channel after some delay specified by the `delayProvider`. Can hold any number of objects and delays happen simultaneously.

### Fork node
```kotlin
class ForkNode<T>(
    label: String,
    source: PushInputChannel<T>,
    destinations: List<PushOutputChannel<T>>,
    policy: ForkPolicy<T> = RandomForkPolicy(),
) : Node(label, listOf(source), destinations)
```

Takes in a object, and emits it to any one of its destinations, as long as the output channel is open. Use `policy` to define policy for where the node should emit.

Equivalent of OrForks in queue theory.

### Join node
```kotlin
class JoinNode<T>(
    label: String, 
    sources: List<PushInputChannel<T>>, 
    destination: PushOutputChannel<T>
) : Node(label, sources, listOf(destination))
```

Push joins join multiple streams together.

### Match node
```kotlin
class MatchNode<MainInputT, SideInputT, OutputT, ChannelT : ChannelType<ChannelT>>(
    label: String,
    mainSource: InputChannel<MainInputT, ChannelT>,
    sideSource: PullInputChannel<SideInputT>,
    destination: OutputChannel<OutputT, ChannelT>,
    combiner: (MainInputT, SideInputT) -> OutputT,
) :
    PassthroughNode<MainInputT, OutputT, ChannelT>(
        label,
        mainSource,
        destination,
        listOf(mainSource, sideSource),
        listOf(destination),
    ),
    Match<MainInputT, SideInputT, OutputT>
```

Match nodes wait till the combine an object from the main and side input to produce one object using a `combiner`. Crucially used with a split node in bounded subnetworks.

### Pass-through node
```kotlin
abstract class PassthroughNode<InputT, OutputT, ChannelT : ChannelType<ChannelT>>(
    label: String,
    source: InputChannel<InputT, ChannelT>,
    destination: OutputChannel<OutputT, ChannelT>,
    sources: List<InputChannel<*, *>>,
    destinations: List<OutputChannel<*, *>>,
) : Node(label, sources, destinations)
```

An abstraction for nodes that have a main and side channel like match and split nodes.

### Pump node
```kotlin
class PumpNode<T>(
    label: String,
    source: PullInputChannel<T>,
    destination: PushOutputChannel<T>,
) : Node(label, listOf(source), listOf(destination))
```

Pumps convert a pull channel into a push channel by pulling when available and pushing immediately.

### Queue node
```kotlin
class QueueNode<T>(
    label: String,
    source: PushInputChannel<T>,
    destination: PullOutputChannel<T>,
    policy: QueuePolicy<T> = FIFOQueuePolicy(),
) : ContainerNode<T>(label, listOf(source), listOf(destination)), Queue<T>
```

Keeps a (by default, First-in first-out) queue of objects. Queue nodes are always unbounded.

### Service node
```kotlin
class ServiceNode<T>(
    label: String,
    source: PushInputChannel<T>,
    destination: PushOutputChannel<T>,
    delayProvider: DelayProvider,
) : ContainerNode<T>(label, listOf(source), listOf(destination)), Service<T>
```

Similar to a delay node, but can only serve one object at a time.

### Sink node
```kotlin
class SinkNode<InputT>(
    label: String, 
    source: PushInputChannel<InputT>
) : ContainerNode<InputT>(label, listOf(source), emptyList()), Sink<InputT>
```

A sink or destination for objects. A node with no outputs. 

### Split node
```kotlin
class SplitNode<InputT, MainOutputT, SideOutputT, ChannelT : ChannelType<ChannelT>>(
    label: String,
    source: InputChannel<InputT, ChannelT>,
    mainDestination: OutputChannel<MainOutputT, ChannelT>,
    sideDestination: PushOutputChannel<SideOutputT>,
    splitter: (InputT) -> Pair<MainOutputT, SideOutputT>,
) :
    PassthroughNode<InputT, MainOutputT, ChannelT>(
        label,
        source,
        mainDestination,
        listOf(source),
        listOf(mainDestination, sideDestination),
    ),
    Split<InputT, MainOutputT, SideOutputT>
```

Splits an object into two using a `splitter` and sends them down the main and side channel. Crucially used with a match node in bounded subnetworks.
