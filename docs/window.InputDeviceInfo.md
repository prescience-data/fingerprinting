---
title: window.InputDeviceInfo
layout: default
---
# window.InputDeviceInfo
## Purpose
The `InputDeviceInfo` interface, which inherits from `MediaDeviceInfo`, represents a single media input device like a webcam or microphone. Its primary purpose is to allow a web application to query the specific hardware capabilities (e.g., supported resolutions, frame rates) of that device via the `getCapabilities()` method.

## Fingerprinting Usage
`InputDeviceInfo` is a high-entropy source for fingerprinting and a critical API for bot detection due to the detailed hardware information it exposes. It is accessed by calling `navigator.mediaDevices.enumerateDevices()`.

*   **Hardware-Specific Capabilities:** The primary fingerprinting vector is the `MediaTrackCapabilities` object returned by `getCapabilities()`. The combination of supported `width`, `height`, `frameRate`, `aspectRatio`, `sampleRate` (for audio), and `facingMode` (for mobile) is highly specific to the exact hardware model of the webcam or microphone. A cheap generic webcam will report different capabilities than a high-end Logitech Brio or a built-in MacBook FaceTime HD Camera.

*   **Device Enumeration:** The number, `kind` (`audioinput`, `videoinput`), and `groupId` (which links devices from the same physical hardware, like a headset's mic and speakers) of all enumerated devices form a stable fingerprint for a user's setup. A system with one webcam and two microphones is distinct from one with no webcam and one microphone.

*   **Virtual Device Detection:** This is a cornerstone of bot detection. Bots and privacy-conscious users often use virtual media drivers (e.g., OBS Virtual Camera, VB-CABLE, DroidCam). The `label` property of the device often contains a tell-tale string like `"CABLE Output (VB-Audio Virtual Cable)"` or `"OBS Virtual Cam"`. Anti-bot systems maintain blocklists of these labels to immediately flag a session as suspicious.

*   **Consistency Checks:** The reported device capabilities can be cross-referenced with other browser properties for consistency. For example, if a `User-Agent` string indicates a specific model of iPhone, the `InputDeviceInfo` should report front and back cameras (`facingMode: 'user'` and `'environment'`) with capabilities that match Apple's hardware specifications. A mismatch, such as a desktop `User-Agent` reporting mobile camera capabilities, is a strong bot signal.

*   **Behavioral Signals:** The `label` property is empty until the user grants media permissions (`getUserMedia`). A bot may fail to correctly simulate this state change, or it may expose virtual device labels from the start, which is anomalous behavior.

## Sample Code
This snippet demonstrates how to enumerate media devices and extract their capabilities for fingerprinting analysis.

```javascript
async function getMediaDeviceFingerprint() {
  try {
    // This call does not require user permission, but device labels will be empty.
    // The presence and capabilities of devices are still available.
    const devices = await navigator.mediaDevices.enumerateDevices();
    const fingerprintData = [];

    for (const device of devices) {
      const deviceInfo = {
        kind: device.kind,
        label: device.label, // Key signal for virtual devices (e.g., "OBS Virtual Camera")
        deviceId: device.deviceId, // Randomized per-origin, but its presence/count is a signal
        groupId: device.groupId,   // Links related devices (e.g., mic/speaker on a headset)
      };

      // The getCapabilities() method is unique to InputDeviceInfo.
      // This is the highest-entropy signal, revealing specific hardware model capabilities.
      if (typeof device.getCapabilities === 'function') {
        deviceInfo.capabilities = device.getCapabilities();
      }
      
      fingerprintData.push(deviceInfo);
    }

    // The resulting array is a rich fingerprint of the user's media hardware.
    // It can be hashed or analyzed for anomalies.
    console.log('Media Device Fingerprint:', fingerprintData);
    return fingerprintData;

  } catch (error) {
    // An error itself can be a fingerprinting signal (e.g., API is disabled or spoofed).
    console.error('Media device enumeration failed:', error.name);
    return { error: error.name };
  }
}

getMediaDeviceFingerprint();
```