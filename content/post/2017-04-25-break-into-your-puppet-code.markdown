---
layout: post
title: "Break into your puppet code"
date: 2017-04-25 13:15:36 -0700
comments: true
sharing: true
categories: [devops, puppet, testing, debugger, breakpoint]
---
The puppet 4 language introduced a slew of new features that gives us enormous amounts of power and flexibility.  So much so that
puppet code can become complex and often hard to understand at a glance.

A way to combat this complexity is to write unit tests to validate the end state.  However, one of the shortcomings of unit testing with rspec-puppet is that you can only test against the end state.  Because puppet
is a "compiled" language there is not a good way to perform testing on intermediary states like the value of variables.

With this in mind, we need a way to get inside the puppet compiler and see how our code works from a logical standpoint instead of
just a static end state.  The [puppet-debugger](https://github.com/nwops/puppet-debugger) is a tool that allows us to discover and explore the inner workings of puppet code in real time
as if you are standing inside the compiler with a big poking stick.

{% img [class names] /images/aemzh.jpg 'Poke Harder' 'Dr. Farnsworth poking a dead space creature' %}


Below I will show you the steps needed to combine [rspec-puppet](http://rspec-puppet.com/) and the [puppet-debugger](https://github.com/nwops/puppet-debugger) to step into your code during compilation
time.  So bring a stick because there will be many opportunities to poke things.

## Requirements

In order to use the puppet-debugger please follow the steps below.

### Install puppet-debugger gem
Ensure you have installed the puppet-debugger gem

`gem install puppet-debugger`

or

Add it to your Gemfile

`gem 'puppet-debugger', '>= 0.6'`

### Add the [nwops-debug](https://forge.puppet.com/nwops/debug) puppet module

You will also want to include [this](https://forge.puppet.com/nwops/debug) module in your .fixtures.yml file if using rspec-puppet.

```ruby
debug:
   repo: https://github.com/nwops/puppet-debug
```

Puppet 4 also requires us to specify our dependencies in metadata before using any ruby based functions.  So we will need
to add the debug module to the metadata's dependency list.

The module's metadata.json file should look something like this.  For more info on metadata [click here](https://docs.puppet.com/puppet/latest/modules_metadata.html)

```json
"dependencies": [
    {"name":"puppetlabs-stdlib","version_requirement":">= 4.11.0"},
    {"name":"nwops-debug","version_requirement":">= 0.1.1"}
  ]

```

### Install gems and fixtures

1. bundle install
2. bundle exec rake spec_prep

## Usage

To break into our puppet code we need to first set a breakpoint.  This is done by using the `debug::break()` function.
You can put this breakpoint anywhere in your puppet code.  Once inside the debugger REPL, start poking variable values and run other debugger commands.

```ruby
class testmod (
  $var1 = 'value'
) {
  debug::break()
  file{"/tmp/${var1}":
    ensure => present
  }

}
```

Now in order to break into the puppet code you need to have puppet evaulate the code.  This can be done in multiple ways.
The easiest is to use rspec-puppet, but you could also use `puppet debugger` or even `puppet apply` commands.

In the example below we just need to create a simple unit test and then run `bundle exec rake spec`

```ruby

it do
    is_expected.to contain_file("/tmp/value")
        .with({
          "ensure" => "present"
        })
end

```

{% img [class names] /images/debug_rspec.gif 'Debug using rspec' 'Debug using rspec animated gif' %}


As I mentioned above you can also use the puppet debugger directly and playback puppet code.
ie. `puppet debugger --play examples/debug.pp`

{% img [class names] /images/debug_directly.gif 'Debug using puppet debugger command' 'Debug using rspec animated gif' %}


When using puppet apply, just remember you either need to include the class like `include testmod` from within the debugger or puppet code.
Just remember when using with puppet apply puppet will actually make changes so be very careful.
This is why the `puppet debugger` command exist since it does not make changes.

```ruby

root@f6e624d48fe2:~/testmod# puppet apply manifests/init.pp
Ruby Version: 2.2.7
Puppet Version: 4.10.0
Puppet Debugger Version: 0.6.1
Created by: NWOps <corey@nwops.io>
Type "exit", "functions", "vars", "krt", "whereami", "facts", "resources", "classes",
     "play", "classification", "types", "datatypes", "benchmark",
     "reset", or "help" for more information.

From file: init.pp
          1: class testmod (
          2:   $var1 = 'value'
          3:
          4:
          5: ) {
      =>  6:   debug::break()
          7:   file{"/tmp/${var1}":
          8:     ensure => present
          9:   }
         10:
         11: }
1:>> exit
Notice: Compiled catalog for f6e624d48fe2 in environment production in 700.28 seconds
Notice: /Stage[main]/Testmod/File[/tmp/value]/ensure: created
Notice: Applied catalog in 0.03 seconds

```

I hope these examples have given you an overview of all the different ways to invoke the puppet debugger and utilization of the `debug::break()`
function.  You should now be equipped to start poking fun at puppet.

Stay tuned next time while I show how to debug with different facter sets using facterdb and the [puppet-debugger](https://github.com/nwops/puppet-debugger).
