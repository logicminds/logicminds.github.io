---
layout: post
title: "Retrospec - the task generator"
date: 2017-10-12 11:17:30 -0700
comments: true
published: true
categories: [puppet, devops, bolt]
---

Puppet introduced Bolt at Puppetconf 2017 this year and so far I like what I see.  Simple, easy to use remote task execution
without a huge requirement of any one language.  Best of all puppet modules can start adding one off bolt tasks to help with
the administrative duties of various applications.  Bolt makes it really easy to get started but adds some required scaffolding to
create a properly defined task, namely the metadata file.

One of the use cases of [retrospec puppet](https://github.com/nwops/puppet-retrospec) is to build out this scaffolding for you with the many generators it has.  So starting with version 1.5.0, retrospec
can now generate tasks and save you time for other things.

To get started follow these steps:

    1. gem install puppet-retrospec
    2. cd into your favorite module  (example: testmod)
    3. retrospec puppet new_task -n reboot -t bash -p "name, count, amount"

This will create a tasks folder and two files with.

1. the task file
2. the task metadata file with the parameters filled in already.


## Example run:

```
retrospec -m ~/testmod puppet new_task -n reboot -t bash -p "name, count, amount"

 + /Users/cosman/testmod/tasks
 + /Users/cosman/testmod/tasks/reboot.sh
 + /Users/cosman/testmod/tasks/reboot.json
```


All you need to do now is is:

1. Update the task metadata to match your code and parameter types
2. Add code to your task
3. Publish

When you run the generator you can pick from any language and if the one you need is not in the list a generic file
will be created.

https://github.com/nwops/puppet-retrospec


Enjoy
