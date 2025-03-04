# Biketerra - Tauri app spec

Biketerra is a web-based 3D virtual cycling app.

Users pair their bluetooth fitness devices (smart trainer, power meter, heart rate monitor, etc). Biketerra displays the various metrics, and uses the power readings to "move" the user's avatar along the route. It also supports 2-way communication with smart trainers, for things like adjusting resistance based on terrain grade.

<img src="https://uploads.biketerra.com/homepage/v03/workouts.webp" alt="Biketerra 3D" style="width: 48rem" />

## Purpose of the native app

The purpose of this app is to provide a better Bluetooth experience for users.

Biketerra currently relies on `web-bluetooth` (built into Chromium-based browsers), which has major limitations:

* Scanning / pairing is extremely slow vs. native
* Users need to pair devices one-at-a-time (very inconvenient for users needing to pair numerous devices)
* Users need to manually re-pair all devices on every pageload
* No ability to "remember" devices
* A 10 year-old [Chromium bug](https://issues.chromium.org/issues/40502943) causes `web-bluetooth` to crash when unable to reconnect after dropouts

## Goals

Build a cross-platform (Windows / Mac / iOS / Android) app using `Tauri 2` and `btleplug`.

*The app will consists of 3 parts:*

* An external webview (pointing to https://biketerra.com)
* A Bluetooth API (see below)
* A communication layer between the webview and the Bluetooth API

*The high-level goals of the app:*

* Quickly and easily pair Bluetooth devices (incl. multiple devices at once)
* Automatic handling of dropouts -- advertisement listener with timeout + reconnect
* Persistent bluetooth connections across webview pageloads

## Bluetooth API

Using `btleplug`, here are the envisioned methods:

### scan(filterUuids)

Run a 10 second scan for devices. Fire off an event for each detected device containing 1 or more services matching `filterUuids`. Store the `filterUuids` array for later.

```js
// events sent from Tauri/Rust to the webview
emit('scan', {status: 'started'});
emit('scan', {status: 'device-found', id: <deviceId>, name: <deviceName>});
emit('scan', {status: 'stopped'});
```

`filterUuids` contains UUIDs for both `services` and `characteristics`. We'd ideally also use `filterUuids` in the connect() method as well, to include only matching service characteristics.

### connect(deviceIds)

Connect to one or more devices, handle dropouts, call `_listen` for any relevant characteristics.

```js
emit('connect', {status: 'connecting', id: <deviceId>, retrying: false});
emit('connect', {status: 'success', id: <deviceId>, services: <services>});
emit('connect', {status: 'fail', id: <deviceId>, reason: <string>});
```

Example of `<services>`:

```js
{
    '00001818-0000-1000-8000-00805f9b34fb': { // service UUID
        '00002a63-0000-1000-8000-00805f9b34fb': {props: ['notify']} // {<characteristicUUID>: props, value: <valueIfHasReadProp>}
    },
    '00001826-0000-1000-8000-00805f9b34fb': {
        '00002ad2-0000-1000-8000-00805f9b34fb': {props: ['read', 'notify', 'writeWithResponse'], value: <Uint8Array>},
        '00002ad9-0000-1000-8000-00805f9b34fb': {props: ['notify']}
    }
}
```

Bluetooth characteristics contain property flags to indicate which features are available (read, write, writeWithResponse, notify, etc)

### disconnect(deviceIds)

Disconnect from one or more devices. Call `_unlisten` for any relevant characteristics.

```js
emit('disconnect', {status: 'success', id: <deviceId>});
emit('disconnect', {status: 'fail', id: <deviceId>, reason: <string>});
```

### writeValue(deviceId, serviceUuid, characteristicUuid, value)

Write to a characteristic. The `value` will be a Uint8Array. The response will depend on whether the characteristic has a `writeWithoutResponse` property.

### readValue(deviceId, serviceUuid, characteristicUuid)

Read and return the value from a characteristic. Mainly useful for periodically grabbing updated battery percentages.

### _listen(characteristic)

(Internal) called from `connect`. If the characteristic has "notify" or "indicate" properties, then listen for BLE notifications and fire an event when detected:

```js
emit('notifyValue', {id: <deviceId>, service: <serviceUuid>, characteristic: <characteristicUuid>, value: <value>});
```

### _unlisten(characteristic)

(Internal) called from `disconnect`. Stop BLE notifications for this characteristic if in use.
