---
title: window.MediaKeySession
layout: default
---
# window.MediaKeySession
## Purpose
Part of the Encrypted Media Extensions (EME) API, `MediaKeySession` represents a session with a Content Decryption Module (CDM). Its primary function is to manage the exchange of licenses and keys required to decrypt and play DRM-protected media content, such as movies from streaming services.

## Fingerprinting Usage
The EME API, and by extension `MediaKeySession`, is a high-entropy source for fingerprinting and a strong signal for bot detection. It probes low-level, hardware-tied system components that are difficult for bots and spoofed environments to emulate correctly.

1.  **Supported Key Systems:** The primary fingerprinting vector is querying which DRM key systems the browser supports. Different operating systems and browsers support different combinations.
    *   `com.widevine.alpha`: Google Widevine (common on Chrome, Firefox, Android).
    *   `com.microsoft.playready`: Microsoft PlayReady (common on Edge, Windows).
    *   `com.apple.fairplay.v1`: Apple FairPlay (common on Safari, macOS, iOS).
    The specific set of available key systems is a powerful identifier of the user's OS and browser environment.

2.  **CDM Implementation Details:** Even for a single key system like Widevine, the underlying CDM's capabilities can vary. Scripts can query for supported video/audio codecs (`videoCapabilities`, `audioCapabilities`), robustness levels (`"SW_SECURE_CRYPTO"`, `"HW_SECURE_DECODE"`), and persistence support (`persistentState`). The exact configuration returned by `MediaKeySystemAccess.getConfiguration()` provides a detailed fingerprint of the specific CDM version installed.

3.  **Bot Detection & Behavioral Anomalies:**
    *   **Absence of DRM:** Most headless browsers (e.g., standard Puppeteer/Playwright) do not include proprietary CDMs. The complete inability to create any `MediaKeySession` is a very strong indicator of a bot.
    *   **Inconsistent Behavior:** Anti-fingerprinting tools may try to spoof the EME API, for instance, by overriding `navigator.requestMediaKeySystemAccess` to report fake supported systems. A detection script can verify this by attempting to actually create a `MediaKeySession`. If the initial query succeeds but session creation fails with an unexpected error, it reveals the spoofing attempt.
    *   **Error Messages:** The specific error messages or codes thrown during a failed session creation can leak information about the underlying environment, especially if they don't match those of a standard browser.

## Sample Code
This snippet demonstrates how a script would probe for supported DRM key systems to fingerprint the browser environment.

```javascript
async function getSupportedDrmSystems() {
  const keySystems = [
    'com.widevine.alpha',
    'com.microsoft.playready',
    'com.apple.fairplay.v1',
    'org.w3.clearkey' // Standard, but less identifying
  ];

  const supportedSystems = {};

  for (const keySystem of keySystems) {
    try {
      // We only need to check for basic video capabilities.
      // The goal is not to play media, but to see if the system is recognized.
      const config = [{
        initDataTypes: ['cenc'],
        videoCapabilities: [{
          contentType: 'video/mp4; codecs="avc1.42E01E"'
        }]
      }];

      // requestMediaKeySystemAccess is the entry point for the check.
      const keySystemAccess = await navigator.requestMediaKeySystemAccess(keySystem, config);
      
      // If the promise resolves, the system is supported.
      // The configuration details can be used for deeper fingerprinting.
      supportedSystems[keySystem] = keySystemAccess.getConfiguration();

    } catch (error) {
      // If the promise rejects, the system is not supported.
      supportedSystems[keySystem] = 'unsupported';
    }
  }

  // The resulting object is a fingerprint of the browser's DRM capabilities.
  // e.g., { "com.widevine.alpha": { ...config }, "com.microsoft.playready": "unsupported", ... }
  console.log(supportedSystems);
  return supportedSystems;
}

getSupportedDrmSystems();
```