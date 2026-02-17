---
title: Defining node policy
layout: default
parent: Tutorials
nav_order: 4
---
# Policy Tutorial

## Queue Policy
Let's walk through creating a custom queue policy, _PreferCompanyA_, where the queue would preferentially emit company A's vehicles with a higher probability than other companies' vehicles.

```kotlin
class PreferCompany
```
