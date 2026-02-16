---
title: Node builders
layout: default
parent: Documentation
nav_order: 2
---
# Node builders

Node builders are sugar that help you build ports. They are designed to be used with our Domain Specific Language.

## DSL helpers

These helper functions have use type safety to ensure you create a valid port. If you understand the concept of a push and pull channel, the DSL should be a intuitive way to express port layouts (without any requirement of understanding all the type weirdness – it's just here to keep you safe).

### Then connect
```kotlin
fun <ItemT, ChannelT : ChannelType<ChannelT>> RegularNodeBuilder<*, out ItemT, ChannelT>
        .thenConnect(connection: Connection<in ItemT, ChannelT>)
```

Connect a node that produces an `ItemT` to a channel.

### Then output
```kotlin
fun <ItemT, ChannelT : ChannelType<ChannelT>> RegularNodeBuilder<*, ItemT, ChannelT>.thenOutput(
    outputRef: OutputRef<ItemT, ChannelT>
)
```

Connects a node output to an `OutputRef`. Useful in compound nodes, for example.


### Arrivals
```kotlin
fun <T> arrivals(
    label: String, 
    generator: Generator<T>
): RegularNodeBuilder<ArrivalNode<T>, T, ChannelType.Push>
```

Build an arrivals (source) node.

### Then delay
```kotlin
fun <T> NodeBuilder<T, *>.thenDelay(
    label: String,
    delayProvider: DelayProvider,
): RegularNodeBuilder<DelayNode<T>, T, ChannelType.Push>
```

Build a delay node with a delay provider.

### Then pump
```kotlin
fun <T> NodeBuilder<T, ChannelType.Pull>.thenPump(
    label: String = "Pump"
): RegularNodeBuilder<PumpNode<T>, T, ChannelType.Push>
```

Build a pump node.

### Then fork
```kotlin
// Push fork with lanes defined in a list
fun <ItemT, R> NodeBuilder<ItemT, ChannelType.Push>.thenFork(
    label: String,
    lanes: List<(RegularNodeBuilder<PushForkNode<ItemT>, ItemT, ChannelType.Push>) -> R>,
    policy: ForkPolicy<ItemT> = RandomPolicy<PushOutputChannel<ItemT>>().asForkPolicy(),
): List<R>
```
```kotlin
// Push fork with lanes defined in a lambda function
fun <ItemT, R> NodeBuilder<ItemT, ChannelType.Push>.thenFork(
    label: String,
    numLanes: Int,
    policy: ForkPolicy<ItemT> = RandomPolicy<PushOutputChannel<ItemT>>().asForkPolicy(),
    laneAction: (Int, RegularNodeBuilder<PushForkNode<ItemT>, ItemT, ChannelType.Push>) -> R,
): List<R>
```

```kotlin
// Pull fork with lanes defined in a list
fun <ItemT, R> NodeBuilder<ItemT, ChannelType.Pull>.thenFork(
    label: String,
    lanes: List<(RegularNodeBuilder<PullForkNode<ItemT>, ItemT, ChannelType.Pull>) -> R>,
): List<R>
```

```kotlin
// Pull fork with lanes defined in a lambda function
fun <ItemT, R> NodeBuilder<ItemT, ChannelType.Pull>.thenFork(
    label: String,
    numLanes: Int,
    laneAction: (Int, RegularNodeBuilder<PullForkNode<ItemT>, ItemT, ChannelType.Pull>) -> R,
): List<R>
```

```kotlin
// Syntactic sugar to convert from Pull-output source to Push-input destination lanes
fun <ItemT, R> NodeBuilder<ItemT, ChannelType.Pull>.thenPushFork(
    label: String,
    numLanes: Int,
    policy: ForkPolicy<ItemT> = RandomPolicy<PushOutputChannel<ItemT>>().asForkPolicy(),
    laneAction: (Int, RegularNodeBuilder<PushForkNode<ItemT>, ItemT, ChannelType.Push>) -> R,
): List<R>
```

Build a fork node, which may be a Push Fork or Pull Fork depending on the output type of the previous node. Returns a `List<R>` so you can iterate over the outputs of the fork with all the usual sugar.

### Then join
```kotlin
// Push join
fun <T> List<NodeBuilder<T, ChannelType.Push>>.thenJoin(
    label: String
): RegularNodeBuilder<PushJoinNode<T>, T, ChannelType.Push>
```

```kotlin
// Pull join
fun <T> List<NodeBuilder<T, ChannelType.Pull>>.thenJoin(
    label: String,
    policy: JoinPolicy<T> = RandomPolicy<PullInputChannel<T>>().asJoinPolicy(),
): RegularNodeBuilder<PullJoinNode<T>, T, ChannelType.Pull>
```

Build a join node of appropriate type, which merges a list of lanes in the DSL to a single pathway.

### Then match
```kotlin
fun <MainInputT, SideInputT, OutputT, ChannelT : ChannelType<ChannelT>> NodeBuilder<MainInputT, ChannelT>.thenMatch(
    label: String,
    side: NodeBuilder<SideInputT, ChannelType.Pull>,
    combiner: (MainInputT, SideInputT) -> OutputT,
): RegularNodeBuilder<MatchNode<MainInputT, SideInputT, OutputT, ChannelT>, OutputT, ChannelT>
```

Build a match node. 

### Then queue
```kotlin
fun <T> NodeBuilder<T, *>.thenQueue(
    label: String,
    policy: QueuePolicy<T> = FIFOQueuePolicy(),
): RegularNodeBuilder<QueueNode<T>, T, ChannelType.Pull>
```

Build an unbounded queue node.

### Then service
```kotlin
fun <T> NodeBuilder<T, *>.thenService(
    label: String,
    delayProvider: DelayProvider,
): RegularNodeBuilder<ServiceNode<T>, T, ChannelType.Push>
```

Build a service node.

### Then split
```kotlin
fun <
    InputT, 
    MainOutputT, 
    SideOutputT, 
    ChannelT : ChannelType<ChannelT>
> NodeBuilder<InputT, ChannelT>.thenSplit(
    label: String,
    splitter: (InputT) -> Pair<MainOutputT, SideOutputT>,
): Pair<
    RegularNodeBuilder<SplitNode<InputT, MainOutputT, SideOutputT, ChannelT>, MainOutputT, ChannelT>,
    RegularNodeBuilder<SplitNode<InputT, MainOutputT, SideOutputT, ChannelT>, SideOutputT, ChannelType.Push>,
>
```

Build a split node. Returns a pair of node, one or the main output and one for the side output.

### Then sink
```kotlin
fun <T> NodeBuilder<T, *>.thenSink(label: String): SinkNode<T>
```

Build a sink node.

### Then dead end
```kotlin
fun <T> NodeBuilder<T, *>.thenDeadEnd(label: String): DeadEndNode<T>
```

Build a dead end node.

### Build scenario
```kotlin
fun buildScenario(
    builder:
        context(ScenarioBuilderScope, GroupScope)
        () -> Unit
): Scenario
```

Build a scenario from a builder.

### With metrics
```kotlin
fun Scenario.withMetrics(builder: MetricsBuilderScope.() -> Unit): Scenario
```

Adds the metrics to track to a scenario.
