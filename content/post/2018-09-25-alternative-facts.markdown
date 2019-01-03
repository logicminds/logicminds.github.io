---
layout: post
title: "Alternative Facts"
date: 2018-09-25 15:26:41 -0700
comments: true
published: true
categories: [debugger, devops, puppet, testing, mocking]
---

What does Puppet and alternative facts have in common?

No, this is not some political rant about how real facts matter.  This article
is about sub-planting alternative puppet facts provided by custom external FacterDB facts.

## But first.  What the hell are FacterDB facts?
[FacterDB](https://github.com/camptocamp/facterdb) facts are artisanal hand-made facter sets from real systems
stashed into a filesystem based "database" and searchable via facts.  Basically just run `puppet facts` and save them off
into git to get an idea.

To summarize, a fact set is snapshot in time of all the facts that make up a particular system.  So this can also be seen as a system level mock.

Why are these useful?  In a large puppet organization you might have multiple operating systems
to deal with.  Remembering what the kernel, os, and other hard to remember fact structures can be
cumbersome. You will likely need to write a case statement at some point and have to remember what values to use.

```
case $facts['os']['name'] {
  'Solaris':           { include role::solaris } # Apply the solaris class
  'RedHat', 'CentOS':  { include role::redhat  } # Apply the redhat class
  /^(Debian|Ubuntu)$/: { include role::debian  } # Apply the debian class
  default:             { include role::generic } # Apply the generic class
}

```

Not to mention the [fact structures](https://github.com/camptocamp/facterdb/tree/master/facts) can change between facter versions.  See 2.x and 3.x fact structures, or even 1.x.

So having every fact structure immediately available to reference is extremely helpful.



## That sounds great, but now what?
The fact of the matter is you can utilize facterdb facts with any supporting
application.  Thereby, making the application assume the mock system you created.

  * [puppet-debugger](https://www.puppet-debugger.com)
  * [rspec-puppet-facts](https://github.com/mcanevet/rspec-puppet-facts)
  * puppet  (where you can supply facts)


### Puppet Debugger Example
  Example of debian and facter 2.4

  ```
  puppet debugger --facterdb-filter 'facterversion=/^2.4./ and operatingsystem=Debian'
Ruby Version: 2.3.7
Puppet Version: 6.0.0
Puppet Debugger Version: 0.10.0
Created by: NWOps <corey@nwops.io>
Type "commands" for a list of debugger commands
or "help" to show the help screen.


1:>> $os
 => {
   "family" => "Debian",
      "lsb" => {
        "distcodename" => "squeeze",
     "distdescription" => "Debian GNU/Linux 6.0.10 (squeeze)",
              "distid" => "Debian",
         "distrelease" => "6.0.10",
      "majdistrelease" => "6",
    "minordistrelease" => "0"
  },
     "name" => "Debian",
  "release" => {
     "full" => "6.0.10",
    "major" => "6",
    "minor" => "0"
  }
}

  ```

  Or how about windows and facter 3.7

  ```
  puppet debugger --facterdb-filter 'facterversion=/^3.7./ and operatingsystemrelease=10'
Ruby Version: 2.3.7
Puppet Version: 6.0.0
Puppet Debugger Version: 0.10.0
Created by: NWOps <corey@nwops.io>
Type "commands" for a list of debugger commands
or "help" to show the help screen.


1:>> $os
 => {
  "architecture" => "x64",
        "family" => "windows",
      "hardware" => "x86_64",
          "name" => "windows",
       "release" => {
     "full" => "10",
    "major" => "10"
  },
       "windows" => {
    "system32" => "C:\\Windows\\system32"
  }
}
```

### Rspec puppet facts example
Unit testing with rspec-puppet-facts allows you to automatically inject facter sets.
This makes it really simple to setup unit tests because you don't need to mock facts
since facterdb does this for you.

Example below tests on 4 different facter sets automatically.   Why we still use `i386` is beyond me.

```
require 'spec_helper'

# Why fixtures dir?  
# Because you want to clone your alternative facts down with rake spec_prep command
ENV['FACTERDB_SEARCH_PATHS'] = File.join(fixtures_dir, 'facterdb_facts')

describe 'myclass::debian' do
  test_on = {
    :hardwaremodels => ['x86_64', 'i386'],
    :supported_os   => [
      {
        'operatingsystem'        => 'Debian',
        'operatingsystemrelease' => ['6', '7'],
      },
    ],
  }

  on_supported_os(test_on).each do |os, facts|
    let (:facts) { facts }
    it { is_expected.to compile.with_all_deps }
  end
end

```

## That's cool, but I still don't know what Alternative facts are.
Alternative facts are simply facter sets that are not part of FacterDB's built in
library of facts.  They are facts made by you or someone else in your organization.
All you need to do is collect a "library" of these alternative facts and tell FacterDB
where they are.  These facts stay external to FacterDB and sometimes contain sensitive
information that should not be published publicly.

You can read up here on [external facts](https://github.com/camptocamp/facterdb#supplying-custom-external-facts).

But the fact is that you just have to set an environment variable when running a tool that uses facterdb.

1. create the fact set `puppet facts | jq '.values' > /tmp/custom_facts/datacenter_a/2.4/os_x.facts` (must have jq command)
2. `FACTERDB_SEARCH_PATHS="/tmp/custom_facts"  puppet-debgugger --facterdb-filter 'facterversion=/^2.4./ and operatingsystem=Darwin'`

Because these facts were created by you, they will likely contain all the custom facts you need to test or debug your code.
And this is a huge deal. Custom puppet modules will always utilize a custom fact for something.

This means you can create a frankenstein system of your own to test with.  Ever wanted
a linux system with a kernel version of 6?  Now you can.

Stay tuned later when I detail how to use rspec-puppet-facts and the puppet-debugger and test a windows 9 system.
