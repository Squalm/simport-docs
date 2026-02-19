---
title: Defining node policy
layout: default
parent: Tutorials
nav_order: 5
---
# Policies, Tags, and Containers

## Queue Policy
Let's walk through creating a custom queue policy, _PreferCompany_, where the queue would preferentially emit the selected company's vehicles with a higher probability than other companies' vehicles.
This can be done through implementing the `QueuePolicy`
 interface described in the API guide. 
```kotlin
data class CompanyVehicle(val company: Company)

class PreferCompany(
    private val preferredCompany : Company,
    private val bias : float = 0.75, // Probability of selecting the preferred company's vehicles
) : QueuePolicy<CompanyVehicles>
```

Implementing the interface means the basic functions need to be defined for this new policy:

```kotlin
class PreferCompany(
    private val preferredCompany: Company,
    private val bias: float = 0.75, // Probability of selecting the preferred company's vehicles
) : QueuePolicy<CompanyVehicles> {
    /* Privately, the contents of our queue can be stored in as 
       complex of a data structure as needed */
    private val biasedCompanyBuffer = ArrayDeque<CompanyVehicle>()
    private val otherCompaniesBuffer = ArrayDeque<CompanyVehicle>()

    /* Contents should return all things held in the queue,
    *  so make sure to combine your data structures into a single sequence */
    override val contents: Sequence<CompanyVehicles>
        get() =
            biasedCompanyBuffer.asSequence() + otherCompaniesBuffer.asSequence()

    /* Given the object, describe the behaviour of the queue policy when an
    *  enqueue event takes place */
    override fun enqueue(obj: CompanyVehicles) {
        // If the object is preferred by the policy, then it is saved in the special buffer
        if (obj.company == preferredCompany) {
            biasedCompanyBuffer.addFirst(obj)
        } else {
            otherCompaniesBuffer.addFirst(obj)
        }
    }

    /* The behaviour of the queue when asked to produce a vehicle. In this case,
    *  a random float is selected and the company's vehicle is emitted if selected */
    override fun dequeue(): CompanyVehicles {
        return if (Random.nextFloat() <= bias && biasedCompanyBuffer.size > 0) {
            biasedCompanyBuffer.removeLast()
        } else {
            otherCompaniesBuffer.removeLast()
        }
    }
    
    override fun reportOccupancy(): Int {
        return biasedCompanyBuffer.size + otherCompaniesBuffer.size
    }
}
```

## Push Fork / Pull Join Generic Policy

Certain policies can be generic over both push forks and pull joins due to their shared nature of selecting from a set of channels, and performing the action onto it.

Generic policies can be defined by implementing the `GenericPolicy` interface described in the API guide.

Let's walk through creating the included generic policy `RandomPolicy`, which dispatches to and takes from a random channel controlled by the node.

```kotlin
/* GenericPolicy takes a generic ChannelT, which describes the type of channel is being handled
* (PushOutputChannel for Fork use, and PullInputChannel for Join use)
* Please cascade this Generic type into GenericPolicy */
class RandomPolicy<ChannelT> : GenericPolicy<ChannelT>() {
    /* Most policies keep a 'openChannels' data structure representing the channels that 
    can be selected by the policy for the node to use */
    private val openChannels = mutableSetOf<ChannelT>()
 
    /* Selects a channel for the node to use. 
    In this case, we select a random open channel from our internal knowledge */ 
    override fun selectChannel(): ChannelT {
     return openChannels.random()
    }
 
    /* When a channel that we are concerned with becomes opened, or becomes ready to be pulled from, this function is called.
    * The conversion to a ForkPolicy or JoinPolicy using conversion systems will automatically read the correct channels, 
    * and link their `onReady` and `whenOpened` to this.
    * 
    * In this case, we will just add it into our set of opened channels that can be selected from.
    *  */
    override fun onChannelAvailable(channel: ChannelT) {
     openChannels.add(channel)
    }
    
    /* When a channel that we are concerned with becomes closed, or is no longer ready to be pulled from. 
    * Similarly to onChannelAvailable, the conversion systems will link the appropriate channels, so simply defined
    * any internal behaviour to the policy here. */
    override fun onChannelUnavailable(channel: ChannelT) {
     openChannels.remove(channel)
    }
    
    /* Returns whether all channels are unavailable, in which case the node will close */
    override fun allUnavailable(): Boolean {
     return openChannels.isEmpty()
    }
    
    /* For more complex policies, we may want to initialise internal attributes */
    context(_: Simulator)
    override fun initialize(channels: List<ChannelT>) {}
}
```

