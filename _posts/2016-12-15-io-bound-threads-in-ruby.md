---
layout: post
author: Mike Kim
title: "IO Bound Threads in Ruby"
description: "MRI threads can sometimes run in parallel"
category: articles
tags: [ruby, threads, IO-bound, MRI]
image: 
        feature: soft-trees.jpg
---

In the past couple of years, I've been hearing more and more about Ruby's performance issues. A [recent benchmark](https://www.techempower.com/benchmarks/) shows Ruby frameworks lagging behind many other frameworks. And then we have the infamous [Twitter incident](https://carlosbecker.com/posts/twitter-drops-ruby-bullshit/). While this may be cause for some concern, I feel that Ruby's ease of use and expressiveness more than make up for its lack of performance, _in certain use cases_. Of course we would never use Ruby to train a multi-stage neural network with tens of millions of parameters. However, Ruby is quite well suited for prototyping and for web applications that don't have Google-scale traffic. (That means most of them!)

Nevertheless, one way to increase Ruby's performance is to use threads, and thus leverage the parallelism offered by many of today's multi-core processors. However, one flavor of Ruby, and the one that I'm referring to in this post, [MRI](https://en.wikipedia.org/wiki/Ruby_MRI), is notorious for not supporting true parallelism due to its [global interpreter lock](http://www.jstorimer.com/blogs/workingwithcode/8085491-nobody-understands-the-gil). The GIL, in essence, promises thread safety at the expense of performance.

Let me demonstrate through some code.

Let's try to run some nonsense operations many times with and without threads. I'm using the following code.

{% gist 634172347c7480a50edb3ce37b1ab418 cpu_bound.rb %}

And here are the results on my machine (a quad-core Macbook Pro from 2014).

{% gist 634172347c7480a50edb3ce37b1ab418 cpu_bound_results.sh %}

Even though I'm running on a quad-core machine capable of running up to 8 threads simultaneously, we don't see any performance improvements from using threads. This is due to the GIL.

### Parallelism within the Ruby Community is Misunderstood

I've met several experienced Ruby developers eschew the use of Ruby threads because of the restrictions imposed by the GIL. I think they're only partially right. For CPU-bound operations, you will not see any performance improvements when using threads because thread scheduling is handled within the Ruby process (user space).

I think those Ruby devs who avoid using threads are only partially right because there are a class of operations where one can achieve parallelism using Ruby threads. IO-bound operations are tasks whose speed is limited by the speed of a computer's I/O. These IO operations are handled outside of the Ruby interpreter in kernel space. The kernel is able to schedule threads simultaneously on multiple cores. (You can find an excellent explanation of kernel and user space and how it applies to Ruby threads in Christoph Schiessl's [blog post](http://www.csinaction.com/2014/10/10/multithreading-in-the-mri-ruby-interpreter/)).

Let's try another example using IO-bound operations.

{% gist 634172347c7480a50edb3ce37b1ab418 io_bound.rb %}

And here are the corresponding results.

{% gist 634172347c7480a50edb3ce37b1ab418 io_bound_results.sh %}

The key column to look at is the last column which tells us the actual clock time that has elapsed. As you can see, using threads has decreased the required time by approximately 5 times. We can achieve parallelism with MRI!

### Conclusion

MRI's GIL prevents true parallelism from occuring _only for CPU-bound tasks_. If you're dealing with an IO-bound task and you need a performance boost, consider employing Ruby threads.
