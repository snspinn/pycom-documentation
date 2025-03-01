---
title: "Bluetooth"
aliases:
    - chapter/firmwareapi/pycom/network/bluetooth
---

This class provides a driver for the Bluetooth radio in the module. Currently, only basic BLE functionality is available.

## Quick Usage Example

```python
from network import Bluetooth
import time
bt = Bluetooth()
bt.start_scan(-1)

while True:
  adv = bt.get_adv()
  if adv and bt.resolve_adv_data(adv.data, Bluetooth.ADV_NAME_CMPL) == 'Heart Rate':
      try:
          conn = bt.connect(adv.mac)
          services = conn.services()
          for service in services:
              time.sleep(0.050)
              if type(service.uuid()) == bytes:
                  print('Reading chars from service = {}'.format(service.uuid()))
              else:
                  print('Reading chars from service = %x' % service.uuid())
              chars = service.characteristics()
              for char in chars:
                  if (char.properties() & Bluetooth.PROP_READ):
                      print('char {} value = {}'.format(char.uuid(), char.read()))
          conn.disconnect()
          break
      except:
          print("Error while connecting or reading from the BLE device")
          break
  else:
      time.sleep(0.050)
```

## Bluetooth Low Energy (BLE)

Bluetooth low energy (BLE) is a subset of classic Bluetooth, designed for easy connecting and communicating between devices (in particular mobile platforms). BLE uses a methodology known as Generic Access Profile (GAP) to control connections and advertising.

GAP allows for devices to take various roles but generic flow works with devices that are either a Server (low power, resource constrained, sending small payloads of data) or a Client device (commonly a mobile device, PC or Pycom Device with large resources and processing power). Pycom devices can act as both a Client and a Server.

## Constructors

#### class network.Bluetooth(id=0, ...)

Create a Bluetooth object, and optionally configure it. See init for params of configuration.

Example:

```python
from network import Bluetooth
bluetooth = Bluetooth()
```

## Methods

### bluetooth.init([id=0, mode=Bluetooth.BLE, antenna=Bluetooth.INT_ANT, modem_sleep=True, pin=None, privacy=True, secure_connections=True, mtu=200])


* `id` Only one Bluetooth peripheral available so must always be 0
* `mode` currently the only supported mode is `Bluetooth.BLE`
* `modem_sleep` Enables or Disables BLE modem sleep, Disable modem sleep as a workaround when having Crashes due to flash cache being disabled, as this prevents BLE task saving data in external RAM while accesing external flash for R/W	
* `antenna` selects between the internal and the external antenna. Can be either:
  * `bluetooth.INT_ANT`: The on-board antenna
  * `bluetooth.EXT_ANT`: The U.FL connector (external antenna)
  * `bluetooth.MAN_ANT`: Manually select the state of the antenna switch on `P12`. By default, this will select the on-board antenna
* `secure` enables or disables the GATT Server security feature
* `pin` a six digit number to connect to the GATT Sever	Setting any valid pin, GATT Server security features are activated.
* `privacy` Enables or Disables local privacy settings so address will be random or public.
* `secure_connections` Enables or Disables Secure Connections and MITM Protection.
* `mtu` Maximum Transmission Unit (MTU) is the maximum length of an ATT packet. Value must be between `23` and `200`.

With our development boards it defaults to using the internal antenna, but in the case of an OEM module, the antenna pin (`P12`) is not used, so it's free to be used for other things.
    Initialises and enables the Bluetooth radio in BLE mode.



### bluetooth.deinit()

Disables the Bluetooth radio.

### bluetooth.start_scan(timeout)

Starts performing a scan listening for BLE devices sending advertisements. This function always returns immediately, the scanning will be performed on the background. The return value is `None`. After starting the scan the function `get_adv()` can be used to retrieve the advertisements messages from the FIFO. The internal FIFO has space to cache 16 advertisements.

The arguments are:

* `timeout` specifies the amount of time in seconds to scan for advertisements, cannot be zero. If timeout is &gt; 0, then the BLE radio will listen for advertisements until the specified value in seconds elapses. If timeout &lt; 0, then there's no timeout at all, and stop\_scan() needs to be called to cancel the scanning process.

