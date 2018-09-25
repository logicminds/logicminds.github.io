---
layout: post
title: "Leveraging Docker for Puppet Development"
date: 2015-05-28 20:16:05 -0700
comments: true
categories: [devops, puppet, testing]
published: true
---
Last week I decided to format my macbook pro after several years of gem and vagrant clutter.  My computer was suffering
from lag, wasted space, spinning beach balls, and weird crashes.  So after I backed up and formatted, I decided I was
going to do things differently from now on because starting from scratch is such a pain.   Since I had some free time on
hand I thought I could take a chance to explore docker as a development environment.

After reading this great [docker development](https://medium.com/@treeder/why-and-how-to-use-docker-for-development-a156c1de3b24) article I figured I could do the
same for puppet development as puppet development sorta falls under ruby development.

Now since I use bundler bundles for all my gems and puppet modules. I often accumulate lots of .bundle directories with
the same gems installed across many repos.  This means I often have to run `bundle install`  and `find /repos/ -name
'.bundle' -type d -exec rm -rf {} \;` which can take some time. I am sure you have seen this before? This pain becomes
exponential on airplane wifi.

```
Fetching gem metadata from https://rubygems.org/...........
Fetching gem metadata from https://rubygems.org/..
Resolving dependencies...
Installing rake (10.4.2)
Installing CFPropertyList (2.2.8)
Installing addressable (2.3.8)
Installing backports (3.6.4)
Installing builder (3.2.2)
Installing hitimes (1.2.2)
Installing timers (4.0.1)
Installing celluloid (0.16.0)
Installing coderay (1.1.0)
Installing multi_json (1.11.0)
Installing gherkin (2.12.2)
Installing cucumber-core (1.1.3)
Installing diff-lcs (1.2.5)
Installing multi_test (0.1.2)
.
.
.
Installing typhoeus (0.7.1)
Installing travis (1.7.7)
Installing travis-lint (2.0.0)
```

Its this kind of repetition and wasted space that [really grinds my gears](https://www.youtube.com/watch?v=Q685Ko2DHDs).  Furthermore, I can only test one puppet
version at a time before running into gem conflicts.

So then I thought, Docker has the layered filesystem thingy, and reusable images which could greatly speed up the testing
process and save space. I can even use this workflow in my CI pipelines too.  So why wouldn't I use containers?

Now the whole point is to ditch ruby version managers and bundler and rely purely on the system ruby and pre installed gems.
We have the ability with docker images to mix and match ruby versions with puppet to give us the ultimate testing matrix.

I have mentioned in the past about the importance of unit testing your puppet code [here.](http://logicminds.github.io/blog/2015/04/14/testing-your-infallibleness-part-1/) If you have spent any time unit
testing your code you have probably only tested against a single version of puppet because  it takes too much time to
test against a good version matrix unless your using travis.  However, by using containers we can easily swap out
versions with a single command thereby reducing the feedback loop.

Now, before we start using docker for development, there are a few things you need.  This should work on most platforms,
but some instructions may differ slightly especially around using boot2docker.

## Setup boot2docker
We need boot2docker to run the Docker container platform since Docker does not natively run on OS X.

  1. Install boot2docker  `brew install boot2docker`
  2. boot2docker init
  3. boot2docker up
  4. eval \`boot2docker shellinit\`
  5. docker version # test docker works
  6. `docker run --rm --hostname=puppetdev -t logicminds/centos-puppetdev:latest3.8.x puppet --version`

Now if your curious if docker-machine can be used in place of boot2docker, docker-machine suffers from a [volume mount issue](https://github.com/docker/machine/issues/641).
Furthermore, you need to ensure your puppet modules are under your home directory as detailed [here](https://docs.docker.com/userguide/dockervolumes/#mount-a-host-directory-as-a-data-volume).

**ENSURE YOUR MODULE CODE IS CHECKED OUT UNDER YOUR HOME DIRECTORY**  

## Using puppet development containers
The point of using containers for development is to not only conserve time but also isolate dependency issues.  Additionally,
some gems require development dependencies that you may not want on your system.  So keeping them in a container allows
you to keep your laptop in pristine condition while also having the ability to throw away your development environment with ease.

While the container images are static in nature, the container in which your test runs on will only last for the duration of
the test. The lifecycle of containers is short and only meant to perform the task before exiting (unless its a long
running process like a guard, webservice, or pry).   So consider this an ephemeral development environment.  When we start a container
we are going to mount the current module directory on your machine into the docker container so that you can change your code
on your computer and the container will automatically see these changes. Then we pass in any
command we want the container to run.  So before we start using docker, lets dissect the command first.

So given the following docker command, we can break down the parts below:
`docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -t logicminds/centos-puppetdev:latest3.8.x puppet --version`

1. docker run (simple docker command to start a container given an image)
2. --rm (remove the container upon exit)
3. -v $(PWD):/module  (mount a volume designated by the $PWD environment variable to /module inside the container
4. --workdir (change the working directory inside the container to /module
5. --hostname (give the container a real hostname)
6. -t (start s pseudo-TTY) - makes the colors show
7. logicminds/centos-puppetdev:latest3.8.x  (the docker image to use when starting the container)
8. puppet --version (the command the container will run)

Run Multiple versions of puppet just by switching docker images
```
Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -t logicminds/centos-puppetdev:latest3.8.x puppet --version
3.8.1
Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -t logicminds/centos-puppetdev:latest3.7.x puppet --version
3.7.5
Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -t logicminds/centos-puppetdev:latest4.1.x puppet --version
4.1.0
Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -t logicminds/centos-puppetdev:latest3.2.x puppet --version
3.2.4
```

## Interactive Usage
Now if you want the container to stick around so you can login and play around just pass the `-i`.  
The `-i` makes the container interactive (if the command is /bin/bash you can login into the container and play around)

The interactive option only makes sense with shells like /bin/bash, or when the process your running requires some interaction.

## Sample Workflows

### Basic testing against three versions of puppet
1. cd ~
2. git clone https://github.com/logicminds/gitlab_mirrors.git
3. cd gitlab_mirrors
4. `docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -t logicminds/centos-puppetdev:latest3.8.x rake spec`
5. `docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -t logicminds/centos-puppetdev:latest3.7.x rake spec`
6. `docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -t logicminds/centos-puppetdev:latest4.1.x rake spec`

```
Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.8.x rake spec
Cloning into 'spec/fixtures/modules/stdlib'...
remote: Counting objects: 6896, done.
remote: Compressing objects: 100% (129/129), done.
remote: Total 6896 (delta 83), reused 22 (delta 22), pack-reused 6738
Receiving objects: 100% (6896/6896), 1.44 MiB | 852.00 KiB/s, done.
Resolving deltas: 100% (2987/2987), done.
HEAD is now at da11903 Merge pull request #299 from apenney/432-release
Cloning into 'spec/fixtures/modules/git'...
remote: Counting objects: 29, done.
remote: Compressing objects: 100% (18/18), done.
remote: Total 29 (delta 0), reused 25 (delta 0), pack-reused 0
Unpacking objects: 100% (29/29), done.
/usr/bin/ruby -I/home/puppet/.gem/ruby/gems/rspec-core-3.2.3/lib:/home/puppet/.gem/ruby/gems/rspec-support-3.2.2/lib /home/puppet/.gem/ruby/gems/rspec-core-3.2.3/exe/rspec --pattern spec/\{classes,defines,unit,functions,hosts,integration\}/\*\*/\*_spec.rb --color
..................

Finished in 2.16 seconds (files took 0.64817 seconds to load)
18 examples, 0 failures

Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.7.x rake spec
Cloning into 'spec/fixtures/modules/stdlib'...
remote: Counting objects: 6896, done.
remote: Compressing objects: 100% (129/129), done.
remote: Total 6896 (delta 83), reused 22 (delta 22), pack-reused 6738
Receiving objects: 100% (6896/6896), 1.44 MiB | 853.00 KiB/s, done.
Resolving deltas: 100% (2987/2987), done.
HEAD is now at da11903 Merge pull request #299 from apenney/432-release
Cloning into 'spec/fixtures/modules/git'...
remote: Counting objects: 29, done.
remote: Compressing objects: 100% (18/18), done.
remote: Total 29 (delta 0), reused 25 (delta 0), pack-reused 0
Unpacking objects: 100% (29/29), done.
/usr/bin/ruby -I/home/puppet/.gem/ruby/gems/rspec-core-3.2.3/lib:/home/puppet/.gem/ruby/gems/rspec-support-3.2.2/lib /home/puppet/.gem/ruby/gems/rspec-core-3.2.3/exe/rspec --pattern spec/\{classes,defines,unit,functions,hosts,integration\}/\*\*/\*_spec.rb --color
..................

Finished in 2.19 seconds (files took 0.64453 seconds to load)
18 examples, 0 failures

Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest4.1.x rake spec
Cloning into 'spec/fixtures/modules/stdlib'...
remote: Counting objects: 6896, done.
remote: Compressing objects: 100% (129/129), done.
remote: Total 6896 (delta 83), reused 22 (delta 22), pack-reused 6738
Receiving objects: 100% (6896/6896), 1.44 MiB | 845.00 KiB/s, done.
Resolving deltas: 100% (2987/2987), done.
HEAD is now at da11903 Merge pull request #299 from apenney/432-release
Cloning into 'spec/fixtures/modules/git'...
remote: Counting objects: 29, done.
remote: Compressing objects: 100% (18/18), done.
remote: Total 29 (delta 0), reused 25 (delta 0), pack-reused 0
Unpacking objects: 100% (29/29), done.
/usr/bin/ruby -I/home/puppet/.gem/ruby/gems/rspec-core-3.2.3/lib:/home/puppet/.gem/ruby/gems/rspec-support-3.2.2/lib /home/puppet/.gem/ruby/gems/rspec-core-3.2.3/exe/rspec --pattern spec/\{classes,defines,unit,functions,hosts,integration\}/\*\*/\*_spec.rb --color
..................

Finished in 2.87 seconds (files took 1 second to load)
18 examples, 0 failures
```
### Using pry with containers
1. cd ~
2. git clone https://github.com/logicminds/gitlab_mirrors.git
3. cd gitlab_mirrors
4. echo "require 'pry'\nbinding.pry" >> spec/spec_helper.rb   (example of pry interaction only)
5. `docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.8.x rake spec`

```
Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.2.x rake spec       
Cloning into 'spec/fixtures/modules/stdlib'...
remote: Counting objects: 6896, done.
remote: Compressing objects: 100% (129/129), done.
remote: Total 6896 (delta 83), reused 22 (delta 22), pack-reused 6738
Receiving objects: 100% (6896/6896), 1.44 MiB | 406.00 KiB/s, done.
Resolving deltas: 100% (2987/2987), done.
HEAD is now at da11903 Merge pull request #299 from apenney/432-release
Cloning into 'spec/fixtures/modules/git'...
remote: Counting objects: 29, done.
remote: Compressing objects: 100% (18/18), done.
remote: Total 29 (delta 0), reused 25 (delta 0), pack-reused 0
Unpacking objects: 100% (29/29), done.
/usr/bin/ruby -I/home/puppet/.gem/ruby/gems/rspec-core-3.2.3/lib:/home/puppet/.gem/ruby/gems/rspec-support-3.2.2/lib /home/puppet/.gem/ruby/gems/rspec-core-3.2.3/exe/rspec --pattern spec/\{classes,defines,unit,functions,hosts,integration\}/\*\*/\*_spec.rb --color

From: /module/spec/spec_helper.rb @ line 3 :

    1: require 'puppetlabs_spec_helper/module_spec_helper'
    2: require 'pry'
 => 3: binding.pry
    4: puts "hello"

[1] pry(main)> exit
 hello
 ..................

 Finished in 2.14 seconds (files took 45.41 seconds to load)
 18 examples, 0 failures

```
### Login to the container and look around
1. `docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.8.x /bin/bash`
2. which puppet
3. puppet --version
4. exit

```
Coreys-MacBook-Pro% docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.8.x /bin/bash
[puppet@puppetdev module]$ which puppet
~/bin/puppet
[puppet@puppetdev module]$ which ruby
/usr/bin/ruby
[puppet@puppetdev module]$ ruby --version
ruby 2.0.0p598 (2014-11-13) [x86_64-linux]
[puppet@puppetdev module]$ puppet --version
3.8.1
[puppet@puppetdev module]$ facter virtual
docker
```
### Using bundler bundles to override the default gemset in the container
Now the containers I built are somewhat opinionated by this [Gemfile](https://github.com/logicminds/docker-centos-puppetdev/blob/master/scripts/puppet-bootstrap/Gemfile). So if you
need something else without building a new container image you can use bundler bundles.  Although this workflow now
becomes a two step process and bundle install is now being used which slows down your workflow.  This uses the container
to download and compile all the gems and places them inside a bundle folder for later use which is persistent on your
machine.  So each successive docker run will be able to use the prebuilt gems, provided you use `bundle exec rake spec`
Additionally, bundler seems to spend a few seconds doing background tasks for each use which can be annoying.

1. cd ~
2. git clone https://github.com/logicminds/gitlab_mirrors.git
3. cd gitlab_mirrors
4. rm Gemfile.lock
5. docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.8.x bundle install --standalone
6. docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.8.x bundle exec rake spec



## Puppetdev Container Repo
These images are available on the [docker hub](https://registry.hub.docker.com/u/logicminds/centos-puppetdev/) so its as easy as `docker pull logicminds/centos-puppetdev:latest3.8.x`

All images and various tags run on centos.  Depending on the ruby version required by puppet this usually dictates which version
of centos is used.  This is due to centos7 containing ruby 2.0 by default and centos6 using ruby 1.9.3.  There is no ruby
version manager in the mix since we rely on the system ruby.


Currently Available tags:

 * latest4.1.x
 * latest3.2.x  (not really compatible with ruby 2.0, but thats only during agent runtime which isn't used during testing)
 * latest3.7.x
 * latest3.8.x

These are easy to make so others versions will be available upon request


## Going a step further
If your tired of typing these commands you can create an alias for each puppet version.

```
# Puppet environment aliases
alias puppet38='docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.8.x'
alias puppet37='docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest3.7.x'
alias puppet41='docker run --rm -v $(PWD):/module --workdir /module --hostname=puppetdev -ti logicminds/centos-puppetdev:latest4.1.x'

```

Then to run a test against a container just use an alias `puppet38 rake spec`.  


As you can see using containers is a great way to conserve time and space on your machine.  Not having to wait 60s for a
bundle install to run can save you time.  There are probably a few more workflows out there as well.  Imagine running
guard in the background of a container that checks your code across multiple versions of puppet!  Its like having a mini CI on
your laptop.  So give containers a try for your development environment.


## References
http://www.hokstad.com/docker/patterns
https://medium.com/@treeder/why-and-how-to-use-docker-for-development-a156c1de3b24
