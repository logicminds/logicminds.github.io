---
layout: post
title: "Getting real with the RAL"
date: 2020-10-10T11:15:48-07:00
comments: true
published: true
categories: [puppet devops facts ral]
---

A client of mine recently had a need to discover that state of a particular package from inside a fact.  Ironically, someone else in the puppet-community had a similar need and was asking questions about this.  So I thought it 
would a good idea to blog about how to query the system agnostically without shelling out in a fact. 

The original solution for many of us was to create a fact like the following:

```ruby
# lib/facter/is_ntp_installed.rb
Facter.add(:is_ntp_installed) do
  confine :osfamily => 'RedHat'
  setcode do
     # if the returned value is empty, ntp is not installed
     ! Facter::Core::Execution.execute('rpm -qa ntp').empty?
  end
end

```

However, this leads us to a fact that only works on systems with rpm.   If this is all you want then no need to implement anything else.  Just remember to confine the fact to only RedHat based systems.  

But if this module is expected to work on many operating systems you need an abstraction layer on top to find if ntp is present.  Otherwise you end up writing a huge case statement with every similar fact that queries the system.

Enter the puppet RAL (Resource abstraction layer).  The RAL determines how to discover the ntp package using the type and provider mechanisms built into puppet.   We utilize puppet to create a fact to use in puppet code.  Genius, right?


```ruby
# lib/facter/is_ntp_installed.rb
require 'puppet'
Facter.add(:is_ntp_installed) do
	setcode do
		package_resource = Puppet::Resource.indirection.find('package/ntp')
		# Output of using the RAL
	  # => Package[ntp]{:name=>"ntp", :ensure=>:purged, :provider=>:yum, :audit=>[:ensure, :package_settings], 
	  # :configfiles=>:keep, :allow_virtual=>true, :reinstall_on_refresh=>:false, :loglevel=>:notice}
	  package_resource[:ensure] != :purged || package_resource[:absent]
	end
end
```

This fact will call out to puppet and ask it what packages it has that have ntp as the name.  What you get back is a package resource with a few attributes that can be further filtered.  You might have noticed that we never shelled out or specified rpm/deb/yum or any package provide to determine if ntp is installed. 

So next time you need to test for a package try the Puppet RAL instead to make your code OS agnostic. 