Examples:

```python
bluetooth.start_scan(10)        # starts scanning and stop after 10 seconds
bluetooth.start_scan(-1)        # starts scanning indefinitely until bluetooth.stop_scan() is called
```

### bluetooth.stop_scan()

Stops an ongoing scanning process. Returns `None`.

### bluetooth.isscanning()

Returns `True` if a Bluetooth scan is in progress. `False` otherwise.

### bluetooth.get_adv()

Gets an named tuple with the advertisement data received during the scanning. The tuple has the following structure: `(mac, addr_type, adv_type, rssi, data)`

* `mac` is the 6-byte ling mac address of the device that sent the advertisement.
* `addr_type` is the address type. See the constants section below for more details.
* `adv_type` is the advertisement type received. See the constants section below fro more details.
* `rssi` is signed integer with the signal strength of the advertisement.
* `data` contains the complete 31 bytes of the advertisement message. In order to parse the data and get the specific types, the method `resolve_adv_data()` can be used.

Example for getting `mac` address of an advertiser:

```python
import ubinascii

bluetooth = Bluetooth()
bluetooth.start_scan(20) # scan for 20 seconds

adv = bluetooth.get_adv() #
ubinascii.hexlify(adv.mac) # convert hexadecimal to ascii
```

### bluetooth.get_advertisements()

Same as the `get_adv()` method, but this one returns a list with all the advertisements received.

### bluetooth.resolve_adv_data(data, data_type)

Parses the advertisement data and returns the requested `data_type` if present. If the data type is not present, the function returns `None`.

Arguments:

* `data` is the bytes object with the complete advertisement data.
* `data_type` is the data type to resolve from from the advertisement data. See constants section below for details.

Example:

```python
import ubinascii
from network import Bluetooth
bluetooth = Bluetooth()

bluetooth.start_scan(20)
while bluetooth.isscanning():
    adv = bluetooth.get_adv()
    if adv:
        # try to get the complete name
        print(bluetooth.resolve_adv_data(adv.data, Bluetooth.ADV_NAME_CMPL))

        mfg_data = bluetooth.resolve_adv_data(adv.data, Bluetooth.ADV_MANUFACTURER_DATA)

        if mfg_data:
            # try to get the manufacturer data (Apple's iBeacon data is sent here)
            print(ubinascii.hexlify(mfg_data))
```

### bluetooth.set_pin()

Configures a new PIN to be used by the device. The PIN is a 1-6 digit length decimal number, if less than 6 digits are given the missing leading digits are considered as 0. E.g. 1234 becomes 001234. When a new PIN is configured, the information of all previously bonded device is removed and the current connection is terminated. To restart advertisement the advertise() must be called after PIN is changed.

### bluetooth.connect(mac_addr, [timeout=None])

* `mac_addr` is the address of the remote device to connect
* `timeout` specifies the amount of time in milliseconds to wait for the connection process to finish. If not given then no timeout is applied The function blocks until the connection succeeds or fails (raises OSError) or the given `timeout` expires (raises `Bluetooth.timeout TimeoutError`). If the connections succeeds it returns a object of type `GATTCConnection`.

```python
bluetooth.connect('112233eeddff') # mac address is accepted as a string
```

### bluetooth.callback([trigger=None, handler=None, arg=None])

Creates a callback that will be executed when any of the triggers occurs. The arguments are:

* `trigger` can be either `Bluetooth.NEW_ADV_EVENT`, `Bluetooth.CLIENT_CONNECTED`, or `Bluetooth.CLIENT_DISCONNECTED`
* `handler` is the function that will be executed when the callback is triggered.
* `arg` is the argument that gets passed to the callback. If nothing is given the bluetooth object itself is used.

