---
layout: post
title:  "Test GATT services notifications using Python/D-Bus"
date:   2018-07-04 11:02:20 +0200
categories: test iot ubuntu
tags: dbus python3 gatt bluetooth bluez
---

As part of the [Ubuntu IoT certification program][certification], we developed 
tests checking the Bluetooth Low Energy capabilities. Let's see how we can 
write a python3 script to develop a [GATT][gatt] notification test using the 
[BlueZ D-Bus API][bluez-git-doc] (which is the recommended way[^1] to write 
applications talking to Bluetooth devices). We'll use a [Nordic52 Thingy IoT 
Sensor kit][Nordic-Thingy-52] to send the raw measurement notifications.

[GATT][gatt] is an acronym for the *Generic Attribute Profile*, and it defines 
the way that two Bluetooth Low Energy devices send and receive data using a 
collection of *Services* and *Characteristics*.

Prerequisites
-------------

[python3-dbus][python3-dbus] and [python3-gi][python3-gi] either 
installed from repositories or available as stage packages from your snap (by 
default on all checkbox snaps via the [plainbox-provider-checkbox remote 
part][p-p-c-parts]).
BlueZ comes preinstalled on classic, just install the 
[snap](https://snapcraft.io/bluez) on Ubuntu Core:

{% highlight console %}
$ snap install bluez
$ snap connect checkbox-XXX:bluez bluez:service
{% endhighlight %}

The python3 script skeleton, with all the needed [D-Bus][dbus] interfaces:

{% highlight python %}
#!/usr/bin/env python3

import argparse
import logging
import os
import sys
import time

import dbus
import dbus.service
import dbus.mainloop.glib
from gi.repository import GObject

logger = logging.getLogger(__file__)
logger.addHandler(logging.StreamHandler(sys.stdout))

ADAPTER_INTERFACE = 'org.bluez.Adapter1'
DEVICE_INTERFACE = 'org.bluez.Device1'
PROP_INTERFACE = 'org.freedesktop.DBus.Properties'
OM_INTERFACE = 'org.freedesktop.DBus.ObjectManager'
GATT_SERVICE_INTERFACE = 'org.bluez.GattService1'
GATT_CHRC_INTERFACE = 'org.bluez.GattCharacteristic1'

dbus.mainloop.glib.DBusGMainLoop(set_as_default=True)
{% endhighlight %}

Find the correct Adapter
------------------------

On desktop systems, I'd recommend [D-Feet](https://wiki.gnome.org/Apps/DFeet) 
to get a browsable D-Bus tree but on IoT running Ubuntu Core, our best option 
is the `bluez.bluetoothctl` command line tool from the bluez snap, it gives 
the MAC address of all adapters:

{% highlight console %}
$ sudo bluez.bluetoothctl 
[bluetooth]# list
Controller EE:B1:E5:DB:C7:A4 C663432546-00515 #2 [default]
Controller E0:4F:43:44:55:C3 C663432546-00515 
{% endhighlight %}

In order to get the HCI ids, simply run `rfkill`:

{% highlight console %}
$ rfkill list bluetooth
1: hci0: Bluetooth
	Soft blocked: no
	Hard blocked: no
2: hci1: Bluetooth
	Soft blocked: no
	Hard blocked: no
{% endhighlight %}

Depending on the IoT device, several controllers (or adapters) can be listed. The
script does need to select one for pairing with the [Nordic52 Thingy IoT 
Sensor kit][Nordic-Thingy-52].

> _Note_: We cannot reuse [bt_helper.py][bt_helper] from `checkbox-support` as 
its `BtManager` class does not treat adapters individually.

The first class will then be a wrapper around the dbus Adapter objects, set 
up with a "pattern" parameter that  will help identifying the correct controller 
(it can be hci number or the address, both will work)

{% highlight python %}
class BtAdapter:
    """Bluetooth LE Adapter class."""
    def __init__(self, pattern):
        self._pattern = os.path.basename(pattern)
        self._bus = dbus.SystemBus()
        self._manager = dbus.Interface(
            self._bus.get_object("org.bluez", "/"), OM_INTERFACE)
        self._main_loop = GObject.MainLoop()
        self._adapter = self._find_adapter()
        self._path = self._adapter.object_path
        self._props = dbus.Interface(self._adapter, PROP_INTERFACE)
        self._name = self._props.Get(ADAPTER_INTERFACE, "Name")
        self._addr = self._props.Get(ADAPTER_INTERFACE, "Address")
        self._alias = self._props.Get(ADAPTER_INTERFACE, "Alias")
        logger.info('Adapter found: [ {} ] {} - {}'.format(
            self._path, self._addr, self._alias))

    def _get_managed_objects(self):
        return self._manager.GetManagedObjects()

    def _find_adapter(self):
        for path, ifaces in self._get_managed_objects().items():
            adapter = ifaces.get(ADAPTER_INTERFACE)
            if adapter is None:
                continue
            if (self._pattern == adapter["Address"] or
                    path.endswith(self._pattern)):
                obj = self._bus.get_object("org.bluez", path)
                return dbus.Interface(obj, ADAPTER_INTERFACE)
        raise SystemExit("Bluetooth adapter not found!")
{% endhighlight %}

Find the Nordic Thingy
----------------------

Once the correct adapter is selected, the first thing to do is to power it on. 
It can be done by setting the following property (See the [adapter 
API][adapter-api]):

{% highlight python %}
    def ensure_powered(self):
        """Turn the adapter on."""
        self._props.Set(ADAPTER_INTERFACE, "Powered", dbus.Boolean(1))
        logger.info('Adapter powered on')
{% endhighlight %}

Next API call will set the adapter in Discovery (or Scan) mode. right after 
calling the `StartDiscovery`, the script enters in the dbus mainloop waiting 
for the timeout event.

{% highlight python %}
    def scan(self, timeout=10):
        """Scan for BT devices."""
        dbus.Interface(self._adapter, ADAPTER_INTERFACE).StartDiscovery()
        logger.info('Adapter scan on ({}s)'.format(timeout))
        GObject.timeout_add_seconds(timeout, self._scan_timeout)
        self._main_loop.run()

    def _scan_timeout(self):
        dbus.Interface(self._adapter, ADAPTER_INTERFACE).StopDiscovery()
        logger.info('Adapter scan completed')
        self._main_loop.quit()
{% endhighlight %}

New devices will then populate the dbus tree and another call to the object 
manager will list them all. Their dbus paths will start with the adapter path. 
To select the right device, not only we have to check for the adapter path 
prefix but also find in the `UUIDs` property the "main" service identifier. Let 
me explain what "main" means here. Only few (maybe just one actually) service 
UUID are broadcasted. For the [Nordic52 Thingy IoT Sensor 
kit][Nordic-Thingy-52], the only one advertised is the `Thingy configuration 
service`. To get access to all GATT services UUIDs, the device has to be 
connected to the adapter. That's why the following code snippet will just look 
up for the main service UUID:

{% highlight python %}
    def find_device_with_service(self, ADV_SVC_UUID):
        """Find a device with a given remote service."""
        for path, ifaces in self._get_managed_objects().items():
            device = ifaces.get(DEVICE_INTERFACE)
            if device is None:
                continue
            logger.debug("{} {} {}".format(
                path, device["Address"], device["Alias"]))
            if ADV_SVC_UUID in device["UUIDs"] and path.startswith(self._path):
                obj = self._bus.get_object("org.bluez", path)
                logger.info('Device found: [ {} ] {} - {}'.format(
                    path, device["Name"], device["Address"]))
                return dbus.Interface(obj, DEVICE_INTERFACE)
        raise SystemExit("Bluetooth device not found!")
{% endhighlight %}

GATT service discovery
----------------------

To find the GATT service we're interested in (e.g the [Motion 
service][motion-service]), the selected adapter has to connect the Thingy:

