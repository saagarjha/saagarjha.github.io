---
layout: post
title: "Swift Concurrency Waits for No One"
---

> There was once a master engineer who lived by herself in a mystical, far off place, where nourishment flowed freely and the dirt beneath your feet was more valuable than gold. Inaccessible even by foot, only very few could make the trip for a chance at receiving her wisdom. A novice programmer, enthusiastic but still wet behind the ears, visited her one day.  
> 
> "I have read your code," he began, "and I can only describe it as sublime. But, I've been learning a lot about Swift Concurrency and I see that you don't use it all the time. Why is that?"
> 
> The master engineer replied promptly with a question of her own: "Tell me, how would an asynchronous program call synchronous code?"
> 
> "That's easy," he replied. "You can just call it directly."
> 
> "Now, how would one invoke asynchronous code from a synchronous context?"
> 
> "Spawn a `Task`, obviously!" replied the novice engineer, glad that his studies had come in handy.
> 
> The master engineer smiled. "Very good. But now, imagine waiting for the work to complete before proceeding. What then? The API, naturally, is provided by the system and cannot be redesigned."
> 
> This troubled the novice, and he furrowed his brow in concentration for some time. Finally, a half-forgotten memory came back to him: "`DispatchSemaphore`! I can use a semaphore to wait for the asynchronous work!"
> 
> "Swift Concurrency requires tasks to make forward progress", responded the master engineer. Then she fell silent. After a while, the novice was enlightened.

## Background

[Swift Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/) promises to make it possible to write correct, performant code designed for today's world of asynchronous events and ubiquitous hardware parallelism. And indeed, when wielded appropriately it does exactly that. However–much like an iceberg–the simple APIs it exposes hide a staggering amount of complexity underneath. Unfortunately, concurrency is a challenging topic to reason about when compared to straight-line, synchronous code, and it is difficult for any programming model to paper over all of its subtleties.

Concurrency is nothing new for most Swift developers, of course. Those programming for Apple's platforms are almost certainly aware of [Grand Central Dispatch](https://developer.apple.com/documentation/dispatch), or even other APIs such as [POSIX threads](https://en.wikipedia.org/wiki/Pthreads). Many applications will use several of these at once! What's novel, however, is that Swift Concurrency is an entirely different paradigm for concurrent programming, governed by a completely new set of rules. The penalties for violating these conditions are, on occasion, just as severe as they were for the technologies that came before it–unintentional reentrancy, deadlocks, even data corruption. Thankfully, one of the major selling points for Swift Concurrency is that the compiler tries to enforce many of the rules for us. Often, code that violates the guidelines will fail to build! Other rules are well documented and easy to follow, even if they aren't checked by the compiler. Some invariants are unconditionally validated at runtime when practical, and others are available as opt-in debug checks.

Despite being critical for writing correct programs, sometimes the rules are less clear (or not mentioned at all). While the runtime is open source, it is constantly evolving and difficult for non-experts to understand. In pathological cases, it may be impossible to write certain code using Swift Concurrency, but easy to construct something that seems like it works–whether it be due to bugs, implementation details of the runtime, or even just plain luck. As a result, a successful practitioner needs a firm grasp of how to reason about the correctness of their code. We will explore one way to analyze programs by tackling what has historically been a problematic area for many developers: Swift Concurrency's concept of forward progress.

## Concurrency and Parallelism

If you're coming from the world of threads and queues, Swift Concurrency can look somewhat similar on the surface: you can kick off tasks to run asynchronously and wait on their completion, just like you might with older APIs. In many instances, a model of the runtime having "unlimited threads" to run the work we schedule is sufficient for understanding whether some code is correct or not. Obviously, Swift Concurrency manages its own cooperative thread pool under the hood, so we don't actually spawn infinitely many threads, but in these cases it's easy to treat the runtime as "magically" doing the right thing for us.

Occasionally, however, the nature of the cooperative thread pool becomes very important. This is one of those times. Before we talk about how, we need to go over two important terms: concurrency and parallelism. If you're like me your recollection probably extends to them having something to do with managing several jobs at once, but not much further. After all, you can just look up their exact definitions when you need them. That said, the difference matters here so you can review them now or come back if you forget:

* Concurrency lets you have multiple tasks "in progress" (that is: not completed) at any given time.
* Parallelism enables running multiple tasks that are actively executing at the same time.

