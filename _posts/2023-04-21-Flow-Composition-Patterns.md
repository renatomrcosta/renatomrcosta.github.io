---
layout: post
title: "Flow Composition Patterns?"
description: "Extending and operating on top of existing flows has never been easier"
date: 2021-10-24
tags: [coroutines, flow, kotlin]
---
[Kotlin Flows](https://kotlinlang.org/docs/flow.html) are powerful tools to handle streams of data in a suspending manner. And, while most of the very useful operators are widely documented (such as the game-changing [transform](https://kotlinlang.org/docs/flow.html#transform-operator) operator), the composition and extension pattern for Flows is something that I don't think I see mentioned often

In this article, We shall explore this pattern, and hopefully add another tool to our toolbox when composing flow code.

<!--more-->

## The Problem

Consider the following: You have a function that produces a Flow of your favorite DTO, such as a JPA Repository like the following:

```kotlin
interface CustomerRepository : CoroutineCrudRepository<CustomerEntity, String> {}
```

Later, you can query the data from this repository, and extract a `Flow<customerEntity>` from it, and want to do some operations on it:

```kotlin
suspend fun greetAllCustomersMatchingSomeCriteria() {
    val flow: Flow<CustomerEntity> = customerRepository.findAll()
    flow.filter {
        someRemoteService.isCustomerEligible(it)
    }.map {
        createMessageForElegibleCustomer(it)
    }.collect {
        sendGreetingsToCustomer(it)
    }
}
```

the flow composition works as a sequence: for every `CustomerEntity` streamed in, it will call an external service to verify its eligibility to the greeting function, then generate a message, and then collect this calling another service to actually send that greeting. And while we are working in a suspenseful fashion, could we potentially make these calls happen in parallel?

The initial instinct to add `async {}` or `launch {}` blocks are immediately met with a problem: the flow composition functions do not have a `coroutineScope`! THis is intentional: Flows are meant to work sequentially. For async operations, we have `channelFlows` that allow us to do that. However, we hit another snag:

{% include image_caption.html imageurl="/images/posts/2023-04-20/1.png" title="Internal API" %}

The ChannelFlow type is meant to be an internal type. How could we compose our flow in such a way to leverage the power of channel flow, but having to declare our types using the generic `FLow<T>` type?

## The Solution

We can leverage the power of cold flows, and compose them by **collecting a flow in the definition of another flow**. In our case, to leverage a `ChannelFlow` in our function above, we could write code as such:

```kotlin
val flow: Flow<CustomerEntity> = customerRepository.findAll()

channelFlow {
    flow.collect {  value ->
        launch {
            if(someRemoteService.isCustomerEligible(it)) {
                send(it)
            }
        }
    }
}.map  {
    createMessageForElegibleCustomer(it)
}.collect {
    launch {
        sendGreetingsToCustomer(it)
    }
}
```

If the composed flow is not collected, the original flow won't be either: the cold, inert nature of flows ensures that collection happens only when requested!

In other words, the pattern for composing flows, in more generic terms, is:

```kotlin
fun <I, O> compose(flow: Flow<I>) = flowBuilder<O> {
   flow.collect {  value ->
        val result: O = doYourTransformationOrManipulationHere(value)
        yield(result)
   }
}
```
Where `flowBuilder` matches the construction function for your flow (such as `flow {}` or `channelFlow {}`), and yield is the function that will `emit` the value to the flow (such as `send` for channelFlow).

## More usages

Something that had eluded me for a long while was doing aggregation functions within the `Flow` API. Let's suppose we have a stream of values, and we want to emit the average of the last `N` results received. We can utilize this composition pattern as such:

```kotlin
val infiniteFlow: Flow<Int> = flow {
    while(true) {
        emit(Random.nextInt())
    }
}

fun averageOfLastFlow(n: Int) = flow {
    val buffer = ArrayDeque<Int>()
    infiniteFlow.collect { value ->
        buffer.add(value)
        if(buffer.size > n) {
            emit(buffer.average())
            buffer.removeFirst()
        }
    }
}
```

This is a very simple example, but it shows how we can compose flows to do more complex operations. In this case, we are using a `ArrayDeque` to store the last `N` values, and then emitting the average of those values, and removing the first value in the buffer to store the next emission. One could use this for different grouping and/or data massaging functions as needed.

## Conclusion

Flows are a powerful tool, and while the documentation is very good, I think this pattern is not as well known as it should be. I hope this article helps you in your journey to compose flows in a more powerful way!