An example of how this may be used can be seen in the [`bluetooth.events()`](./#bluetooth-events) method.

### bluetooth.events()

Returns a value with bit flags identifying the events that have occurred since the last call. Calling this function clears the events.

Example of usage:

```python
from network import Bluetooth

bluetooth = Bluetooth()
bluetooth.set_advertisement(name='LoPy', service_uuid=b'1234567890123456')

def conn_cb (bt_o):
    events = bt_o.events()   # this method returns the flags and clears the internal registry
    if events & Bluetooth.CLIENT_CONNECTED:
        print("Client connected")
    elif events & Bluetooth.CLIENT_DISCONNECTED:
        print("Client disconnected")

bluetooth.callback(trigger=Bluetooth.CLIENT_CONNECTED | Bluetooth.CLIENT_DISCONNECTED, handler=conn_cb)

bluetooth.advertise(True)
```

### bluetooth.set_advertisement([name=None, manufacturer_data=None, service_data=None, service_uuid=None])

Configure the data to be sent while advertising. If left with the default of `None` the data won't be part of the advertisement message.

The arguments are:

* `name` is the string name to be shown on advertisements.
* `manufacturer_data` manufacturer data to be advertised (hint: use it for iBeacons).
* `service_data` service data to be advertised.
* `service_uuid` uuid of the service to be advertised.

Example:

```python
bluetooth.set_advertisement(name="advert", manufacturer_data="lopy_v1")
```

### bluetooth.set_advertisement_params(adv_int_min=0x20, adv_int_max=0x40, adv_type=Bluetooth.ADV_TYPE_IND, own_addr_type=Bluetooth.BLE_ADDR_TYPE_PUBLIC, channel_map=Bluetooth.ADV_CHNL_ALL, adv_filter_policy=Bluetooth.ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY)

Configure the parameters used when advertising.

The arguments are:

* `adv_int_min` is the minimum advertising interval for undirected and low duty cycle directed advertising.
* `adv_int_max` is the maximum advertising interval for undirected and low duty cycle directed advertising.
* `adv_type` is the advertising type.
* `own_addr_type` is the owner bluetooth device address type.
* `channel_map` is the advertising channel map.
* `adv_filter_policy` is the advertising filter policy.

### bluetooth.advertise([Enable])

Start or stop sending advertisements. The `set_advertisement()` method must have been called prior to this one.

#### bluetooth.service(uuid, [isprimary=True, nbr_chars=1, start=True])

Create a new service on the internal GATT server. Returns a object of type `BluetoothServerService`.

The arguments are:

* `uuid` is the UUID of the service. Can take an integer or a 16 byte long string or bytes object.
* `isprimary` selects if the service is a primary one. Takes a `bool` value.
* `nbr_chars` specifies the number of characteristics that the service will contain.
* `start` if `True` the service is started immediately.

```python
bluetooth.service('abc123')
```

### bluetooth.disconnect_client()

Closes the BLE connection with the client.

### bluetooth.tx_power(type, level)

Gets or sets the TX Power level.
If called with only `type` parameter it returns with the current value belonging to the given type.
If both `type` and `level` parameters are given, it sets the TX Power.

Valid values for `type`: `Bluetooth.TX_PWR_CONN` -> for handling connection, `Bluetooth.TX_PWR_ADV` -> for advertising, `Bluetooth.TX_PWR_SCAN` -> for scan, `Bluetooth.TX_PWR_DEFAULT` -> default, if others not set
Valid values for `level`: Bluetooth.TX_PWR_N12` -> -12dbm, `Bluetooth.TX_PWR_N9` -> -9dbm, `Bluetooth.TX_PWR_N6` -> -6dbm, `Bluetooth.TX_PWR_N3` -> -3dbm, `Bluetooth.TX_PWR_0` -> 0dbm, `Bluetooth.TX_PWR_P3` -> 3dbm, `Bluetooth.TX_PWR_P6` -> 6dbm, `Bluetooth.TX_PWR_P9` -> 9dbm

## Constants

* Bluetooth mode: `Bluetooth.BLE`
* Advertisement type: `Bluetooth.CONN_ADV`, `Bluetooth.CONN_DIR_ADV`, `Bluetooth.DISC_ADV`, `Bluetooth.NON_CONN_ADV`, `Bluetooth.SCAN_RSP`
* Address type: `Bluetooth.PUBLIC_ADDR`, `Bluetooth.RANDOM_ADDR`, `Bluetooth.PUBLIC_RPA_ADDR`, `Bluetooth.RANDOM_RPA_ADDR`
* Advertisement data type: `Bluetooth.ADV_FLAG`, `Bluetooth.ADV_16SRV_PART`, `Bluetooth.ADV_T16SRV_CMPL`, `Bluetooth.ADV_32SRV_PART`, `Bluetooth.ADV_32SRV_CMPL`, `Bluetooth.ADV_128SRV_PART`, `Bluetooth.ADV_128SRV_CMPL`, `Bluetooth.ADV_NAME_SHORT`, `Bluetooth.ADV_NAME_CMPL`, `Bluetooth.ADV_TX_PWR`, `Bluetooth.ADV_DEV_CLASS`, `Bluetooth.ADV_SERVICE_DATA`, `Bluetooth.ADV_APPEARANCE`, `Bluetooth.ADV_ADV_INT`, `Bluetooth.ADV_32SERVICE_DATA`, `Bluetooth.ADV_128SERVICE_DATA`, `Bluetooth.ADV_MANUFACTURER_DATA`
* Advertisement parameters: `Bluetooth.ADV_TYPE_IND`, `Bluetooth.ADV_TYPE_DIRECT_IND_HIGH`, `Bluetooth.ADV_TYPE_SCAN_IND`, `Bluetooth.ADV_TYPE_NONCONN_IND`, `Bluetooth.ADV_TYPE_DIRECT_IND_LOW`, `Bluetooth.ADV_BLE_ADDR_TYPE_PUBLIC`, `Bluetooth.ADV_BLE_ADDR_TYPE_RANDOM`, `Bluetooth.ADV_BLE_ADDR_TYPE_RPA_PUBLIC`, `Bluetooth.ADV_BLE_ADDR_TYPE_RPA_RANDOM`, `Bluetooth.ADV_CHNL_37`, `Bluetooth.ADV_CHNL_38`, `Bluetooth.ADV_CHNL_39`, `Bluetooth.ADV_CHNL_ALL`, `Bluetooth.ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY`, `Bluetooth.ADV_FILTER_ALLOW_SCAN_WLST_CON_ANY`, `Bluetooth.ADV_FILTER_ALLOW_SCAN_ANY_CON_WLST`, `Bluetooth.ADV_FILTER_ALLOW_SCAN_WLST_CON_WLST`
* Characteristic properties (bit values that can be combined): `Bluetooth.PROP_BROADCAST`, `Bluetooth.PROP_READ`, `Bluetooth.PROP_WRITE_NR`, `Bluetooth.PROP_WRITE`, `Bluetooth.PROP_NOTIFY`, `Bluetooth.PROP_INDICATE`, `Bluetooth.PROP_AUTH`, `Bluetooth.PROP_EXT_PROP`
* Characteristic callback events: `Bluetooth.CHAR_READ_EVENT`, `Bluetooth.CHAR_WRITE_EVENT`, `Bluetooth.NEW_ADV_EVENT`, `Bluetooth.CLIENT_CONNECTED`, `Bluetooth.CLIENT_DISCONNECTED`, `Bluetooth.CHAR_NOTIFY_EVENT`, `Bluetooth.CHAR_SUBSCRIBE_EVENT`
* Antenna type: `Bluetooth.INT_ANT`, `Bluetooth.EXT_ANT`, `Bluetooth.MAN_ANT`
* TX Power type: `Bluetooth.TX_PWR_CONN`, `Bluetooth.TX_PWR_ADV`, `Bluetooth.TX_PWR_SCAN`, `Bluetooth.TX_PWR_DEFAULT`
* TX Power level: `Bluetooth.TX_PWR_N12`, `Bluetooth.TX_PWR_N9`, `Bluetooth.TX_PWR_N6`, `Bluetooth.TX_PWR_N3`, `Bluetooth.TX_PWR_0`, `Bluetooth.TX_PWR_P3`, `Bluetooth.TX_PWR_P6`, `Bluetooth.TX_PWR_P9`

## Exceptions

* `Bluetooth.timeout`
