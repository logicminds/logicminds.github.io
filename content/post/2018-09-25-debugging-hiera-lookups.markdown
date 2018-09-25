---
title: "Debugging and Inspecting hiera lookups"
date: 2018-09-25 10:21:49 -0700
comments: true
categories: [debugger, hiera, puppet, devops]
draft: false
toc: false
---

Summary: In this article you will learn how to debug hiera lookups using the traditional methods and more importantly with the puppet-debugger which can cover a multitude of use cases.

Previously, debugging [hiera](https://puppet.com/docs/puppet/4.10/hiera_intro.html) lookups has been limited to the command line and debug logs. If you wanted to debug hiera
lookups you had to use the hiera command and pass in the config file along with the scope (variables, facts) you wanted to use.

```
$ hiera -d -c /etc/puppet/hiera.yaml key1 environment=production

DEBUG: 2017-10-20 13:10:48 -0700: Hiera YAML backend starting
DEBUG: 2017-10-20 13:10:48 -0700: Looking up key1 in YAML backend
DEBUG: 2017-10-20 13:10:48 -0700: Ignoring bad definition in :hierarchy: 'nodes/'
DEBUG: 2017-10-20 13:10:48 -0700: Looking for data source common
DEBUG: 2017-10-20 13:10:48 -0700: Cannot find datafile /etc/puppetlabs/code/environments/production/hieradata/common.yaml, skipping
nil
```

This was a good start in the puppet 3.x days, because we were new to hirea. However, hiera has become a huge part of module development
and is a backbone to putting data in your code.  Thus debugging became super annoying with always having to pass in the scope or override variables.

Then came Puppet 4 which introduced hiera 3,4 and 5 along with many new capabilities.  Furthermore, Puppet 4.7+ now integrates hiera into the same puppet gem.
This integration and new version provided much improved debugging capabilities.  You now get way more output from the puppet [lookup command](https://puppet.com/docs/puppet/5.3/man/lookup.html).
In Puppet 5, Puppet took the old hiera command and renamed it to puppet lookup along with other refinements.  This was great because it ships with puppet and is right there at your fingertips.  More importantly it knows about your puppet
environment, so passing the scope and hiera config file is not necessary.   The explain option is quite useful where there are complex hiera configurations.  Using the puppet lookup --explain
command will help you solve many problems.


```
 ~/testmod $ puppet lookup key1 --explain
Searching for "key1"
  Global Data Provider (hiera configuration version 5)
    Using configuration "/Users/user/github/puppet-debugger-demo/hieradata/hiera.yaml"
    Hierarchy entry "common"
      Path "/Users/user/github/puppet-debugger-demo/hieradata/data/common.yaml"
        Original path: "common.yaml"
        Found key: "key1" value: "hello"
```


But what if your hiera config or hiera data requires even something more complex?  So complex that lookup explain doesn't explain enough.

What if you want to know how hiera interpolation tokens work? How do control which scopes are passed in?

What if you want to play around with [lookup options](https://docs.puppet.com/puppet/4.5/lookup_quick.html#setting-lookupoptions-in-data) and merge behavior?

What if you need to know why hiera is not returning the value you expected?

In order to answer these kinds of questions we need to use the puppet debugger.  You might have previously thought that the [puppet debugger](https://www.puppet-debugger.com) was just for puppet code.
But the puppet debugger can do so much more because how how the language interacts with all these other tools.

Puppet by design allows us to use puppet functions that are executed during the compilation process.  Because the debugger is a interactive REPL we can
execute functions directly in real time.  This functionality is similar to the puppet lookup command but also allows you see and play with the returned
data.

## Steps for Debugging hiera with the debugger
Perform the following steps from your puppet development system

1. Install the debugger  `gem install puppet-debugger`
2. Setup your puppet configuration file for the global hiera tier
  a. `mkdir -p $(puppet config print confdir)`
  b. `touch $(puppet config print config)`
  c. Add the hiera_config setting  `puppet config set hiera_config /home/user1/repos/hieradata/hiera.yaml`  (Your hiera config path will be different)
3. Make sure the hiera config file points to the data directory relative to the location of the hiera config file.
4. The example datadir I have is in the same directory as the hiera.yaml file, feel free to move it elsewhere.

```
.
├── data
│   ├── common.yaml
│   ├── datacenter
│   │   ├── datacenter1.yaml
│   │   └── datacenter2.yaml
│   ├── node
│   │   ├── foo.example.com.yaml
│   │   └── master.local.yaml
│   ├── operatingsystem
│   │   ├── RedHat.yaml
│   │   ├── centos.yaml
│   │   └── ubuntu.yaml
│   └── osfamily
│       └── Debian.yaml
├── hiera.yaml
├── hiera2.yaml
└── otherdata
    └── common.yaml

$ more hiera.yaml
---
version: 5
defaults:
  datadir: data
  data_hash: yaml_data
hierarchy:
    - name: "common"
      path: "common.yaml"
```

Note: you can swap out the hiera config by having multiple configs handy.  So in this exmaple I can change from hiera.yaml to hiera2.yaml

`puppet config set hiera_config /home/user1/repos/hieradata/hiera2.yaml`

Additionally if you want to override the hiera config from the command line you can use

`puppet debugger --hiera_config=/home/user1/repos/hieradata/hiera2.yaml`

*NOTE* the hiera_config is a [puppet config](https://puppet.com/docs/puppet/5.5/configuration.html), and there are many more configs you can swap out that will change the environment the debugger uses.  

### Start debugging
Once the debugger is setup you can start the debugger by running `puppet debugger`

Then use the lookup function to lookup your key

```
Ruby Version: 2.4.1
Puppet Version: 5.3.2
Puppet Debugger Version: 0.8.1
Created by: NWOps <corey@nwops.io>
Type "commands" for a list of debugger commands
or "help" to show the help screen.


1:>> lookup('key1')
 => "hello"
2:>>
```

Not very helpful by itself as you only get the value.  But the scope was injected for you by the debugger.  This
means that the hiera.yaml file interpolates the facts and variables you set in the debugger.

### More output
Turn up the loglevel to "debug" to get the output normally provided with `puppet lookup --explain`

```
Ruby Version: 2.4.1
Puppet Version: 5.3.2
Puppet Debugger Version: 0.8.1
Created by: NWOps <corey@nwops.io>
Type "commands" for a list of debugger commands
or "help" to show the help screen.

1:>> set loglevel debug
loglevel debug is set
1:>> lookup('key1')
  Searching for "key1"
    Global Data Provider (hiera configuration version 5)
      Using configuration "/Users/user1/github/puppet-debugger-demo/hieradata/hiera.yaml"
      Hierarchy entry "common"
        Path "/Users/user1/github/puppet-debugger-demo/hieradata/data/common.yaml"
          Original path: "common.yaml"
          Found key: "key1" value: "hello"
 => "hello"
2:>>

```

This log level provides a more detailed lookup process and can help identify faulty keys and values.

## But Wait!  There is even more debugging to be had
We know from above that we can lookup the key.  And we can also see how the value of that key is derived with the debug log level.  But how do
we know the value of that key is the same datatype that puppet is expecting.

You can [compare the datatype](/posts/testing_datatypes/) returned with the expected datatype with the `=~` operator.

```
1:>> lookup('key1')
 => "hello"
2:>> lookup('key1') =~ String
 => true
3:>>

```

Or something more complex

```
10:>> lookup('foo_complex3') =~ Array[Hash]
 => true
11:>>
```

Or even store the returned value in a variable and have a look

```
11:>> $complex_value = lookup('foo_complex3')
12:>> $complex_value[0]
 => {
  "key1" => [
    [0] {
      "name" => "value1"
    },
    [1] {
      "name" => "value2"
    },
    [2] {
      "name" => "value3"
    }
  ]
}
```

The debugger allows you to get into the nitty gritty details of the variable returned from hiera. Additionally,
you get to mess around and transform the variable contents with looping iterators like map/reduce/dig. You don't have this ability
when using the `explain` method, simply because the debugger is the only way to interact with puppet code.

So start debugging your code and hiera lookups with the debugger and demystify the lookup process.