Less abstractly, when you stop coding and start replying to a message from your boss on Slack, you're practicing *concurrency*. Likewise, when you're scrolling through Hacker News while sitting through a boring meeting you're showing off your *parallelism* skills. How this usually works out in practice for computers is that concurrent scheduling of tasks involves slicing them up and interleaving the parts, while parallel scheduling requires multiple "cores". Note that it is possible to have concurrency without parallelism: a single-core machine does not have the ability to do any work in parallel, but it will typically context switch between multiple tasks.

Swift Concurrency (as its name suggests) is a system for *concurrently* scheduling tasks. One core feature of its implementation is that it can transparently scale to take advantage of all the parallelism on the system, but this detail is exactly that: an *implementation* detail. A correctly written program will, for the most part, not be able to tell the difference between a single thread and a hundred.

{% include aside.html type="Fun fact" content="Typically the underlying cooperative pool is scaled to match the number of cores on the system, since (at least in theory) there isn't much point scheduling more tasks than that at once. However, the runtime is free to choose fewer or greater threads at its discretion. For example, older versions of the iOS simulator used to set the pool size to one, and the `DISPATCH_COOPERATIVE_POOL_STRICT` environment variable lets you opt into this width for debugging purposes." %}

## Forward progress

Forward progress, roughly speaking, means that something needs to be able to keep doing work. A program that makes forward progress can "wait" (e.g. by taking locks, sleeping, or performing I/O) but there must always be a way for it to come out of the wait. One simple example of a program that is not making forward progress is one that is stuck in an infinite loop, because it's never going to be able to do anything else no matter what you try or how long you wait.

