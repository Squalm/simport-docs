---
title: Policy
layout: default
parent: Internals
nav_order: 70
---

# Policy

Policies dictate how Fork, Join, and Queue nodes behave, from which complex scenarios can be built from. For example, a custom `Queue` node policy to preferentially send out Company A's vehicles creates the basis to observe the effects on port throughput by prioritising a specific company.

## Fork and Join Policies

### Generic Policies

Due to the symmetrical nature of push forks and pull joins, some policies are shared between the two nodes. For example, `RandomPolicy` is shared, and can be converted into a ForkPolicy or JoinPolicy where needed.

```kotlin
open class RandomPolicy<ChannelT> : GenericPolicy<ChannelT>()

...
    .thenQueue("Queue")
    .thenPump()
    .thenFork("Example Push Fork", numLanes, RandomPolicy<PushOutputChannel<T>>().asForkPolicy()) {
        i, lane ->
            lane
                .thenQueue("Inner queue")
                ...
    }
    .thenJoin("Example Pull Join", RandomPolicy<PullInputChannel<T>>().asJoinPolicy())
    ...
```

Other policies that are shared include:

- `RoundRobinPolicy`
- `PriorityPolicy` where some comparator is needed to compare channels
- `FirstAvailablePolicy` which picks the first channel whenever available.

To create custom generic policies, simply inherit the `GenericPolicy` abstract class:
```kotlin
abstract class GenericPolicy<ChannelT> {
    abstract fun selectChannel(): ChannelT

    abstract fun onChannelAvailable(channel: ChannelT)

    abstract fun onChannelUnavailable(channel: ChannelT)

    abstract fun allClosed(): Boolean

    context(_: Simulator)
    abstract fun initialize(channels: List<ChannelT>)
}
```

### Push Fork Policies

Some policies may only be useful for push forks, such as `LeastFullForkPolicy`:
```kotlin
class LeastFullForkPolicy<T>(
    private val tags: Iterable<ForwardWalkTag<Container<*>>>,
) : ForkPolicy<T>
```

Which performs a traversal through its downstream lanes to identify the first **container** in each lane, and forwards items to the lane with the lowest occupancy.

To create custom push-fork exclusive policies, simply inherit the `ForkPolicy` interface:
```kotlin
interface ForkPolicy<T> {
    fun selectChannel(obj: T): PushOutputChannel<T>

    fun onChannelOpen(channel: PushOutputChannel<T>)

    fun onChannelClose(channel: PushOutputChannel<T>)

    fun allClosed(): Boolean

    // default implementation provided to join the channels to the policy class
    context(_: Simulator)
    fun initialize(source: PushInputChannel<T>, destinations: List<PushOutputChannel<T>>)
}
```

### Pull Join Policies

Some policies may only be useful for pull joins, such as `MostFullJoinPolicy`:
```kotlin
class MostFullJoinPolicy<T>(
    private val tags: Iterable<BackwardWalkTag<Container<*>>>
) : JoinPolicy<T>
```

Which performs a reverse traversal through its upstream lanes being merged together, to find the lane with the highest occupancy, and draws from there first.

To create custom pull-join exclusive policies, simply inherit the `JoinPolicy` interface:
```kotlin
interface JoinPolicy<T> {
    fun selectChannel(): PullInputChannel<T>

    fun onChannelReady(channel: PullInputChannel<T>)

    fun onChannelNotReady(channel: PullInputChannel<T>)

    fun allClosed(): Boolean

    // default implementation provided
    context(_: Simulator)
    fun initialize(sources: List<PullInputChannel<T>>, destination: PullOutputChannel<T>)
}
```

## Queue Policies

Queue policies control the behaviour of queues, which by default are **FIFO**. Custom queue policies can be created through implementing the `QueuePolicy` interface:

```kotlin
interface QueuePolicy<T> {
    val contents: Sequence<T>

    fun enqueue(obj: T)

    fun dequeue(): T

    fun reportOccupancy(): Int
}
```

Provided queue policies include:
- `FIFOQueuePolicy` which simulates a normal FIFO queue, and is the default parameter for a queue defined in DSL without passing an explicit policy.
- `RandomQueuePolicy`
- `PriorityQueuePolicy`