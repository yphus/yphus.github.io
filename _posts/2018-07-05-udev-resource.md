---
layout: post
title:  "udev_resource secret options"
date:   2018-07-05 11:02:20 +0200
categories: test ubuntu
tags: udev python3
---

The `plainbox-provider-resource` [udev_resource][bin] script contains some 
"hidden" options that I'd like to highlight in this post as quite often I'm 
using them for debugging or even to develop new jobs.

Fake the udev resource job to validate template jobs
----------------------------------------------------

I don't have any CANbus controllers on my laptop so it's tricky to check the 
[template jobs][templates] that we have in the `plainbox-provider-checkbox` 
other than running checkbox on devices in the lab which have such controllers.

The trick here is to fake the parser ouput by supplying the `udevadm info -e` 
output from a different system.

Edit the `device` resource job definition this way:

{% highlight console %}
id: device
estimated_duration: 0.48
plugin: resource
command: udev_resource -c 'cat /home/sylvain/lp/checkbox-support/checkbox_support/parsers/tests/udevadm_data/CARA_T_SOCKETCAN.txt'
_description: Creates resource info from udev
_summary: Collect information about hardware devices (udev)
{% endhighlight %}

Run `checkbox-cli` in a terminal and select the `socketcan-auto-remote` test 
plan. You now check that the template jobs are working as expected (notice 
the _can0 job suffix): 

{% highlight console %}
 Choose tests to run on your system:
┌────────────────────────────────────────────────────┐
│[X] - Test SocketCAN interfaces                     │
│[X]    socketcan/send_packet_remote_eff_can0        │
│[X]    socketcan/send_packet_remote_fd_can0         │
│[X]    socketcan/send_packet_remote_sff_can0        │
│[X] + Uncategorised                                 │
│                                                    │
│                                                    │
│                                                    │
│                                                    │
└────────────────────────────────────────────────────┘
 Press (T) to start Testing                  (H) Help
{% endhighlight %}

Filter the parsing output by categories
---------------------------------------

Having to grep (with context) the full output of the udev parser just to find a 
product name or a udev property can quickly become painful. `udev_resource` 
provides the `-f` option to filter the device by category. This way it's super 
easy to identify the vendor ids of my webcams:

{% highlight console %}
sylvain@ThinkPad-T480s:~/lp/plainbox-provider-resource/bin$ ./udev_resource -f CAPTURE
path: /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4.2/1-4.2:1.0
bus: usb
category: CAPTURE
driver: uvcvideo
product_id: 2093
vendor_id: 1133
product: HD Pro Webcam C920
vendor: Logitech, Inc.
product_slug: HD_Pro_Webcam_C920
vendor_slug: Logitech__Inc.

path: /devices/pci0000:00/0000:00:14.0/usb1/1-5/1-5:1.0/input/input11
bus: usb
category: CAPTURE
driver: uvcvideo
product_id: 46613
vendor_id: 1266
product: Integrated IR Camera: Integrate
product_slug: Integrated_IR_Camera__Integrate

path: /devices/pci0000:00/0000:00:14.0/usb1/1-8/1-8:1.0/input/input13
bus: usb
category: CAPTURE
driver: uvcvideo
product_id: 46612
vendor_id: 1266
product: Integrated Camera: Integrated C
product_slug: Integrated_Camera__Integrated_C
{% endhighlight %}

It's even possible to pass multiple categories:

{% highlight console %}
$ ./udev_resource -f NETWORK WIRELESS
path: /devices/pci0000:00/0000:00:1c.6/0000:3d:00.0
bus: pci
category: WIRELESS
driver: iwlwifi
product_id: 9469
vendor_id: 32902
subproduct_id: 16
subvendor_id: 32902
product: Wireless 8265 / 8275 (Dual Band Wireless-AC 8265)
vendor: Intel Corporation
interface: wlp61s0
mac: 68:ec:c5:6c:c1:36
product_slug: Wireless_8265___8275__Dual_Band_Wireless-AC_8265_
vendor_slug: Intel_Corporation

path: /devices/pci0000:00/0000:00:1f.6
bus: pci
category: NETWORK
driver: e1000e
product_id: 5591
vendor_id: 32902
subproduct_id: 8792
subvendor_id: 6058
product: Ethernet Connection (4) I219-LM
vendor: Intel Corporation
interface: enp0s31f6
mac: 8c:16:45:2b:17:8c
product_slug: Ethernet_Connection__4__I219-LM
vendor_slug: Intel_Corporation
{% endhighlight %}

Count devices by categories and pretty-print the result
-------------------------------------------------------

Originally developed to create the [device check][device_check] job, the `-l` 
option pretty prints the devices found by the parser and count them by 
categories:

{% highlight console %}
$ ./udev_resource -l CAPTURE
CAPTURE (3):
 - Logitech, Inc. HD Pro Webcam C920
 - None Integrated IR Camera: Integrate
 - None Integrated Camera: Integrated C
{% endhighlight %}

With `-l` "detect" jobs can get a pretty and useful output:

{% highlight console %}
plugin: shell
category_id: com.canonical.plainbox::optical
id: optical/detect
requires: device.category == 'CDROM'
estimated_duration: 1.2
_summary: Displays discovered optical drives
_description: Detects optical drives (CD/DVD) attached to the system.
command: udev_resource -l CDROM
{% endhighlight %}

[src]:          https://git.launchpad.net/checkbox-support/tree/checkbox_support/parsers/udevadm.py
[bin]:          https://git.launchpad.net/plainbox-provider-resource/tree/bin/udev_resource
[templates]:    https://git.launchpad.net/plainbox-provider-checkbox/tree/units/socketcan/jobs.pxu#n81
[device_check]: https://git.launchpad.net/plainbox-provider-checkbox/tree/units/miscellanea/jobs.pxu#n356
