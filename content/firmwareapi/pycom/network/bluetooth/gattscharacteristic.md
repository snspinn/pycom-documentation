---
title: "GATTSCharacteristic"
aliases:
    - firmwareapi/pycom/network/bluetooth/gattscharacteristic.html
    - firmwareapi/pycom/network/bluetooth/gattscharacteristic.md
---

The smallest concept in GATT is the Characteristic, which encapsulates a single data point (though it may contain an array of related data, such as X/Y/Z values from a 3-axis accelerometer, longitude and latitude from a GPS, etc.).

The following class allows you to manage Server characteristics.

## Methods

#### characteristic.value(\[value\])

Gets or sets the value of the characteristic. Can take an integer, a string or a bytes object.

```python

characteristic.value(123) # set characteristic value to an integer with the value 123
characteristic.value() # get characteristic value
```

#### characteristic.events()

Returns a value with bit flags, identifying the events that have occurred since the last call. Calling this function clears the events.

#### characteristic.callback(trigger=None, handler=None, arg=None)

Creates a callback that will be executed when any of the triggers occur. The arguments are:

* `trigger` can be either `Bluetooth.CHAR_READ_EVENT` or `Bluetooth.CHAR_WRITE_EVENT`.
* `handler` is the function that will be executed when the callback is triggered.
* `arg` is the argument that gets passed to the callback. If nothing is given, the characteristic object that owns the callback will be used.

Beyond the `arg` a tuple (called `data`) is also passed to `handler`. The tuple consists of (event, value), where `event` is the triggering event and `value` is the value strictly belonging to the `event` in case of a WRITE event. If the `event` is not a WRITE event, the `value` has no meaning.

We recommend getting both the `event` and new `value` of the characteristic via this tuple, and not via `characteristic.event()` and `characteristic.value()` calls in the context of the `handler` to make sure no event and value is lost.
The reason behind this is that `characteristic.event()` and `characteristic.value()` return with the very last event received and with the current value of the characteristic, while the input parameters are always linked to the specific event triggering the `handler`. If the device is busy executing other operations,  the `handler` of an incoming event may not be called before the next event occurs and is processed.

An example of how this can be implemented is shown below, via an example of advertising and creating services on the device:

```python

from network import Bluetooth

def conn_cb (bt_o):
    events = bt_o.events()
    if  events & Bluetooth.CLIENT_CONNECTED:
        print("Client connected")
    elif events & Bluetooth.CLIENT_DISCONNECTED:
        print("Client disconnected")

def char1_cb_handler(chr, data):

    # The data is a tuple containing the triggering event and the value if the event is a WRITE event.
    # We recommend fetching the event and value from the input parameter, and not via characteristic.event() and characteristic.value()
    events, value = data
    if  events & Bluetooth.CHAR_WRITE_EVENT:
        print("Write request with value = {}".format(value))
    else:
        print('Read request on char 1')

def char2_cb_handler(chr, data):
    # The value is not used in this callback as the WRITE events are not processed.
    events, value = data
    if  events & Bluetooth.CHAR_READ_EVENT:
        print('Read request on char 2')

bluetooth = Bluetooth()
bluetooth.set_advertisement(name='LoPy', service_uuid=b'1234567890123456')
bluetooth.callback(trigger=Bluetooth.CLIENT_CONNECTED | Bluetooth.CLIENT_DISCONNECTED, handler=conn_cb)
bluetooth.advertise(True)

srv1 = bluetooth.service(uuid=b'1234567890123456', isprimary=True)
chr1 = srv1.characteristic(uuid=b'ab34567890123456', value=5)
char1_cb = chr1.callback(trigger=Bluetooth.CHAR_WRITE_EVENT | Bluetooth.CHAR_READ_EVENT, handler=char1_cb_handler)

srv2 = bluetooth.service(uuid=1234, isprimary=True)
chr2 = srv2.characteristic(uuid=4567, value=0x1234)
char2_cb = chr2.callback(trigger=Bluetooth.CHAR_READ_EVENT, handler=char2_cb_handler)
```