Generic policies are then converted into `JoinPolicy` and `ForkPolicy` instances by passing the generic policy into `GenericJoinPolicy` and `GenericForkPolicy` classes respectively:

```kotlin
fun <T> joinPolicy(policy: GenericPolicy<PullInputChannel<T>>): JoinPolicy<T> 
    = GenericJoinPolicy(policy)

class GenericJoinPolicy<T>(
    private val policy: GenericPolicy<PullInputChannel<T>>
) : JoinPolicy<T> {
    override fun selectChannel() = policy.selectChannel()

    override fun onChannelReady(channel: PullInputChannel<T>) = policy.onChannelAvailable(channel)

    override fun onChannelNotReady(channel: PullInputChannel<T>) = policy.onChannelUnavailable(channel)

    override fun noneReady() = policy.allUnavailable()

    context(_: Simulator)
    override fun initialize(sources: List<PullInputChannel<T>>, destination: PullOutputChannel<T>) {
        policy.initialize(sources)
        super.initialize(sources, destination)
    }
}
```

The type system can automatically infer the `ChannelT` used by the generic policy, so these policies in practise can be converted like such:

```kotlin
val randomJoinPolicy: JoinPolicy<Int> = joinPolicy(RandomPolicy())
```

## Complex Fork / Join Policies, Tags, Containers

Fork and join policies can make routing decisions based on downstream or upstream occupancy information, searched for and kept using the `Tag` and `Container` system. Let's take a look at `LeastFullForkPolicy`:

```kotlin
class LeastFullForkPolicy<T>(
    private val tags: Iterable<OutputTag<Container<*>>> =
        generateSequence { newDynamicOutputTag { it.walkDownstream().filterIsInstance<Container<*>>().first() } }
            .asIterable()
) : ForkPolicy<T> {
    private val containers = IdentityHashMap<PushOutputChannel<T>, Container<*>>()
    private val openDestinations = Collections.newSetFromMap<PushOutputChannel<T>>(IdentityHashMap())
    ...

    override fun selectChannel(obj: T): PushOutputChannel<T> {
        return openDestinations.minBy { containers.getValue(it).occupants }
    }
 
    ...
 
    context(_: Simulator)
    override fun initialize(source: PushInputChannel<T>, destinations: List<PushOutputChannel<T>>) {
        destinations
            .asSequence()
            .zipCompletely(tags.asSequence())
            .associateTo(containers) { (destination, tag) ->
                destination to tag.find(destination) 
            }
     
        super.initialize(source, destinations)
    }
}
```

**Tags** are identifiers that are bound to some object that can be identified elsewhere in the graph. _DynamicTags_ are tags that move depending on the starting location. In the above default parameter, a sequence of `DynamicOutputTag` are generated such that when associated with a `PushOutputChannel`, performs a downstream walk from the channel and 'binds' itself to the first container found. 

`LeastFullForkPolicy` uses _dynamic tags_ which are assigned their 'starting channel' at policy initialisation.

```kotlin
sealed interface OutputTag<out T> {
    fun find(start: PushOutputChannel<*>): T
}

sealed interface InputTag<out T> {
    fun find(start: PullInputChannel<*>): T
}
```

**Containers** are metric-tracking objects, implemented by any node that has can contain item(s):
```kotlin
interface Container<out T> {
    val occupants: Int

    fun onEnter(
        callback:
            context(Simulator)
            (T) -> Unit
    )

    fun onLeave(
        callback:
            context(Simulator)
            (T) -> Unit
    )

    fun supportsResidenceTime(): Boolean = true
}

interface BoundedContainer<out T> : Container<T> {
    val capacity: Int

    val isFull
        get() = occupants >= capacity
}
```

For example, a subnetwork's occupancy is determined by maximum capacity subtracted by the number of free tokens, and a queue's occupancy is determined by the number of items inside presently. 

By using both the tags and containers systems, policies can consider downstream occupancy information. By default, `LeastFullForkPolicy` and `MostFullJoinPolicy` are implemented, which pushes to the least full fork branch, and pulls from the most full join input branch, respectively, for fork and join nodes. 