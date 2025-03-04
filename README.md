# openwrt-daemon-button: A Small Primer on Common Service Management Issues #


# Introduction #

This project is a primer on common service management issues.  With
blinking lights!

But at the same time, this is a bad project.  You should not use it,
stay away, it is dangerous.

So why did I publish it?

To document the issues that are common in daemon and service
management.  Specifically how providing more security options and
visibility to users can lead to a false sense of security and actually
decrease the security of a system.  clients.

This project is a set of OpenWRT scripts to add support for enabling
and disabling daemons with device buttons.  It also interacts with the
device LEDs showing when the services are enabled or disabled.

It currently works with the WPS button and the associated LED on some
wireless routers.  I've tested it on Netgear EX6150v2 devices
specifically.  It has been tested on the OpenWRT 23.xx release series.

First, some warnings upfront about the current state:

**WARNING**: The current version doesn't do periodic checks of service
  state.  This is a potential security issue.

**WARNING**: With opkg upgrades, it is VERY likely your custom
  /etc/rc.button/wps file and your /etc/rc.local file WILL get
  over-written, unless you are very careful about configuraton
  management and the upgrade process.  This can lead to a state where
  the service may be running or stopped and the LED may indicate the
  opposite.  Do not use in production environments.
  

# Motivation #

I have a fairly portable OpenWRT device I use for wireless roaming,
sharing wireless and as an extra firewall layer.  It also doubles as a
learning platform for embedded device configuration and fleet
management.

Currently, it's locked down with standard practices seen on commercial
devices.  A management VLAN locked to physical ethernet ports, and a
bunch of other things that would be nice in a base OpenWRT install.
But I don't usually need a SSH daemon running, and the WPS button
doesn't work with the base OpenWRT install.  So why not use that
button to enable/disable SSH access to the router?

But walking through the process of creating these scripts, I kept
hitting security issues that went beyond simple bugs.  When users,
developers, or devops people install security-oriented features, they
expect them to work.  Not only do they expect them to work, the act of
installing the software increases the user's expectation that their
system is more secure.  With the patterns this project exposes, that
mistaken belief in security can be exploited.

Managing system services is complicated.  Flapping services,
unintentional DDOS due to uneven load, etc.  These issues are talked
about frequently in the operations and devops community.  This
document looks at a small subset of the issues around using
information design and user interfaces to reveal security information
and where that can go wrong.


# Development Log #

Below is a development log of what, where and how the scripts and
system were implemented.  It is not complete, but it lists some of
what has been done.


## Initial rc.button script ##

First, I needed basic functionality to enable and disable SSH by
pressing a button.

There is a file in the default OpenWRT install called wps in the
/etc/rc.button directory.  This script is called whenever the WPS
button on a router is pushed.  With OpenWRT 23.05.0, WPS isn't enabled
on OpenWRT by default.

So let's use it to disable/enable SSH!  Specifically the dropbear service.

The script wps in the openwrt-daemon-button repo is a short example of
how this can be done.

But after implementing it, a can of worms opened up around
synchronizing service state and LED state.

What happens on system start?  What does the LED indicate?  This
script isn't enough to know if SSH is started on boot.

What happens if the user enables/disable the service via the
/etc/init.d/SERVICE_NAME script?  The LED won't reflect that.

What happens if the service dies without getting restarted?  The LED
won't reflect that.

What happens if the script is just buggy or there is a change in the
LED or button API?  The LED won't reflect that.

These are just a few of the issues that can give a false sense of
security where none was expected before.


## rc.local script ##

To help mitigate the sync on system start, an rc.local script was
created that checks the state of the service after 20 seconds.

There are two major issues with this script.  First, the existing
rc.local has an "exit 0" at the end.  If this isn't removed, the
custom script never gets called.

The second issue is that we created an arbitrary wait period for
dropbear.  20 seconds should work most of the time, but a better
option would be a more integrated system and service manager that
provides more dependency management hooks and status tools.

Finally, instead of relying on the user to manually append the
rc.local file, we should scan the existing rc.local file and insert
our code if it hasn't been inserted already.  This is made difficult
by the text editing tools available on a busybox-oriented system.


## Customizing the service to update the LED ##

TODO

Next, the dropbear service itself must be customized to change the LED
state if it's state is changed.


## periodic checker script ##

TODO

Even with changing the LED state on startup and integration with every
possible way to start/stop the service, it's still not enough.
Services sometimes die, they sometimes restart randomly hours later.
We need to periodically check the state of the monitored service and
change the LED when it changes.

This of course introduces another layer of complexity because the
periodic monitor script is prone to failures itself.


## Indicating Exceptions with Blinking Lights ##

TODO 

What happens when there isn't a problem with the monitored service,
but the monitoring scripts themselves?  APIs change and bugs happen.
We can utilize a third visual UX pattern and make the WPS light blink
when the scripts themselves fail.

This should be combined with logging to external logging systems where
the logs can themselves be monitored and processed.


## opkg packaging ##

TODO

Finally, opkg, the package management format and tool used in OpenWRT,
provides a way to package this script and make sure that changes don't
get over-written by other default packages.


# Installation #

The wps script replaces the wps script in your /etc/rc.button
directory on your OpenWRT device.

Backup your copy of the wps script to somewhere safe.  Then copy the
wps file to /etc/rc.button

The rc.local script should be appended to your /etc/r.local

First make sure the exit 0 is removed from your existing rc.local

**WARNING** Not removing the exit 0 from your existing rc.local will result in
LED state being out-of-sync with service state.  This is a security risk.
