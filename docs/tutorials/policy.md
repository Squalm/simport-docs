---
title: Defining node policy
layout: default
parent: Tutorials
nav_order: 4
---
# Policy Tutorial

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