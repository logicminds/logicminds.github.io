---
layout: post
title: "Testing DataTypes with the Puppet Debugger"
date: 2017-03-25 10:17:23 -0700
comments: true
aliases: [/posts/testing_datatypes/]
sharing: true
categories: [devops, puppet, testing, debugger]
---
The puppet language comes with a lot of extremely useful syntax and concepts.  However, sometimes it is difficult to understand
how these work or how to use them.  Datatypes are found in almost every programming language so it is no surprise that puppet 4 has a similar feature for validating parameter data.  If you have never used a puppet datatype before have a look [here](https://docs.puppet.com/puppet/4.9/lang_data_type.html) first.

The other day I was showing my client the awesome power of puppet datatypes and I was using the [puppet-debugger](https://github.com/nwops/puppet-debugger) to illustrate how datatypes work.  By using the puppet debugger the client was immediately able to understand datatypes without writing a manifest, so I wanted to share how to use datatypes using the debugger to the rest of the world because datatypes can be very complex and using the debugger will help you understand how they work.

Let us first detail how to use a datatype with puppet parameters.

## Using a datatype
Below is a snippet of a puppet defined type that has no datatype validations.  Obviously the result of running this code would fail.

```ruby
define foo(
  $bar = 'barbar'
  ) {
    file{'/tmp/test':
      ensure => $bar,
    }
  }
  foo{'test': bar => [1,2,3]}
```

So to prevent an error, we should be checking the value of `bar` in the parameter. Previously you might have used `validate_string($bar)` which was a function from the [stdlib](https://github.com/puppetlabs/puppetlabs-stdlib) module that performed validations.  This validation would occur only when the catalog is applied to a system. But that is way too late, we need to fail faster.  So if you want an error to popup well before you deploy the code you need to inform Puppet that you expect a certain type of data.  Hence the word 'datatype'.

Adding a datatype would cause the compilation to fail immediately at compile time which you can check by writing a unit test or running puppet apply.  

```ruby
define foo(
  Enum['directory', 'present', 'file'] $bar,
  ) {
    file{'/tmp/test':
      ensure => $bar,
    }
  }
  foo{'test': bar => 'present' }
```

However, if you are not ready to jump into unit testing, you can utilize the [puppet-debugger](https://github.com/nwops/puppet-debugger) to bridge the gap.
Using the [puppet-debugger](https://github.com/nwops/puppet-debugger) is the easiest, fastest way to validate that the parameter datatype accurately reflects the type of data your manifest requires.

## Installing the Puppet Debugger
To get started you need to install the [puppet-debugger](https://github.com/nwops/puppet-debugger) on any system with puppet >= 3.8

`gem install puppet-debugger`

### Testing a DataType with the Debugger
In order to test the data against the datatype we are going to use the `=~` operator.  This operator lets us test out the datatype.
The data being tested goes on the LHS (left hand side) while the datatype goes on the RHS (right hand side).

Examples:

  1. `true =~ Boolean`
  2. `'true' =~ String`
  3. `'https://www.google.com' =~ Stdlib::HttpsUrl`

{% img [class names] /images/datatypes.gif 'Puppet Datatypes Demo' 'Puppet Datatypes Demo' %}

If the value on the LHS matches the requirements on the RHS, puppet will return true, other false is returned and the
value does not match the datatype.

### Testing a DataType with the Debugger using the new function

Another way to test your datatype is use the [new operator](https://docs.puppet.com/puppet/latest/function.html#new_.  This only works with some datatypes like structures and core types.
But if you have a complex datatype like a structure you can use the new operator to test out the custom datatype.  In this example puppet shows an error because
we did not specify an owner attribute on line 11.

[More Info about Abstract Data Types](https://docs.puppet.com/puppet/4.9/lang_data_abstract.html)

```ruby
1:>> type MyType = Struct[{
  2:>>         mode => Enum[read, write, update],
  3:>>         path            => Optional[String[1]],
  4:>>         NotUndef[owner] => Optional[String[1]]}]
5:>> MyType.new({
  6:>> mode => 'read',
  7:>> path => '/dev/null',
  8:>> owner => 'root'
  9:>> }
  10:>> )
 => {
   "mode" => "read",
  "owner" => "root",
   "path" => "/dev/null"
}
11:>> MyType.new({mode => 'read'})
 => Evaluation Error: Error while evaluating a Method call, Converted value from MyType = Struct[{'mode' => Enum['read', 'update', 'write'], 'path' => Optional[String[1, default]], NotUndef['owner'] => Optional[String[1, default]]}].new() has wrong type, expects a value for key 'owner' at /var/folders/v0/nzsrqr_n40d4v396b2bqjvdw0000gp/T/puppet_debugger_input20170424-91861-6grwv7.pp:1:11
12:>>
```



As you can see above the [puppet-debugger](https://github.com/nwops/puppet-debugger) makes it dead simple to test datatypes.  For repeatable datatype testing you will want to write unit tests instead which is not covered in this article. Additionally, the [stdlib](https://github.com/puppetlabs/puppetlabs-stdlib#data-types) module also has some nice datatypes that you can use in your modules.

I hope this has been of value to you, please share with others and star the [puppet-debugger](https://github.com/nwops/puppet-debugger) project if you enjoy using it.
