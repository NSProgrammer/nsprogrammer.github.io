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

Having used `NSOperation` since its introduction *Mac OS X 10.5*, I have seen it go through some bumpy rides.
It definitely has a history of bugs that have hurt the developer community's perception for this API, but as of *iOS 10* / *macOS 10.12*
it has stabilized and is very much *the* way to organize large units of work on Apple platforms.
As interesting as going through the history of problems would be, that is in the past and we can focus on it being solid now.
That's means (at the time of this post) there are 5 releases worth of rock solid support, so it isn't the same risk as it has been in the past.

Putting everything "substantial" in an `NSOperation` gives you many features to leverage that would be complicated to build on your own,
not to mention inefficient compared to Apple's own optimized API.  I'm not going to go through the benefits or features in this post,
just keeping it on topic to *"how to subclass NSOperation"*.
  
## NSOperation Intro

`NSOperation` API comes with

## Subclassing


{% highlight objc %}
@interface NSOperation : NSObject
@end
{% endhighlight %}

