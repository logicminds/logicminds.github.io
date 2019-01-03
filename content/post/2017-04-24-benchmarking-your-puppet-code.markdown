---
layout: post
title: "Benchmarking Your Puppet Code"
date: 2017-04-24 18:04:34 -0700
comments: true
sharing: true
categories: [devops, puppet, testing, debugger, benchmark]
---
Did you know the puppet debugger can measure how fast your puppet code runs? Starting with release 0.6.1 of the [puppet-debugger](https://github.com/nwops/puppet-debugger)
you can now perform simple benchmarks against your puppet code.
So if you ever wondered how fast your puppet code is or that custom puppet function you wrote now there is a way to benchmark those things.

All you need to do is enable benchmark mode, and then use the debugger as you normally do.  Every time puppet evaluates your input
a benchmark will be returned with your result of running the puppet code.  This can be helpful when trying to determine
which code might take a long time to evaluate.  A simple use case is measuring your puppet functions.

```ruby
1:>> benchmark
Benchmark Mode On
2:BM>> md5('dsfasda')
 => [
  [0] "1b0590733e5d6ad1f122d95c47558564",
  [1] "Time elapsed 50.57 ms"
]
3:BM>> md5('dsfasda')
 => [
  [0] "1b0590733e5d6ad1f122d95c47558564",
  [1] "Time elapsed 0.64 ms"
]
4:BM>>
```

As an example, running these functions back to back shows how the puppet application lazy loads puppet functions.
The first run took 50 ms which included loading the function, while the second only half a millisecond since the puppet function
was already loaded.

The benchmark feature is now available with [puppet-debugger](https://github.com/nwops/puppet-debugger) release 0.6.1.

`gem install puppet-debugger` or on the web at https://www.puppet-debugger.com

If you have a unique use case I would love to hear about it.

![Benchmark demo](/images/benchmark.gif)
