---
title: Tags
layout: default
parent: Documentation
nav_order: 9
---
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
