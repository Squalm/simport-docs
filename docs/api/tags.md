<!-- ---
title: Tags
layout: default
parent: Internals
nav_order: 9
nav_exclude: true # hide while incomplete
--- -->
### Tagged
```kotlin
fun <BuilderT : RegularNodeBuilder<NodeT, *, *>, NodeT : NodeGroup> BuilderT.tagged(
    tag: MutableBasicTag<in NodeT>
): BuilderT
```

```kotlin
fun <BuilderT : RegularNodeBuilder<NodeT, *, *>, NodeT : NodeGroup> BuilderT.tagged(
    tags: MutableList<in BasicTag<NodeT>>
): BuilderT
```

Gives a node a tag.
