---
layout: post
title: "Remotely Controlling Your Server With IPMI"
date: 2015-04-10 12:51:16 -0700
comments: true
published: true
categories:  [sysadmin, remote, ipmi, bmc]
---

In 2010 my life changed dramatically as I moved from Atlanta, GA to Portland, OR.  During this time I maintained a 100%
remote position where I controlled twenty unique datacenters all over the country with my computer and a VPN connection.
Due to to the nature of the company, problems only seemed to present themselves during the early hours (After 12AM).  On
several occasions I found the need to turn off a production system due to power or cooling issues.  Traditionally, this
was easily accomplished by picking up the phone and instructing a person to perform a task.  However, finding an able
body in the middle of the night sometimes extended downtime past a few hours.  And god forbid they might turn off the
wrong system.  This led to unnecessary downtime in my opinion that could have been prevented by using out-of-band
management.

What is out-of-band management?  Essentially, out-of-band management is a device that allows you to perform
actions on a server as if you were standing in front of it.  This type of management works independently of the OS, so
if the OS crashes you still have a way in.  The standard protocol behind out-of-band management is called IPMI.
However, many device manufactures had similar solutions before IPMI was even available.  Solutions like HP’s iLO, IBM’s
OSA, Dell’s DRAC, and Sun’s iLom.  When IPMI came along, server manufactures shoe-horned IPMI support into their
proprietary out-of-band management devices.  The end result is a buggy IPMI device that “mostly” works.  However, what
IPMI provides is standard mechanism to control any kind of server despite its origin.  Fortunately, newer IPMI devices
are more compliant with the IPMI protocol than their predecessors.

## Setup

The first thing you need to know about IPMI is that it is not configured out the box.  In many situations you will need
to configure the IPMI device from the BIOS or upon bootup.  If your lucky enough to be using a server that sets up IPMI
automatically via DHCP then your in luck.  For example, HP ilo devices will use the dhcp assigned ip address when plugged
in.  Some devices like HP ilo come with a unique username/password to initially login to the device, while others have
some sort of insecure default credentials hidden in the manual. Essentially what you need to do is login (via ssh or
http) to the device and configure to your requirements.  In a later article I will detail how you can automate the
deployment of IPMI devices so you never have to perform this step manually again.



## Locating that special port

As you can see in the following picture the ilo device is the special out-of-band management device that allows us to
use IPMI commands to control the server remotely.  It looks just like a normal ethernet port but is usually marked
specifically for the BMC controller (ilo, ipmi, drac, ...). So just plug a ethernet cable in and find the IP address using
nmap, dhcp logs, or other mechanism.


 ![Server with IPMI device](../images/Hp_proliant_dl380_g5_5-300x135.jpg)


## Remotely controlling that server
Usually each IPMI device has a web interface for configuring and controlling the device.  So initially, you will have to
use this interface to set it up.  In the picture below I have logged into the device and clicked on power management and
then clicked momentary press to turn the server on.  Disclaimer: I am not responsible for you powering on/off servers
in any of your environments.  Make sure you know what your doing.

![ilo page](../images/ilo.png)

Once the IPMI device turns the server on the options change and we can now see the server is powered on.

![ilo page](../images/ilo_on.png)

Some of you may be asking yourself, is there a CLI tool to perform these functions?  The answer is yes, however
the CLI commands are generic in nature and some functionality may not be available in the opensource CLI tools.  But,
the majority of what you want to do should be supported.

These tools are [freeipmi](http://www.gnu.org/software/freeipmi/) and
[ipmitool](http://sourceforge.net/projects/ipmitool) both of which perform the same function but differ in implementation.
Additionally, there is [openipmi](http://openipmi.sourceforge.net) which works as a driver that freeipmi and ipmitool
use for communication with the IPMI device.  But openipmi is not necessary if you know the username, password and ip
address of the IPMI device.

With one of these tools installed you can skip the web UI and remotely control the system from any unix based system running
either ipmitool or freeipmi.

```
ipmitool -H 192.168.1.21 -U admin -P password -I lanplus power on
Chassis Power Control: Up/On

```

## Futher reading
Now if your wondering just how to configure your IPMI device using configuration management have a look at the
[bmclib](https://github.com/logicminds/bmclib) puppet module for configuring each IPMI device.  Bmclib internally uses
ipmitool and openipmi so that you don't have to.

Additionally, if you are looking for a way to interact with the IPMI device pragmatically you can use the
[Rubyipmi](https://github.com/logicminds/rubyipmi) library directly without having to know any knowledge of ipmitool or
freeipmi.

There is a ton of imformation you can get out of the IPMI device like sensor information and event logs so its worth exploring
with Rubyipmi if you plan on doing sensor monitoring of some sort.

A great example of monitoring with Rubyipmi and [Sensu](https://github.com/sensu/sensu-community-plugins/blob/master/plugins/ipmi/check-sensor.rb).

## Summary
Now that you know how to remotely control your servers you can spend more time doing other things since you won't be
on the phone instructing other people to turn the server off.  Furthermore, stand up and move your legs since you just
removed the need to walk to the server room and reduced your daily pedometer score.


