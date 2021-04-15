---
layout: post
published: true
title: "Testing Your infallibility - Part 1"
date: 2015-04-14 11:45:54 -0700
comments: true
categories: [puppet, chef, rspec, unit testing, testing]
---
Audience:  System Admin, Self taught coder, Computer Scientist

Summary: How to leverage puppet-retrospec to generate your puppet unit test suite

How many of you remember when the spell checker first came out?  It had such a huge impact in everyones life because
instantly everyone who used the spell checker appeared as if they had won the international spelling bee.  How long did
it take you to switch from looking up every single word in a paper dictionary to simply right clicking on a word?
Assuming you grew up in the 90’s, we had to use books to spell check, crazy isn’t it.
Spell checker turned horrible spellers into artisan letter assemblers overnight. With that said, I consider the spell
checker to be the first form of unit testing because the spell checker tested every word of your summer vacation essay
against a simple algorithm to ensure the word passed a basic spelling test.

So what does spell checking have to do with configuration management tools like Chef and Puppet?  Basically, after you
learn some basic CF (Configuration Management) programming you will need to start testing your own code, just like the
spell checker.  If your new to development which includes anybody using configuration management, unit testing your CF
code brings a spotlight to bugs without having to pay much attention. This is especially important because a
sysadmin’s workday is bombarded with distractions, especially from your trusty feline sidekick.  So lets move
forward and review some basic automated testing principles.

When it comes to testing there are two types one must test.  You might have seen these before but these types
are unit and acceptance/integration testing.

## Unit testing
The unit test performs a very simple test to ensure that you do not have syntax errors and your conditional logic works
against a set of supplied assumptions. Does your conditional statements work as intended?  Do your variables
interpolate correctly?  Do the functions you use, perform the magic you expected?  These are the things you should be
asking yourself when building unit tests.

## Writing your first unit test
With regards to configuration management code we will first test against compile time bugs because that is where the bulk of
your mistakes will be caught and its also the fastest.

Part of the problem with new developers and lack of test code is the amount of time taken away from writing the “real” code.
There are plenty of articles on how to write test code, but its often difficult to find something that caters to the
absolute beginner and just getting the code setup to run tests is often too much.  So questions like,  whats a helper?,
whats rspec?, mocking, mocha, shoulda, doubles, fixtures, unit test, integration test, a/b test? The problem is that you
already don’t know what your doing and to make matters worse, now you have to learn new terminology to test the code
that you barely know how to write. The mountain of knowledge needed to run a basic test serves as a barrier to becoming
a better programmer and is often what separates a junior from a senior level programmer.  Testing should be just as easy
as writing the code itself. While I cannot change the fact that good testing practices is a skill in itself. I can at
least automate some basic testing patterns and remove the immediate barrier to becoming a better configuration
management programmer.

