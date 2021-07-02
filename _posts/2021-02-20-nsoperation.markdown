---
layout: post
title:  "NSOperation Subclassing"
date:   2021-02-20 12:00:00 -0800
categories: jekyll update
---

## Why this post?

There are plenty of resources on [`NSOperation`](https://developer.apple.com/documentation/foundation/nsoperation) out in there,
so it is probably just more noise in the ether.

However, it is also one of the questions I get the most from industry colleagues, because there's nuance and that makes it tough
to commit to memory or concretely discover with a Google search.

So this post will not go into the benefits of `NSOperation` or give a history review of the API over the years or even
provide examples of using the API... it is just going to document how to subclass `NSOperation` so that it can be used
for your use cases.

## Short NSOperation Preamble 

Having used `NSOperation` since its introduction in *Mac OS X 10.5*, I have seen it go through a bumpy ride.
It definitely has a history of bugs that have hurt the developer community's perception for this API, but as of *iOS 10* / *macOS 10.12*
it has stabilized and is very much *the* way to organize large units of work on Apple platforms.
As interesting as going through the history of problems would be, that is in the past and we can focus on it being solid now.
That's means (at the time of this post) there are 5 releases worth of rock solid support, so it isn't the same risk as it has been in the past.

Putting everything "substantial" in an `NSOperation` gives you many features to leverage that would be complicated to build on your own,
not to mention inefficient compared to Apple's own optimized API.  I'm not going to go through the benefits or features in this post,
just keeping it on topic to *"how to subclass NSOperation"*.

### Note on `async await`

The new `async await` available for iOS 15+ introduced at WWDC 2021 is an excellent addition to Swift that will
offer a great deal of value to developers doing async and concurrent programming.  There is no competition between
`async await` and `NSOperation` though; saying that `NSOperation` is no longer useful since `async await` was
introduced would be akin to saying `NSOperation` is no longer useful because `GCD` exists.  They serve different purposes
and actually complement each other.  You can implement your `NSOperation` (`Operation` in Swift) using `async await` without issue!

## NSOperation Intro

`NSOperation` effectively offers an API to encapsulate a unit of work that can be submitted to a priority queue,
an [`NSOperationQueue`](https://developer.apple.com/documentation/foundation/nsoperationqueue).

The `NSOperation` class itself is not terribly useful.  To really have utility, it must be subclassed so that "work" can
be executed and the pattern of `NSOperation` can be adopted to make use of all the benefits that `NSOperation` has to offer.

Apple gives use two concrete subclasses as a way to get started, which effectively enables 3 specific ways of
performing work with an `NSOperation`.

1. Block based work can use `NSBlockOperation`
2. Objective-C object based work can use `NSInvocationOperation`
  - This makes it possible to have the work performed via `target` and `selector`, or via an `NSInvocation`

As valuable as having these convient concrete classes are, more control can be necessary for complex operations.
This is where custom subclasses come in.

## Anatomy of an NSOperation

The [`NSOperation`](https://developer.apple.com/documentation/foundation/nsoperation) has a number of properties that control how it works (obviously).

There are KVO properties for controlling the operation execution and signal its progress:
- `isCancelled`
- `isExecuting`
- `isFinished`
- `isReady`

There are properties controlling it's exectuion:
- `qualityOfService`
- `queuePriority`
- `dependencies`
- `completionBlock`

The property to distinguish its operating behavior:
- `isAsynchronous`

## Asynchronous vs Synchronous Operations

Before we go into the other parts of `NSOperation`, it is worth calling out up front that there are two modes
for `NSOperation`: `isAsynchronous` being `NO` and `isAsynchronous` being `YES`.

This property disambiguates the way the operation operations (asynchronously vs synchronously), but actually
completely splits *how* we subclass `NSOperation`.  This is probably the most important callout to make when
subclassing `NSOperation` -- the single base class has two completely different patterns to follow when subclassing
based on whether the operation is asynchronous or synchronous.

Effectively, synchronous `NSOperation` subclasses can defer all of the KVO related bookkeeping to the super class'
implementation.  They just need to implement the "work", to run synchronously, and that is it.  All the Apple provided
concrete `NSOperation` classes (`NSBlockOperation` and `NSInvocationOperation`) are synchronous.

When it comes to asynchronous subclassing, there is a great deal of care and nuance required to properly having
things operate within the system.

## The KVO Properties of NSOperation

The execution of an `NSOperation` (and other dependent `NSOperation` instances for that matter) is gated by the state of
the KVO state properties.

First is `isReady`.  While `isReady` is `NO`, the operation will not be dequeued by the `NSOperationQueue`.   Once `isReady`
becomes `YES`, the operation can start (as long as there is capacity for the `NSOperationQueue` to run it, of course).

Then, when the operation starts, it moves `isExecuting` from `NO` to `YES`.

Then the work happens, whether it is synchronous or asynchronous.

On the work's completion, `isExecuting` goes back to `NO`.

Finally, the operation finishes by moving `isFinished` from `NO` to `YES`.

All of the above is completely managed for you by synchronous `NSOperation` subclasses.  Asynchronous subclasses
must manage this state and their transitions all on their own.

Additionally, there is the `isCancelled` property.  By default, calling `cancel` on an `NSOperation` just flips `isCancelled` to `YES`.
As work is executing, the "working" part of the execution is responsible for checking `isCancelled` at appropriate places and
finishing early (returning early for synchronous operations, and setting `isExecuting` to `NO` and `isFinished` to `YES` for asynchronous operations).

## `isReady` specifically

The `isReady` property deserves some special attention.  The property prevents the property from running in a queue until
it is set to `YES`.  This property is effectively coupled to all the `dependencies` of the `NSOperation` being `isFinished == YES`.
Unfortunately, the implementation details are hidden so if you override `isReady` in your subclass you will not be able to properly reflect
the dependency graph so overriding `isReady` effectively means you are replacing the behavior with your own and the
`dependencies` will no longer have an effect.

Beyond `isReady` being coupled to `dependencies` being finished, it will also flip to `YES` when `isCancelled` is set to `YES`
before the operation has actually started in its queue.  This makes it possible to rapidly have the operation start and immediately
finish, clearing it from the queue and unblocking operations that depend on it.

Ultimately, there is enough nuance and trouble around `isReady` that it is best to *never* override it.  If you need to apply custom
behavior to have `isReady` depend on, the simplest choice is to use other `NSOperation` objects as dependencies.
Once you think of `isReady` as just `areAllDependenciesFinished`, it becomes much easier to reason about
and you can easily build blocking behavior for running your operation with other operations as `dependencies`.

## Implementing Synchronous NSOperation

Synchronous `NSOperation` subclassing is the most straightforward option, so let's go through what that can look like.

Per the documentation, all you really need to do is implement `main` with synchronously executing work.

### Objective-C Code

{% highlight objc %}
@interface MySyncOperation : NSOperation
@end

@implementation MySyncOperation

- (void)main
{
    ... run work ...

    if (self.isCancelled) {
       return;
    }
    
    ... run more work ...
    
    if (self.isCancelled) {
       return;
    }
    
    ... run final work ...
}

@end
{% endhighlight %}

### Swift Code

{% highlight swift %}

class MySyncOperation: Operation {

    override func main() {
        ... run work ...
    
        guard !self.isCancelled else { return }
    
        ... run more work ...
    
        guard !self.isCancelled else { return }
    
        ... run final work ...
    }

}

{% endhighlight %}

## Implementing Asynchronous NSOperation

Asynchronous `NSOperation` subclassing has a lot more nuance to it.

Instead of overriding `main`, you will override `start`.  You also need to override the state properties.

It is important to keep things thread safety, you don't know what thread any of the state accessors will be called from.
In my sample implementations, I will use `@synchronized(self)` which effectively yields a recursive pthread mutex
under the hood.  If you want to use a different synchronization mechanism, feel free, but keep a few things in mind.

First, for maximum robustness, you will want to be able to encapsulate multiple reads and writes to different states values within in a critical section.
That means making each state value atomic is insufficient for maximum thread safety.
You might be able to get away with only keeping your state values atomic, but unless you can prove there is a performance need,
encapsulating state reads/writes with critical sections is going to be the simpler choice.

Second, you'll also need to be sure that your critical sections support recursion.
Since we are managing KVO as we update state, that can and will lead to accessing the KVO property while in the critical section.
If you go with atomic state values instead of a critical section pattern, you won't need to worry about recursion (but the race conditions will be more of a risk).

### Objective-C Code

{% highlight objc %}
@interface MyAsyncOperation : NSOperation
@end

@implementation MyAsyncOperation
{
    struct {
        BOOL isCancelled;
        BOOL isExecuting;
        BOOL isFinished;
    } _state;
    dispatch_queue_t _queue;
}

- (instancetype)init
{
    if (self = [super init]) {
        _queue = ... however you get the queue to execute on, does not need to be serial ...
    }
    return self;
}

#pragma mark State Accessors

- (BOOL)isFinished
{
    @synchronized(self) {
        return _state.isFinished;
    }
}

- (BOOL)isExecuting
{
    @synchronized(self) {
        return _state.isExecuting;
    }
}

- (BOOL)isCancelled
{
    @synchronized(self) {
        return _state.isCancelled;
    }
}

#pragma mark Method Overrides

- (void)start
{
    @synchronized(self) {
        if (_state.isCancelled) {
            [self _unsafe_finish];
            return;
        }
        
        [self willChangeValueForKey:@"isExecuting"];
        _state.isExecuting = YES;
        [self didChangeValueForKey:@"isExecuting"];
    }
    
    dispatch_async(_queue, ^{
        [self _run];
    );
}

- (void)cancel
{
    @synchronized(self) {
        if (!_state.isCancelled) {
            [self willChangeValueForKey:@"isCancelled"];
            _state.isCancelled = YES;
            [self didChangeValueForKey:@"isCancelled"];
            
         [self _finish];
       }
    }
}

#pragma mark Private Methods

- (void)_run __attribute__((objc_direct))
{
    if (self.isCancelled) {
        return;
    }

    ... do work ...
    
    if (self.isCancelled) {
        return;
    }
    
    ... do more work ...
    
    if (self.isCancelled) {
        return;
    }
    
    @synchronized(self) {
        [self _unsafe_finish];
    }
}

/* this method must be called from a safely synchronized critical section */
- (void)_finish __attribute__((objc_direct))
{
    const BOOL shouldFinish = !_state.isFinished;
    const BOOL shouldStopExecuting = _state.isExecuting;
    
    if (shouldFinish) {
        [self willChangeValueForKey:@"isFinished"]; // triggers a call to self.isFinished
    }
    if (shouldStopExecuting) {
        [self willChangeValueForKey:@"isExecuting"]; // triggers a call to self.isExecuting
    }
    
    _state.isFinished = YES;
    _state.isExecuting = NO;
    
    if (shouldStopExecuting) {
        [self didChangeValueForKey:@"isExecuting"]; // triggers a call to self.isExecuting
    }
    if (shouldFinish) {
        [self didChangeValueForKey:@"isFinished"]; // triggers a call to self.isFinished
    }
}

@end
{% endhighlight %}

### Swift Code

{% highlight swift %}
// TODO
{% endhighlight %}


