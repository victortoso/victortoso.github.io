---
title: On usbredir after SPICE
date: 2024-07-01
tags:
    - SPICE
    - usbredir
categories:
    - coding
keywords:
    - SPICE
    - usbredir
---

# SPICE

[SPICE][] is an open source set of protocols plus the implementation of those protocols which
provides a Virtual Desktop Infrastructure (VDI) solution. In short, a way to connect remotely to a
machine, virtualized or not, and use it as it was your local machine.

This project, while open source, got defunded at Red Hat, [deprecated][] in RHEL 8 and _mostly_
removed in RHEL 9 as we have to keep supporting some components due compatibility concerns.

Now, with SPICE moving away in RHEL-based distros, solutions like VNC or RDP might be good
enough replacements for some. For others, they will miss some of SPICE native features.

Remote USB redirection is one of this features that I see mentioned and requested once in a while.
There are a few ways to do this, but this post is mostly concerning _how_ you can do it in a
_similar_ fashion of how SPICE did it. Remotely on RHEL based distros. 

# usbredir

The [usbredir][] library was born to connect USB devices to QEMU over TCP/IP. It implements a
protocol that wraps the actual USB communication. SPICE does wrap usbredir protocol and deliver it
to QEMU.

This also means that, on QEMU, we have a driver that talks usbredir protocol so, with the right
configuration and a usbredir client, you can connect your local USB device to a remote QEMU and make
the device appear in the Guest OS, no SPICE nor extra guest drivers needed.

## usbredir client

We can use `usbredirect` binary tool to redirect a local USB device to a remote usbredir endpoint.

The tool is able to act as either a TCP Client or a TCP Server in which it would wait, listening to
incoming TCP connections to start USB redirection. This depends on the host configuration, you'll
see example of both shortly.

# Configuring Libvirt

The configuration is not complicated and the [documentation][] should be able to clarify everything.

Going over the options quickly, you should create the `redirdev` element using `type='tcp'` and
`bus='usb'`. Now, you have two options, you either want QEMU to connect to your usbredir client, the
`connect` mode, or you want QEMU to be configured in such a way you can connect the usbredir client,
which is the `bind` mode.

## Bind mode

The configuration takes place in the source element, using mode=`bind`, QEMU will try to bind 
at `host` argument in the specified port in `service` argument. The libvirt snip:

{{< highlight xml "linenos=table, hl_lines=3, linenostart=100" >}}
  <devices>
    <redirdev bus='usb' type='tcp'>
      <source mode='bind' host='localhost' service='5550'/>
      <protocol type='raw'/>
      <address type='usb' bus='0' port='3'/>
    </redirdev>
  </devices>
{{< /highlight >}}

Using `usbredirect` to connect device `04f2:b596`

{{< highlight bash "linenos=false" >}}
# Using vendor:product usb information
$ usbredirect --device 04f2:b596 --to localhost:5550

# Using bus-device_number also works 
$ usbredirect --device 1-4 --to localhost:5550
{{< /highlight >}}


## Connect mode

Using mode=`connect`, QEMU will try to connect at `host` argument in the specified port in `service`
argument. The libvirt snip:


And then launching a VM to act as 'TCP client', the libvirt
configuration used is:

{{< highlight xml "linenos=table, hl_lines=3, linenostart=100" >}}
  <devices>
    <redirdev bus='usb' type='tcp'>
      <source mode='connect' host='localhost' service='5550'/>
      <protocol type='raw'/>
      <address type='usb' bus='0' port='3'/>
    </redirdev>
  </devices>
{{< /highlight >}}

{{< highlight bash "linenos=false" >}}
$ usbredirect --device 04f2:b596 --as 192.168.122.1:5550
{{< /highlight >}}


Note that this happens before VM boot, so the VM will wait till it can connect to the USB device.

# Limitations

When using usbredir over SPICE, we take for granted so many features that were part of SPICE, such
as:
 1. Secure communication channel (TLS)
 2. Migration support
 3. Enhanced performance [^1]

All of the above can be worked on and included in usbredir and QEMU. Those issues are not necessarly
hard but not trivial either to fix. Considering that this work is also in the VDI field, we would
need help from the community to dedicate time to it.

Another limitation is that usbredir is lagging behind USB implementations. This is import to support
newer USB devices with the right configuration and speed.

[SPICE]: https://www.spice-space.org/
[deprecated]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/8.4_release_notes/deprecated_functionality#deprecated-functionality_virtualization
[usbredir]: https://gitlab.freedesktop.org/spice/usbredir/
[documentation]: https://libvirt.org/formatdomain.html#redirected-devices


[^1]: In both QEMU device driver and usbredir client applications there are room to improve, e.g:
    [issue!28](https://gitlab.freedesktop.org/spice/usbredir/-/issues/28)
