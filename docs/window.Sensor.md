---
title: window.Sensor
layout: default
---
# window.Sensor
## Purpose
The `Sensor` interface is the base class for all sensor APIs within the Generic Sensor API framework. It is not used directly but provides the core properties and methods (e.g., `start()`, `stop()`, `onreading`) inherited by concrete sensor interfaces like `Accelerometer`, `Gyroscope`, and `AmbientLightSensor`.

## Fingerprinting Usage
The Generic Sensor API is a high-value target for fingerprinting and bot detection due to the rich, hardware-specific data it exposes.

*   **Hardware Enumeration:** The primary use is to detect the presence or absence of specific sensors. Most desktop machines have no sensors, while mobile devices have a distinct combination (e.g., accelerometer, gyroscope, magnetometer). Headless browsers (Puppeteer, Playwright) and virtual machines typically lack sensor support entirely, making their absence a strong signal of automation. An anti-bot script can attempt to instantiate various sensors and record which ones succeed or fail.

*   **High-Entropy Data Stream:** For devices with sensors, the stream of reading data is a powerful fingerprint. Due to microscopic manufacturing imperfections, no two sensors are identical. The subtle noise and calibration offsets in the data from an accelerometer or gyroscope, even when the device is stationary, create a unique and persistent hardware identifier. This is often called a "motion fingerprint" or "sensor calibration fingerprint."

*   **Behavioral Analysis:** The pattern of sensor data reveals how a user interacts with their device. A real user holding a phone will generate a constant, low-amplitude stream of noise from natural hand tremors. A bot will either provide no data, perfectly static data (e.g., `x:0, y:0, z:0`), or mathematically perfect, artificial data (e.g., a perfect sine wave) that is easily distinguishable from noisy human movement.

*   **Permissions State:** Access to sensors requires user permission. Scripts can use the Permissions API (`navigator.permissions.query`) to check the status (`granted`, `denied`, `prompt`) for sensors like `'accelerometer'` without triggering a user prompt. This permission state contributes to the overall browser fingerprint.

## Sample Code
This snippet attempts to instantiate an `Accelerometer`. The success or failure, and the subsequent data stream, are key fingerprinting signals.

```javascript
const fingerprint = {
  hasAccelerometer: false,
  accelerometerReadings: [],
};

try {
  // Attempt to create a sensor instance. This fails on most desktops and in many bots.
  const accelerometer = new Accelerometer({ frequency: 10 }); // Hz

  accelerometer.onreading = () => {
    // A real device produces a noisy stream of data even when still.
    // A fingerprinting script would hash or analyze this stream.
    const reading = {
      x: accelerometer.x,
      y: accelerometer.y,
      z: accelerometer.z,
      timestamp: performance.now(),
    };
    fingerprint.accelerometerReadings.push(reading);

    // Stop after collecting a few samples to build the fingerprint.
    if (fingerprint.accelerometerReadings.length >= 5) {
      accelerometer.stop();
      console.log('Accelerometer fingerprint captured:', fingerprint.accelerometerReadings);
    }
  };

  accelerometer.onerror = (event) => {
    // An error during reading is also a signal.
    console.error('Sensor Error:', event.error.name, event.error.message);
  };

  accelerometer.start();
  fingerprint.hasAccelerometer = true;

} catch (error) {
  // Catching an error (e.g., ReferenceError or NotSupportedError) is the most common
  // outcome in a non-mobile or automated environment. This is a strong bot signal.
  fingerprint.hasAccelerometer = false;
  console.log('Accelerometer not supported. Likely a bot or desktop.', error.name);
}
```