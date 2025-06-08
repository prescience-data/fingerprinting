---
title: window.MediaDeviceInfo
layout: default
---
# window.MediaDeviceInfo
## Purpose
The `MediaDeviceInfo` interface provides metadata about a single media input or output device, such as a microphone, camera, or speaker. It is primarily accessed via the `navigator.mediaDevices.enumerateDevices()` method, which returns a list of all available media devices.

## Fingerprinting Usage
`MediaDeviceInfo` is a high-entropy source for browser fingerprinting and a critical signal for bot detection. The list of devices, their types, and their properties create a unique signature for a user's hardware configuration.

*   **Device Count and Type:** The sheer number of devices and the count of each `kind` (`audioinput`, `videoinput`, `audiooutput`) is a primary fingerprinting vector. A headless browser in its default configuration will often report an empty list, which is a strong bot signal.
*   **Device Labels (`label`):** This is the most potent property. Before a user grants microphone or camera permissions, the `label` is an empty string for privacy. After permission is granted, the `label` reveals the specific hardware name (e.g., "FaceTime HD Camera", "Logitech C920 HD Pro Webcam", "AirPods Pro Microphone"). This provides extremely high entropy. Anti-bot systems can check for generic or known-spoofed labels.
*   **Device and Group IDs (`deviceId`, `groupId`):**
    *   `deviceId`: A persistent, per-origin unique identifier for the device. While it changes for different websites, its stability within a single origin allows for tracking a returning user.
    *   `groupId`: Identifies devices that belong to the same physical unit (e.g., the microphone and camera on a single webcam). The pattern of `groupId`s across a user's devices adds another layer to the fingerprint.
*   **Anomalies and Inconsistencies:**
    *   **Absence:** As mentioned, a complete lack of media devices is highly indicative of a headless automation environment (e.g., Puppeteer, Playwright).
    *   **Virtual Devices:** The presence of virtual devices (e.g., "VB-Audio Virtual Cable", "OBS Virtual Camera") can be flagged. While legitimate, they are also commonly used in sophisticated bot setups.
    *   **Inconsistency with User-Agent:** A browser identifying as a MacBook Pro via its User-Agent string but lacking devices named "FaceTime HD Camera" or "MacBook Pro Microphone" is a strong indicator of spoofing.

## Sample Code
```javascript
async function getMediaDeviceFingerprint() {
  try {
    // Check for API support first
    if (!navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices) {
      return { error: "MediaDevices API not supported." };
    }

    const devices = await navigator.mediaDevices.enumerateDevices();

    // A common bot signal is having no media devices at all.
    if (devices.length === 0) {
      return { fingerprint: "NO_DEVICES_DETECTED", isBotSignal: true };
    }

    const deviceInfo = devices.map(device => {
      // Extract key properties for the fingerprint.
      return `${device.kind}:${device.label}:${device.groupId}`;
    });

    // The final fingerprint is often a hash of the sorted device strings.
    // Sorting ensures a consistent order.
    const fingerprint = deviceInfo.sort().join(';');
    
    // Example: "audioinput::group1;audiooutput::group1;videoinput:FaceTime HD Camera:group2"
    // A real implementation would hash this string (e.g., with SHA-256).
    return { fingerprint, isBotSignal: false };

  } catch (err) {
    // Errors can also be a signal (e.g., if the API was disabled or spoofed).
    return { error: err.name, message: err.message };
  }
}

getMediaDeviceFingerprint().then(result => {
  console.log(result);
});
```