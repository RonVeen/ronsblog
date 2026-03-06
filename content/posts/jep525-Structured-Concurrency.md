---
title: "Java 26 - part 7: JEP 525 - Structured Concurrency"
date: 2026-02-26
draft: false
tags: ["Java", "JEP", "Java 26"]
cover:
  image: "/images/jep525-structured-concurreny.png"
  alt: "Structured Concurrency"
series: ["Java 26"]
series_order: 8
---

**The "No More Thread Leaks" Update: Java’s Structured Concurrency (JEP 525)**
If you’ve ever written concurrent Java code using ExecutorService, you know it can feel like herding cats. 
If one task fails, the others just keep running in the background like zombies, eating up resources and making debugging a nightmare.

Structured Concurrency is here to fix that. It treats a group of related tasks as a single unit of work. 
If the main task is cancelled or fails, all its sub-tasks are automatically shut down. No leaks, no orphans, no headaches.

It is one of three deliverables of Project Loom. The other two begin virtual threads and scoped values.
I wrote a book about the first versions of Structured Concurrency. Unfortunately, the API for Structured Concurrency has undergone a lot of change since then.
So, while the book is still a good resources for learning Virtual Threads and Scoped Values, this post is aimed at the new Structured Concurrency API.

**Show Me the Code!**
The star of the show is StructuredTaskScope. 
It uses the try-with-resources block to define the "lifetime" of your threads.
If StructuredTaskScope is the container for your threads, the Joiner is the logic that controls them. 
It’s a new addition that makes the API incredibly flexible.
Think of a Joiner as the "Policy Maker." 
When you open a scope, you give it a Joiner to tell it how to handle the results (or failures) of the subtasks you’re about to fork.
Why do we need Joiners?
In the old days of ExecutorService, you had to manually write if/else logic to decide what to do if Task A failed but Task B succeeded. 
Joiners automate that logic. JEP 525 provides the most common ones out of the box: `

`Joiner.allSuccessfulOrThrow()`: The "Team Player."     
It waits for every single task to finish successfully. If even one task throws an exception, the Joiner triggers a shutdown for all other tasks and throws an exception itself. 
It’s perfect for when you need a complete set of data to proceed.

`Joiner.anySuccessfulOrThrow()`: The "Speed Demon."  
It’s a race! As soon as the first subtask finishes successfully, the Joiner grabs that result and kills all the other pending tasks immediately. 
This is great for redundant services or searching multiple data sources for the same info.

`Joiner.awaitAll()`: The "Patient One."   
It simply waits for everything to finish, regardless of whether they succeeded or failed. 
This is useful for "fire and forget" cleanup tasks where you just need to ensure the work is done before moving on.

**Example 1: The "All or Nothing" Pattern**
Imagine you need to fetch a user and their order history. If either fails, the whole request is useless.

```java
Response handleRequest() throws InterruptedException, FailedException {
    // 1. Open a scope with a Joiner that expects everything to succeed
    try (var scope = StructuredTaskScope.open(Joiner.<Object>allSuccessfulOrThrow())) {
        
        // 2. Fork your subtasks (they run in their own virtual threads!)
        Subtask<String> user  = scope.fork(() -> findUser());
        Subtask<Integer> order = scope.fork(() -> fetchOrder());

        // 3. Wait for them to finish (or fail)
        // If 'findUser' fails, 'fetchOrder' is automatically cancelled!
        scope.join(); 

        // 4. Grab the results (we know they are ready because we joined)
        return new Response(user.get(), order.get());
    }
}
```

**Example 2: The "First One Wins" Race**
Need to call three different weather APIs and just want the fastest successful response?
```java
String getWeatherData() throws InterruptedException {
    try (var scope = StructuredTaskScope.open(Joiner.<String>anySuccessfulOrThrow())) {
        
        scope.fork(() -> callServiceA());
        scope.fork(() -> callServiceB());
        scope.fork(() -> callServiceC());

        // Returns the result of the first one to finish successfully
        // and cancels the other two immediately!
        return scope.join(); 
    }
}
```

**How is this different from previous JEPs?**
If you haven't looked at Structured Concurrency since the early days (JDK 19/20), things look a bit different now:

- From Future to Subtask: Earlier versions returned a Future when you called fork(). Now, it returns a Subtask. This is intentional—Future suggests you should call .get() whenever you want, but in Structured Concurrency, you must wait for the scope.join() before touching the results.

- Constructors are out, Factories are in: In the last JEP (505), public constructors for StructuredTaskScope were removed. Now, you always use the static StructuredTaskScope.open() methods.

- The Joiner API: The logic for "how to handle results" was moved into the Joiner interface recently, making it much easier to write custom policies for how your tasks should behave.


**What’s New in JEP 525?**
Since this is the 6th preview, the changes are mostly "polishing the silver" to make the API feel perfect:

- Joiner.onTimeout(): You can now define exactly what result a Joiner should return if the clock runs out.

- Cleaner Results: Joiner::allSuccessfulOrThrow() now returns a nice, clean List of results instead of a messy Stream of subtasks.

- Renaming: anySuccessfulResultOrThrow() was a mouthful, so it’s now just anySuccessfulOrThrow().

- Config Flexibility: The static open methods now use UnaryOperator for configuration, making the functional code slightly sleeker.

**The Bottom Line**
JEP 525 is the "finishing touches" release. It makes concurrent Java code look like sequential code, which is the "holy grail" for backend developers. It’s safer, faster, and—thanks to Virtual Threads—incredibly lightweight.