{% highlight python %}
device = adapter.find_device_with_service(args.ADV_SVC_UUID)
device.Connect()
{% endhighlight %}

Similarly to the `BtAdapter` class, the `BtGATTRemoteService` will find the 
Motion service in the dbus tree, e.g: 
`/org/bluez/hci4/dev_DA_9A_D3_B0_0F_DB/service002f`.

{% highlight python %}
class BtGATTRemoteService:
    """Bluetooth LE GATT Remote Service class."""
    def __init__(self, SVC_UUID, adapter, device, max_notif):
        self.SVC_UUID = SVC_UUID
        self._adapter = adapter
        self._device = device
        self._max_notif = max_notif
        self._notifications = 0
        self._bus = dbus.SystemBus()
        self._manager = dbus.Interface(
            self._bus.get_object("org.bluez", "/"), OM_INTERFACE)
        self._main_loop = GObject.MainLoop()
        self._service = self._find_service()
        self._path = self._service.object_path

    def _get_managed_objects(self):
        return self._manager.GetManagedObjects()

    def _find_service(self):
        for path, ifaces in self._get_managed_objects().items():
            if GATT_SERVICE_INTERFACE not in ifaces.keys():
                continue
            service = self._bus.get_object('org.bluez', path)
            props = dbus.Interface(service, PROP_INTERFACE)
            if props.Get(GATT_SERVICE_INTERFACE, "UUID") == self.SVC_UUID:
                logger.info('Service found: {}'.format(path))
                return service
        self._adapter.remove_device(self._device)
        raise SystemExit("Bluetooth Service not found!") 
{% endhighlight %}

Select Characteristic
---------------------

Almost done at this point, the last steps are finding the right dbus 
`Characteristic` and enable notifications. The first can be done with:

{% highlight python %}
    def find_chrc(self, MSRMT_UUID):
        for path, ifaces in self._get_managed_objects().items():
            if GATT_CHRC_INTERFACE not in ifaces.keys():
                continue
            chrc = self._bus.get_object('org.bluez', path)
            props = dbus.Interface(chrc, PROP_INTERFACE)
            if props.Get(GATT_CHRC_INTERFACE, "UUID") == MSRMT_UUID:
                logger.info('Characteristic found: {}'.format(path))
                return chrc
        self._adapter.remove_device(self._device)
        raise SystemExit("Bluetooth Characteristic not found!")
{% endhighlight %}

Again it's just a matter of querying the right dbus interface after a new call 
to the object manager (basically to refresh the list of known objects).

Enable Notifications
--------------------

Finally triggering notifications can be done thanks to the [GATT dbus 
API][gatt-api], with a call to `StartNotify`.

{% highlight python %}
    def check_notification(self, chrc, timeout=20):
        # Listen to PropertiesChanged signals from the BLE Measurement
        # Characteristic.
        prop_iface = dbus.Interface(chrc, PROP_INTERFACE)
        prop_iface.connect_to_signal("PropertiesChanged", self._changed_cb)

        # Subscribe to BLE Measurement notifications.
        chrc.StartNotify(reply_handler=self._start_notify_cb,
                         error_handler=self._generic_error_cb,
                         dbus_interface=GATT_CHRC_INTERFACE)
        GObject.timeout_add_seconds(timeout, self._notify_timeout)
        self._main_loop.run()
{% endhighlight %}

The main callback setup here is `_changed_cb` as it will be triggered every 
time a new raw notification is set by the Thingy.

{% highlight python %}
    def _changed_cb(self, iface, changed_props, invalidated_props):
        if iface != GATT_CHRC_INTERFACE:
            return
        if not len(changed_props):
            return
        value = changed_props.get('Value', None)
        if not value:
            return
        logger.debug('New Notification')
        self._notifications += 1
        if self._notifications >= self._max_notif:
            logger.info('Notification test succeeded')
            self._main_loop.quit()
{% endhighlight %}

The full script can be found 
[here](https://git.launchpad.net/plainbox-provider-checkbox/tree/bin/gatt-notify-test.py).

Sample output:

{% highlight console %}
Adapter found: [ /org/bluez/hci1 ] EE:B1:E5:DB:C7:A4 - C663432546-00515 #2
Adapter powered on
Adapter scan on (10s)
Adapter scan completed
/org/bluez/hci1/dev_DA_9A_D3_B0_0F_DB DA:9A:D3:B0:0F:DB QAThingy
Device found: [ /org/bluez/hci1/dev_DA_9A_D3_B0_0F_DB ] QAThingy - DA:9A:D3:B0:0F:DB
Device connected, waiting 10s for services to be available
Service found: /org/bluez/hci1/dev_DA_9A_D3_B0_0F_DB/service002f
Characteristic found: /org/bluez/hci1/dev_DA_9A_D3_B0_0F_DB/service002f/char003e
Notifications enabled
New Notification
New Notification
New Notification
New Notification
New Notification
Notification test succeeded
Device properly disconnected
Device properly removed
{% endhighlight %}

Cleanup of connections
----------------------

Because Bluez keeps track of all known devices (even if not currently 
connected), they can still be reported by the object manager. A best practice 
is to remove them properly and not just disconnect them. That's why I recommend 
the following calls:

{% highlight python %}
device.Disconnect()
adapter.remove_device(device)
{% endhighlight %}

According to the documentation, needless to power off the adapter, once the 
script is over the controller automatically goes back to off:

> The value of this property is not persistent. After
> restart or unplugging of the adapter it will reset
> back to false.

Possible improvements
---------------------

Enforce device check in GATT service discovery.

[^1]: [Doing Bluetooth Low Energy on Linux][BLE_on_Linux] (page 22)

[certification]:    https://certification.ubuntu.com/
[gatt]:             https://www.bluetooth.com/specifications/gatt/generic-attributes-overview
[bluez-git-doc]:    https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc
[BLE_on_Linux]:     https://elinux.org/images/3/32/Doing_Bluetooth_Low_Energy_on_Linux.pdf
[Nordic-Thingy-52]: https://www.nordicsemi.com/eng/Products/Nordic-Thingy-52
[python3-dbus]:     https://packages.ubuntu.com/xenial/python3-dbus
[python3-gi]:       https://packages.ubuntu.com/xenial/python3-gi
[p-p-c-parts]:      https://git.launchpad.net/~checkbox-dev/plainbox-provider-checkbox/+git/plainbox-provider-checkbox-parts/tree/snapcraft.yaml#n47
[bt_helper]:        https://git.launchpad.net/checkbox-support/tree/checkbox_support/bt_helper.py
[adapter-api]:      https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc/adapter-api.txt
[gatt-api]:         https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc/gatt-api.txt
[motion-service]:   https://nordicsemiconductor.github.io/Nordic-Thingy52-FW/documentation/firmware_architecture.html#arch_motion
[dbus]:             https://www.freedesktop.org/wiki/Software/dbus/
