# Biketerra - Tauri app spec

The purpose of the Biketerra Tauri app is to provide a better Bluetooth experience for users.

## Limitations of `web-bluetooth`

Biketerra currently relies on `web-bluetooth` (built into Chromium-based browsers), which has major limitations:

* Scanning / pairing is extremely slow vs. native
* Users can only select a single device at a time
* Users need to manually re-pair all devices on every pageload
* No ability to "remember" devices
* A 10 year-old [Chromium bug](https://issues.chromium.org/issues/40502943) causes `web-bluetooth` to crash upon dropout and unable to re-connect within a certain time limit

## Requirements

Build a cross-platform (Windows / Mac / iOS / Android) app using `Tauri 2` and `btleplug`.

*The app will consists of 3 parts:*

* An external webview (pointing to https://biketerra.com)
* A Bluetooth API
* A communication layer, allowing data exchange between the webview and the Bluetooth API

*The high-level goals of the app:*

* Quickly and easily pair their Bluetooth devices
* Ability to select multiple devices at once
* Automatic handling of (and reconnection of) dropouts
* Persistent bluetooth connection across webview pageloads (inherent to native, not supported by web-bluetooth)

## Bluetooth API

Using `btleplug`, here are the envisioned methods:

### scan(serviceUuids)

Run a 10 second scan for devices. Fire off an event for each detected device that matches one of the `serviceUuids`.

```js
// events sent to the webview
event: `scan-started`
event: `device-found`, data: `{id: <deviceId>, name: <deviceName>}`
event: `scan-finished`
```

### connect(deviceIds)

Connect to one or more devices.

### disconnect(deviceIds)

Disconnect from one or more devices.

### sendCommand(deviceId, command, hasResponse = false)

