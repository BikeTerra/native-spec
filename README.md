# Biketerra - Tauri app spec

The purpose of this app is to provide a better Bluetooth experience for users.

Biketerra currently relies on `web-bluetooth` (built into Chromium-based browsers), which has major limitations:

* Scanning / pairing is extremely slow vs. native
* Users can only select a single device at a time
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
* Automatic handling of dropouts -- automatic advertisement listener with timeout + reconnect
* Persistent bluetooth connections across webview pageloads

## Bluetooth API

Using `btleplug`, here are the envisioned methods:

### scan(filterUuids)

Run a 10 second scan for devices. Fire off an event for each detected device that matches one of the `filterUuids`. Store the `filterUuids` array for later.

```js
emit('scan', {status: 'started'});
emit('scan', {status: 'device-found', data: {id: <deviceId>, name: <deviceName>}});
emit('scan', {status: 'stopped'});
```

### connect(deviceIds)

Connect to one or more devices, handle dropouts, and listen for notifications for characteristics containing `notify` or `indicate` properties.

```js
emit('connect', {status: 'connecting', data: {id: <deviceId>, reconnecting: false}}); // reconnecting = dropout handling
emit('connect', {status: 'success', data: {id: <deviceId>, services: <services>}});
emit('connect', {status: 'fail', data: {id: <deviceId>, reason: <string>}});
emit('notifyValue', {id: <deviceId>, service: <serviceUuid>, characteristic: <characteristicUuid>, value: <value>});
```

Example of `<services>`:

```js
{
    '00001818-0000-1000-8000-00805f9b34fb': [
        '00002a63-0000-1000-8000-00805f9b34fb'
    ],
    '00001826-0000-1000-8000-00805f9b34fb': [
        '00002ad2-0000-1000-8000-00805f9b34fb'
        '00002ad9-0000-1000-8000-00805f9b34fb'
    ]
}
```

The outer UUIDs are `services`, the inner ones are `characteristics`. Each characteristic contains properties (read, write, notify, etc)

### disconnect(deviceIds)

Disconnect from one or more devices.

```js
emit('disconnect', {status: 'success', data: {id: <deviceId>}});
emit('disconnect', {status: 'fail', data: {id: <deviceId>, reason: <string>}});
```

### sendCommand(deviceId, serviceUuid, characteristicUuid, command)

Write to a characteristic, returning response data. The `command` will contain a Uint8Array value.

### readValue(deviceId, serviceUuid, characteristicUuid)

Read and return the value from a characteristic.