With this in mind I would like to introduce [Retrospec](https://github.com/logicminds/puppet-retrospec). Retrospec is a
tool that will generate puppet rspec test code based on the code inside the manifests directory.  Its sole purpose is to
get your module setup with automated testing, stat!  Additionally, Retrospec will actually write some basic test code
for you.  Yes, you heard me right, it will write tests for you, even while you sip beer.  While the generated tests are only
basic checks it lays the groundwork for you to create even more advanced test cases and increase your BeerOps time.  So
lets get started.

The first thing you need to do is install retrospec  `gem install puppet-retrospec`

We will use a puppet module I wrote for non-root environments as an example.  Go ahead and clone this repo so you
can follow along.

```
  git clone https://github.com/logicminds/puppet-nonrootlib
  cd puppet-nonrootlib
```

Now we just need to do some house cleaning to show how retrospec generates files easily.  Obviously, you won't do this
in your own module, unless you really want to.
```
  rm -rf spec/  # helps show magic
  rm -f Gemfile Rakefile .fixtures.yml
```

You can run retrospec -h by itself to see what options you have.
```
$ puppet retrospec -h
Options:
        --module-path, -m <s>:   The path (relative or absolute) to the module
                                 directory (Defaults to current directory)
       --template-dir, -t <s>:   Path to templates directory (only for
                                 overriding Retrospec templates)
  --enable-user-templates, -e:   Use Retrospec templates from
                                 /Users/cosman/.puppet_retrospec_templates
    --enable-beaker-tests, -n:   Enable the creation of beaker tests
   --enable-future-parser, -a:   Enables the future parser only during
                                 validation
                   --help, -h:   Show this message

```


Now that we have a module without tests we can use Retrospec to retrofit the module with some basic tests.
Many of these basic files are needed for the puppetlabs_spec_helper and rspec-puppet.

*Note:* If you have set your bundle path to be something other than GEM_HOME you will need to install retrospec in that path.
      `bundle exec gem install puppet-retrospec`, otherwise you will get an error.


```
  $ puppet retrospec     # no need for -m if in the module directory
  + /private/tmp/puppet-nonrootlib/Gemfile
  + /private/tmp/puppet-nonrootlib/Rakefile
  + /private/tmp/puppet-nonrootlib/spec/
  + /private/tmp/puppet-nonrootlib/spec/shared_contexts.rb
  + /private/tmp/puppet-nonrootlib/spec/spec_helper.rb
  + /private/tmp/puppet-nonrootlib/.fixtures.yml
 !! /private/tmp/puppet-nonrootlib/.gitignore already exists and differs from template
 !! /private/tmp/puppet-nonrootlib/.travis.yml already exists and differs from template
  + /private/tmp/puppet-nonrootlib/spec/classes/
  + /private/tmp/puppet-nonrootlib/spec/classes/nonrootlib_spec.rb
  + /private/tmp/puppet-nonrootlib/spec/classes/rpm_spec.rb
  + /private/tmp/puppet-nonrootlib/spec/defines/
  + /private/tmp/puppet-nonrootlib/spec/defines/init_script_spec.rb
  + /private/tmp/puppet-nonrootlib/spec/defines/sysconfig_spec.rb
```

Bam!  Wasn't that easy?

Now Retrospec isn't perfect but it did save us several hours of BeerOps time for us by generating all these files. You
should see something similar like below when you run `cat spec/classes/nonrootlib_spec.rb`.  This gives us a easy place
to start.  Keep following and I'll show you some basic testing patterns to fix some of this generated code.  Retrospec
only generates files that you do not already have as noted by the `!!` and `+` symbols.

```ruby
require 'spec_helper'
require 'shared_contexts'

describe 'nonrootlib' do
  # by default the hiera integration uses hiera data from the shared_contexts.rb file
  # but basically to mock hiera you first need to add a key/value pair
  # to the specific context in the spec/shared_contexts.rb file
  # Note: you can only use a single hiera context per describe/context block
  # rspec-puppet does not allow you to swap out hiera data on a per test block
  #include_context :hiera


  # below is the facts hash that gives you the ability to mock
  # facts on a per describe/context block.  If you use a fact in your
  # manifest you should mock the facts below.
  let(:facts) do
    { }
  end
  # below is a list of the resource parameters that you can override.
  # By default all non-required parameters are commented out,
  # while all required parameters will require you to add a value
  let(:params) do
    {
      #:install_core => $home_dir,
      #:owner => $id,
      #:group => $id,
      #:etc_dir => $home_dir/etc,
      #:bin_dir => $home_dir/bin,
      #:var_dir => $home_dir/var,
      #:usr_dir => $home_dir/usr,
      #:tmp_dir => $home_dir/tmp,
      #:initd_dir => $home_dir/etc/init.d,
      #:lib_dir => $home_dir/lib,
      #:sysconfig_dir => $home_dir/etc/sysconfig,
      #:run_dir => $home_dir/var/run,
      #:lock_dir => $home_dir/var/lock,
      #:subsys_dir => $home_dir/var/lock/subsys,
      #:log_dir => $home_dir/var/log,
    }
  end
  # add these two lines in a single test block to enable puppet and hiera debug mode
  # Puppet::Util::Log.level = :debug
  # Puppet::Util::Log.newdestination(:console)
  it do
    is_expected.to contain_file('[$install_core, $etc_dir, $bin_dir, $var_dir, $usr_dir, $tmp_dir, $initd_dir, $lib_dir, $sysconfig_dir, $run_dir, $lock_dir, $subsys_dir, $log_dir]').
             with({"ensure"=>"directory"})
  end
  it do
    is_expected.to contain_file('$home_dir/bin/service').
             with({"ensure"=>"present",
                   "content"=>"template(nonrootlib/service.erb)",
                   "require"=>"File[$bin_dir]"})
  end
  it do
    is_expected.to contain_file('$home_dir/.bash_profile').
             with({"ensure"=>"present",
                   "content"=>"template(nonrootlib/.bash_profile.erb)",
                   "require"=>"File[$install_core]"})
  end
end
```

### Test Prep
Once you retrospec your module many tests are generated but need to be prepped for testing.  Note the tests will fail
until you refactor the test code.  Since testing puppet code relies heavily on other gems we need to use bundler
to download all these dependencies.

  0. `cd puppet-nonrootlib`  # if not already in the directory
  1. `gem install bundler`  # unless bundler is already installed
  2. `bundle install`  # installs all the gems necessary for puppet unit testing ( You should be in the module directory)
  3. `bundle exec rake spec_prep` # sets up fixtures, not necessary if using rake spec
  4. `bundle exec rspec spec/classes/nonrootlib_spec.rb`  # run your test against a single test file

Normally you would run `bundle exec rake spec` but I wanted to just run a single test file so I used rspec directly.

Lets go ahead and open the spec/classes/nonrootlib_spec.rb file and
[refactor](http://en.wikipedia.org/wiki/Code_refactoring) the test code, because out of the box, these tests will fail.

### Basic mocking
Below is an example of how you [mock](http://en.wikipedia.org/wiki/Mock_object) facts in a rspec-puppet testing
environment. I refactored spec/classes/nonrootlib_spec.rb to work by specifying the facts to mock like home_dir and id.
Note, these are just mocks so directories and users don't actually have to exist. I only need to mock the facts that are
used in my manfiest code.  Go ahead and update your spec/classes/nonrootlib_spec.rb file to match the facts block below.

```ruby
  let(:facts) do
    { :home_dir => '/home/user1', :id => 'user1' }
  end
```

*Note:*
You can only mock hiera values, facts, params, and functions with rspec-puppet.  This is all you really need to influence
conditional logic in your code as your will be testing against the catalog.  I am only covering facts and params in this
article since the other items are considered an advanced topic.  Plus your attention span can't handle much more anyways.

### Parameter Mocking
Below is a real example of how you can mock parameters to incluence your conditional logic.  It follows the same syntax
as mocking facts.  You may have noticed that retrospec comments out any parameters with default values.  However,
I specified each parameter value statically as I consider specifying the parameter values better for long
term test maintainability. If a future developer changes the default parameter values in your manifest code some of
these test will break, so its good practice to set them in stone here.  But your not required to do this which is why
they are commented out in the initial test generation.  This is where you can mock parameter values and test against
different scenarios.  Its worth noting that without Retrospec you would had to specify every parameter by hand.

```ruby
let(:params) do
    {
      :install_core => '/home/user1',
      :owner => 'user1',
      :group => 'user1',
      :etc_dir => '/home/user1/etc',
      :bin_dir => '/home/user1/bin',
      :var_dir => '/home/user1/var',
      :usr_dir => '/home/user1/usr',
      :tmp_dir => '/home/user1/tmp',
      :initd_dir => '/home/user1/etc/init.d',
      :lib_dir => '/home/user1/lib',
      :sysconfig_dir => '/home/user1/etc/sysconfig',
      :run_dir => '/home/user1/var/run',
      :lock_dir => '/home/user1/var/lock',
      :subsys_dir => '/home/user1/var/lock/subsys',
      :log_dir => '/home/user1/var/log',
    }
end
```

### Basic Tests
Since the manifest code creates a bunch of directories you can speed up your test creation by using the ruby
each iterator and iterate around the resources you are creating by defining an array. This only works because all the
resources have the same attributes with the exception of the name.  Alternatively, you could statically define each test
case as well, but call me lazy.   Rspec-puppet which is the testing library required for testing puppet code will
query the catalog that puppet generated during the testing process.

When you write a test it should mentally read, "The manifest named nonrootlib when compiled into a catalog is expected to contain
the file resource XX with ensure set to directory."

```ruby
 dirs = ['/home/user1', '/home/user1/etc', '/home/user1/bin','/home/user1/usr', '/home/user1/tmp', '/home/user1/etc/init.d',
    '/home/user1/lib', '/home/user1/etc/sysconfig', '/home/user1/var/run', '/home/user1/var/lock', '/home/user1/var/lock/subsys',
    '/home/user1/var/log'
    ]
 dirs.each do | dir|
     it do
       is_expected.to contain_file(dir).with({"ensure"=>"directory"})
     end
 end
```

I have also gone ahead and removed the content line from the service and bash_profile resource because I will discuss
verifying content in a future article.  So go ahead and replace the contents below in your own
spec/classes/nonrootlib_spec.rb file just like below.

```ruby
  it do
    is_expected.to contain_file('/home/user1/bin/service').
             with({"ensure"=>"present",
                   "require"=>"File[/home/user1/bin]"})
  end
  it do
    is_expected.to contain_file('/home/user1/.bash_profile').
             with({"ensure"=>"present",
                   "require"=>"File[/home/user1]"})
  end

```

The finished test code after refactor.
```ruby
require 'spec_helper'
require 'shared_contexts'

describe 'nonrootlib' do

  # below is the facts hash that gives you the ability to mock
  # facts on a per describe/context block.  If you use a fact in your
  # manifest you should mock the facts below.
  let(:facts) do
    { :home_dir => '/home/user1', :id => 'user1' }
  end
  # below is a list of the resource parameters that you can override.
  # By default all non-required parameters are commented out,
  # while all required parameters will require you to add a value
  let(:params) do
    {
      :install_core => '/home/user1',
      :owner => 'user1',
      :group => 'user1',
      :etc_dir => '/home/user1/etc',
      :bin_dir => '/home/user1/bin',
      :var_dir => '/home/user1/var',
      :usr_dir => '/home/user1/usr',
      :tmp_dir => '/home/user1/tmp',
      :initd_dir => '/home/user1/etc/init.d',
      :lib_dir => '/home/user1/lib',
      :sysconfig_dir => '/home/user1/etc/sysconfig',
      :run_dir => '/home/user1/var/run',
      :lock_dir => '/home/user1/var/lock',
      :subsys_dir => '/home/user1/var/lock/subsys',
      :log_dir => '/home/user1/var/log',
    }
  end
  # add these two lines in a single test block to enable puppet and hiera debug mode
  # Puppet::Util::Log.level = :debug
  # Puppet::Util::Log.newdestination(:console)
  dirs = ['/home/user1', '/home/user1/etc', '/home/user1/bin','/home/user1/usr', '/home/user1/tmp', '/home/user1/etc/init.d',
   '/home/user1/lib', '/home/user1/etc/sysconfig', '/home/user1/var/run', '/home/user1/var/lock', '/home/user1/var/lock/subsys',
   '/home/user1/var/log'
   ]
  dirs.each do | dir|
    it do
      is_expected.to contain_file(dir).with({"ensure"=>"directory"})
    end
  end
  it do
    is_expected.to contain_file('/home/user1/bin/service').
             with({"ensure"=>"present",
                   "require"=>"File[/home/user1/bin]"})
  end
  it do
    is_expected.to contain_file('/home/user1/.bash_profile').
             with({"ensure"=>"present",
                   "require"=>"File[/home/user1]"})
  end
end
```

### Working Example
At this point your ready to test and should see similar output (I have omitted some deprecation warnings)

```
$ bundle exec rake spec_prep
$ bundle exec rspec spec/classes/nonrootlib_spec.rb


nonrootlib
  should contain File[/home/user1] with ensure => "directory"
  should contain File[/home/user1/etc] with ensure => "directory"
  should contain File[/home/user1/bin] with ensure => "directory"
  should contain File[/home/user1/usr] with ensure => "directory"
  should contain File[/home/user1/tmp] with ensure => "directory"
  should contain File[/home/user1/etc/init.d] with ensure => "directory"
  should contain File[/home/user1/lib] with ensure => "directory"
  should contain File[/home/user1/etc/sysconfig] with ensure => "directory"
  should contain File[/home/user1/var/run] with ensure => "directory"
  should contain File[/home/user1/var/lock] with ensure => "directory"
  should contain File[/home/user1/var/lock/subsys] with ensure => "directory"
  should contain File[/home/user1/var/log] with ensure => "directory"
  should contain File[/home/user1/bin/service] with ensure => "present" and require => "File[/home/user1/bin]"
  should contain File[/home/user1/.bash_profile] with ensure => "present" and require => "File[/home/user1]"

  Finished in 0.35775 seconds (files took 0.86506 seconds to load)
  14 examples, 0 failures
```

Now that you have created your first rspec-puppet test you should be able to start testing your infallibleness on your
own module and find out just how perfect your code is. With the help of Retrospec this should be pretty easy.  But
remember unit testing is not just about testing your code, its about maintaining code integrity long after you have left.
Because many times when new folks are added to a team they make a ton of mistakes until they are familiar with the code
base.  So its important to build a safety net for them with basic unit tests that allows them to test against a feature set
outlined in the test code.

There are many things that I did not discuss in this article that are very important but Retrospec automated these
things such as [.fixtures.yml](https://github.com/puppetlabs/puppetlabs_spec_helper#using-fixtures), spec_helper,
Rakefile, Gemfile and others. So stay tuned and I'll cover the these items in a later article.