In Swift Concurrency, one central rule is that **[all tasks on the cooperative thread pool must make forward progress](https://developer.apple.com/videos/play/wwdc2021/10254/?time=1450)**. Violating this rule will result in deadlocks. With that in mind, can we say anything about the following code?

```swift
let semaphore = DispatchSemaphore(value: 0)

func wait() async {
	semaphore.wait()
}

func signal() async {
	semaphore.signal()
}
```

You may have heard that `DispatchSemaphore` is unsafe to use with Swift Concurrency. The warnings when you build this also point towards that. But why? What is wrong with it?

To start with, we know that `wait()` and `signal()` are both `async` functions, meaning they will be called on the cooperative thread pool. We also know that all tasks here need to make forward progress. We can spot potential trouble immediately: `wait()` calls `DispatchSemaphore.wait()`, which blocks the current thread without notifying the runtime about it as `await` is designed to do. When this happens nothing else can run on that thread until it returns, and this means that it can arbitrarily block forward progress of this task forever.

"But wait!", you say. "Who said anything about blocking forward progress *forever*? Of course I plan to call `signal()` at some point in the future, and *that* will unblock the `wait()` call. See, I can make sure there is forward progress!" However, even if you balance all your calls to `wait()` and `signal()`, this code can still deadlock. How? The answer requires going back to our discussion of parallelism.

The Swift cooperative thread pool is allowed to be any size. This means it can have exactly one thread in it. What happens when you call `wait()` in this scenario? It blocks the sole thread, which means that any future call to `signal()` will never get a chance to be scheduled. It cannot: it *has* to go on the cooperative thread pool, and there are no threads available. Ergo, deadlock.

{% include aside.html content="If you didn't read the previous fun fact, you should probably read it now. We'll proceed without it, though, since it's not critical to the argument. It is an aside, after all." %}

"Hold on!", you protest again. "This is stupid. Why do we even care about a cooperative thread pool of size one? I understand that this is wrong *theoretically* but in practice this is never going to break because there are always more threads." Ok, but you're still looking at deadlocks. Why? Because I know you're not going to write code like my example. Instead, you'll likely write this:

```swift
actor Lock {
	let semaphore = DispatchSemaphore(value: 0)

	func lock() {
		semaphore.wait()
	}

	func unlock() {
		semaphore.signal()
	}
}
```

Who writes an app with just one semaphore? And if you are going to have a bunch, you might as well name them and all. The name doesn't matter, obviously; the important part is what happens when all threads in your app happen to call `lock()` at the same time. This will only happen rarely, but when it does, no further work can happen on the cooperative thread pool and you're deadlocked again. There's nothing special about `Lock`: it's just a stand-in for the more general problem of blocking on multiple threads at once and starving the cooperative pool until it can't service new work anymore.

## Waiting on `async` work

Let's look at a more complicated example. Let's say we have an API for borrowing books. It predates Swift Concurrency and uses a delegate interface:

```swift
protocol LibraryDelegate {
	func shouldLend(_ book: Book) -> Bool
}
```

When the user goes to borrow a book, the framework lets the delegate dissent (maybe they have unpaid fines?). A (simplified) implementation might look like this:

```swift
class CheckoutMachine: LibraryDelegate {
	func shouldLend(_ book: Book) -> Bool {
		let account = library.lookupAccount(forCardNumber: cardNumber)
		return !account.hasFines
	}
}
```

This is a great start, but the library would also like to make sure that we don't check out a book that someone has placed on hold. This could be another library patron, or it can be a researcher at a local university they've set up a catalog-sharing partnership with. Thankfully we have some code for this, too. Interacting with the university system can be a bit slow, but the implementation to talk to it is a little more modern:

```swift
class HoldManager {
	func holds(on book: Book) async -> [Account] {
		let libraryHolds = library.holds(on: book)
		// This reaches out to the university system to see if someone reserved it
		let universityHolds = await university.holds(on: book)
		return libraryHolds + universityHolds
	}
}
```

Now we just update `shouldLend(_:)` to use thi…wait. This is an `async` function, which means we need to `await` its result. But `shouldLend(_:)` is synchronous! How can we make this work?

Clearly, we need to wait for the task to finish somehow. This problem–making asynchronous work synchronous–is super common when interfacing with Swift Concurrency from older code that was designed without it in mind. Bridging synchronous and asynchronous code has never been easy, but in the past we might have used a semaphore for situations like this one. This has its own issues, but in general it will not block forward progress.

{% include aside.html content="The primary issue being the potential for priority inversion when waiting on discretionary work from a high-priority thread, which, depending on system conditions, may not be resolved until an arbitrary point in the future. Since fixing this takes \"a while\" and not \"forever\" it isn't a true deadlock, but it can still be problematic: sometimes this \"kinda\" sucks (e.g. you drop a couple frames because your UI thread blocks); sometimes it *really* sucks (e.g. your app hangs until low power mode disengages). Friends don't let friends use `DispatchSemaphore`."%}

Swift Concurrency's Task initializer is a synchronous function that takes an `async` closure, which at least matches what we are looking for. If we squint at this construction it looks a bit like how we've used to wait for callbacks:

```swift
func shouldLend(_ book: Book) -> Bool {
	class Unreserved: @unchecked Sendable { var value: Bool! }
	let Unreserved = Unreserved()

	let semaphore = DispatchSemaphore(value: 0)
	Task {
		let holds = await holdManager.holds(on: book)
		unreserved.value = holds.isEmpty
		semaphore.signal()
	}
	semaphore.wait()

	let account = library.lookupAccount(forCardNumber: cardNumber)
	return !account.hasFines && unreserved.value
}
```

Because what we're doing is somewhat unusual, we need a little bit of ceremony to hoist the results out of the `Task`. Despite its unsightly appearance, the `@unchecked Sendable` and implicitly unwrapped optional are fine, since our control flow guarantees initialization and exclusive access. Is this code correct, though?

At first glance, this seems like it might be OK: even though we're using `DispatchSemaphore`, this time we only *signal* from an asynchronous context. The blocking wait is on a code path that doesn't "know" about Swift Concurrency at all. However, this can still deadlock!

### Analysis

The rationale for this is not obvious, even though it can be described quite simply: the reason the code deadlocks is that `shouldLend(_:)` might end up being called on the cooperative pool, even though it is a synchronous, pre-Swift Concurrency callback. Here's the actual backtrace of the call for `shouldLend(_:)`:

```console
* thread #2, queue = 'com.apple.root.default-qos.cooperative'
  * frame #0: App`CheckoutMachine.shouldLend(_: Book)
    frame #1: LibraryCore`Library.checkLendability(of: Book)
    frame #2: LibraryCore`Library.reallyDoCheckout(of: Book, from: Catalog)
    frame #3: LibraryCore`Library.checkoutCommonImpl(book: Book)
    frame #4: LibraryCore`Library.checkout(_: Book)
    frame #5: App`CheckoutMachine.checkoutBooks() async throws
```

It's on the cooperative thread pool! This is because in another part of our app, we used `Library.checkout(_:)` from an `async` context, and LibraryCore ended up calling our delegate method. This is bad news, because in our implementation the semaphore blocks this thread. If we reduce parallelism to one, the `Task` we kick off doesn't get a chance to start, and because we rely on its work to signal the semaphore we're waiting on we have yet another deadlock.

This analysis might feel somewhat unfair, since I didn't tell you anything about the rest of `CheckoutMachine` or LibraryCore. But that's the point: we broke the guarantee of forward progress in a surprising, nonlocal way. Even if you have a handle on all the code in your own app (a tall order for any complex project!) the forward progress guarantee depends on scheduling decisions of *every single function in your call stack*, many of which you may not own or even have the source code to. Needless to say, whether something decides to run its code synchronously, on an internal queue, or even use Swift Concurrency itself is an implementation detail you can't rely on.

## Forward progress in legacy code

It's probably obvious at this point that blocking in concurrently-executing code is a recipe for deadlocks. There's a second part to our analysis, though, and it centers around a question you might have had yourself if you read the review above: how can we safely call arbitrary code at all, if we're not allowed to block the cooperative thread pool? After all, some library we use might be implemented internally using a semaphore. Is this a problem for us?

Experimentally, the answer is "no": well behaved Swift Concurrency code does not appear to deadlock, regardless of how the libraries it depends on choose to synchronize themselves. Somehow *we're* not allowed to block or wait, but the code we call synchronously *is*. `DispatchSemaphore` can't be smart enough to know the difference…or is it? Is it capable of punishing us with hangs when we misbehave?

Needless to say, this isn't the case, but the real reason is subtle. We know that code which predates Swift Concurrency might end up running on the cooperative thread pool, but it won't choose to do so *itself*. Library code like this is quite common:

```swift
func foo() {
	let semaphore = DispatchSemaphore(value: 0)
	doSomeAsynchronousWork(completion: {
		semaphore.signal()
	})
	semaphore.wait()
}
```

Even though this looks a lot like our example from above, we can generally call it safely, even from an `async` context. Because this code predates Swift Concurrency, it's not going to spawn a `Task` to do the asynchronous work like we did in our example earlier. It might do a `dispatch_async`, XPC callout, or network call, but none of these will schedule on the cooperative pool. This allows forward progress because the other work will start and complete independently, unblocking our waiting thread at some point in the future.

{% include aside.html type="Warning" content="I said \"generally\" because there is one case where this breaks: if `doSomeAsynchronousWork` ends up being reimplemented in the future to use Swift Concurrency, such that it captures and calls the completion handler in an `async` context, then this code becomes deadlock-prone. This construction is rare today, so I'm mostly just bringing it up as a final reminder of how forward progress invariants can be violated in surprising ways. As Swift Concurrency adoption rises in the future, it's important to remember that even small details–such as where callbacks run–can be serious API-breaking changes with the potential to cause problems for real programs."%}

## Takeaways

Forward progress is a difficult concept to master. While important when reasoning about any concurrent design, in a cooperative system such as Swift Concurrency the repercussions for deadlock are much more pervasive, entangled, and usually harder to debug. Some primitives, notably techniques to wait synchronously without the use of `await`, are all but impossible to use safely from Swift Concurrency. Unfortunately, trying to build a homegrown but subtly incorrect version is far easier than analyzing why it is broken. This, paired with stochastic failures, means there's an an awful lot of code out in the wild with a future full of hangs. (The examples in this post, for example, are derived from searches I performed on GitHub.)

The unwelcome truth is that some code just cannot be bridged with Swift Concurrency. In other cases, the effort needed to write this code and validate its correctness is so exceptionally high that continuing to use the technology is not well justified. This can be particularly painful if discovered long after a choice to use Swift Concurrency was made, and the only solution here (as frustrating as it might be) is to go back and rewrite the program using other APIs.

Despite these limitations, Swift Concurrency remains a good choice for many projects that are seeking to manage asynchronous work; it's just important to understand which those might be. Checking for forward progress is one way we might make such a decision. This analysis can be complex, but the examples presented here should provide a place to start when thinking about the correctness of your own asynchronous code.
