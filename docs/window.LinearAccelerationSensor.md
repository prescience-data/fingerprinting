---
title: window.LinearAccelerationSensor
layout: default
---
# window.LinearAccelerationSensor
## Purpose
Part of the Sensor APIs, `LinearAccelerationSensor` measures acceleration applied to the device on three axes (x, y, z), excluding the contribution of gravity. It provides a direct measure of the acceleration caused by the user or external forces acting on the device.

## Fingerprinting Usage
This API is a potent tool for both fingerprinting and bot detection, primarily on mobile devices where the sensor is present.

1.  **Device-Type Classification:** The mere presence of the `window.LinearAccelerationSensor` constructor is a strong signal. Its existence reliably separates mobile devices and some tablets/laptops from the vast majority of desktop machines, which lack the necessary hardware. Bots running in standard server or desktop-based headless environments will typically not expose this API.

2.  **Hardware Fingerprinting:** If the sensor can be activated, its output reveals hardware-specific characteristics. Even on a perfectly stationary device, physical sensors produce a constant stream of low-level noise due to hardware imperfections. The statistical properties of this noise (e.g., standard deviation, frequency distribution) can form a high-entropy fingerprint unique to the specific sensor chip. The supported `frequency` of the sensor can also differ between device models.

3.  **Behavioral Biometrics & Bot Detection:** This is the most powerful use case.
    *   **Human Interaction:** A human holding a device produces a continuous, complex, and non-repeating stream of acceleration data from natural hand tremors and subtle movements. Actions like tilting the phone to scroll, tapping the screen, or walking generate characteristic patterns that are extremely difficult for a bot to fake realistically.
    *   **Bot Anomaly:** A bot will typically exhibit one of these behaviors:
        *   **No Data:** The API is missing or throws an error upon instantiation (a strong signal of a headless environment).
        *   **Zero-Value Data:** The sensor reports a constant `(0, 0, 0)`, indicating a perfectly stationary, emulated device with no noise model. This is physically impossible for a real device.
        *   **Synthetic Data:** The bot attempts to generate fake data. This is often detectable as it may be too perfect, too random (lacking the autocorrelation of human movement), or follow simple mathematical patterns (e.g., a sine wave).

4.  **Error Analysis:** The way the API fails is a signal. Instantiating the sensor is wrapped in a `try...catch` block. A headless browser might throw a specific `DOMException` (e.g., `NotSupportedError`), while a real device might trigger a permission prompt. The lack of a prompt or a specific error type can be used to flag automated environments.

## Sample Code
This snippet demonstrates how an anti-bot system might probe the sensor for behavioral and environmental signals.

```javascript
async function analyzeMotionSensor() {
  const signals = {
    apiExists: 'LinearAccelerationSensor' in window,
    permissionState: 'unknown',
    instantiationError: null,
    readings: [],
  };

  if (!signals.apiExists) {
    // Strong signal: Likely a desktop or basic headless browser.
    return signals;
  }

  try {
    // Check permission state without prompting the user.
    const perm = await navigator.permissions.query({ name: 'accelerometer' });
    signals.permissionState = perm.state; // 'granted', 'denied', or 'prompt'

    // Attempt to instantiate the sensor. This can fail in headless environments.
    const sensor = new LinearAccelerationSensor({ frequency: 60 });

    sensor.addEventListener('error', (event) => {
      // Catching errors is a key signal for bot detection.
      // e.g., 'NotReadableError' can indicate a hardware or driver issue.
      signals.instantiationError = event.error.name;
      sensor.stop();
    }, { once: true });

    sensor.addEventListener('reading', () => {
      signals.readings.push({ x: sensor.x, y: sensor.y, z: sensor.z });
      // In a real scenario, collect ~50-100 readings for analysis.
      if (signals.readings.length >= 10) {
        sensor.stop();
        // Now analyze the 'readings' array for noise patterns or human-like motion.
        // A bot might return all zeros or predictable patterns.
      }
    }, { once: true }); // Use 'once' for a brief sample.

    sensor.start();

  } catch (error) {
    // This catch block is a primary detection vector.
    // Headless Chrome often throws 'NotSupportedError' here.
    signals.instantiationError = error.name;
  }

  // After a short delay, the 'signals' object contains a rich fingerprint.
  return signals;
}

// Example usage:
// analyzeMotionSensor().then(fingerprint => console.log(fingerprint));
```