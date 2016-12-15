---
layout:      post
author:      Mike Kim
title:       "Learn how to achieve parallelism with Ruby I/O Bound Threads"
description: "Learn the secret to achieving true parallelism in the Ruby programming language with I/O bound threads and why the same can't be achieved for CPU bound threads."
main-header: "The Interpreter, the Lock, and the Block: IO Bound Threads and Ruby"
sub-header:  "Yes, MRI threads can sometimes run in parallel"
category: articles
image: 
        feature: soft-trees.jpg
---

When choosing which language to use, Ruby's performance issues seem to come up more and more these days as a reason not to use Ruby. These complaints are not entirely without merit. [This benchmark](https://www.techempower.com/benchmarks/) shows Ruby frameworks lagging behind many other frameworks. And we can't forget the infamous [Twitter incident](https://carlosbecker.com/posts/twitter-drops-ruby-bullshit/). While this may be cause for concern, I feel that Ruby's ease of use and expressiveness more than make up for its lack of performance, in certain use cases. Of course we would never use Ruby to train a multi-stage neural network with tens of millions of parameters. However, Ruby is quite well suited for prototyping, and for web applications that don't have Google-scale traffic. (That means most web applications!)

If you're still worried about performance, and you're not Google-scale, then one possible way to increase Ruby's performance is to use threads. Threads can leverage the parallelism offered by many of today's multi-core processors. One flavor of Ruby, and the one most devs use, [MRI](https://en.wikipedia.org/wiki/Ruby_MRI), is notorious for not supporting true parallelism due to its [global interpreter lock](http://www.jstorimer.com/blogs/workingwithcode/8085491-nobody-understands-the-gil). You can have many threads running concurrently with MRI, but only one thread will ever be running at any moment in time because of the GIL. The GIL, in essence, promises thread safety at the expense of performance.

To demonstrate let's try to run some operations with and without threads. Here's the code.

{% gist 634172347c7480a50edb3ce37b1ab418 cpu_bound.rb %}

And here are the results on my machine (a quad-core Macbook Pro from 2014).

{% gist 634172347c7480a50edb3ce37b1ab418 cpu_bound_results.sh %}

Even though I'm running on a quad-core machine capable of running up to 8 threads simultaneously, we don't see any performance improvements when using threads. This is due to the GIL.

### Parallelism within the Ruby Community is Misunderstood

I've listened to several experienced Ruby developers eschew the use of Ruby threads because of the restrictions imposed by the GIL. You might as well not use Ruby threads since they can't run in parallel the reasoning goes. I think this view is only partially right. For CPU-bound operations, you will not see any performance improvements when using threads because thread scheduling is handled within the Ruby process (user space), and this scheduling is limited by the GIL. (By the way, the previous code example is a CPU bound example.)

Now to my punchline: There are a class of operations where one can achieve parallelism using Ruby threads. IO-bound operations are tasks whose speed is limited by the speed of a computer's I/O. These IO operations are handled outside of the Ruby interpreter in kernel space. Since they are handled outside of the Ruby interpreter IO operations that are executed within threads are no longer restricted from running in parallel by the GIL. The kernel is able to schedule these threads and have them run simultaneously on multiple cores. (You can find an excellent explanation of kernel and user space and how it applies to Ruby threads in Christoph Schiessl's [blog post](http://www.csinaction.com/2014/10/10/multithreading-in-the-mri-ruby-interpreter/)).

Let's try another example using IO-bound operations.

{% gist 634172347c7480a50edb3ce37b1ab418 io_bound.rb %}

And here are the corresponding results.

{% gist 634172347c7480a50edb3ce37b1ab418 io_bound_results.sh %}

The key column to look at is the last column which tells us the actual clock time that has elapsed. As you can see, using threads has decreased the required time by approximately 5 times. We can achieve parallelism with MRI!

### TL;DR

MRI's GIL prevents true parallelism from occuring only for CPU-bound tasks. If you're dealing with an IO-bound task and you need a performance boost, consider employing Ruby threads. As always, make sure to remain thread safe.
