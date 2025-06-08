---
title: window.SpeechSynthesisUtterance
layout: default
---
# window.SpeechSynthesisUtterance
## Purpose
`SpeechSynthesisUtterance` is an interface of the Web Speech API. It represents a speech request, containing the text to be synthesized and configuration parameters like the voice, pitch, rate, and volume.

## Fingerprinting Usage
While `SpeechSynthesisUtterance` itself is just a configuration object, its use in conjunction with `speechSynthesis.getVoices()` provides a high-entropy fingerprinting vector. Anti-bot systems leverage this to profile the user's environment and detect inconsistencies common in automated clients.

1.  **Voice List Enumeration**: The primary vector is the list of available TTS voices returned by `speechSynthesis.getVoices()`.
    *   **Entropy Source**: The list of voices is highly dependent on the Operating System (Windows, macOS, Linux, Android), the OS version, and any user-installed language packs or third-party TTS engines. The combination of voice `name`, `lang`, and `voiceURI` properties for all available voices creates a unique signature.
    *   **Platform Detection**: The default voice sets are distinct across platforms (e.g., "Microsoft David" on Windows, "Alex" or "Siri" on macOS, "Google US English" on Android), making this a reliable OS identifier.

2.  **Headless Browser Detection**: This is a classic and effective bot detection method.
    *   **Empty Voice List**: By default, many headless browsers (like Puppeteer or Playwright) run in a minimal environment and report an empty array for `getVoices()`. A real user's browser will almost always have at least one voice.
    *   **API Inconsistencies**: A bot may try to spoof the `getVoices()` method to return a fake list of voices. However, attempting to actually use one of these fake voices via `speechSynthesis.speak(new SpeechSynthesisUtterance())` will fail or behave unnaturally. For instance, the `onstart` and `onend` events might not fire, or they might fire synchronously, which is anomalous for a real TTS engine.

3.  **Asynchronous Behavior**: The `getVoices()` list is populated asynchronously. It is often empty on the initial call and filled after the `onvoiceschanged` event fires. Fingerprinting scripts can probe this behavior. Naive bots might return a static, synchronous list, which is a detectable deviation from standard browser behavior.

## Sample Code
This snippet demonstrates how to collect a fingerprint from the available speech synthesis voices, correctly handling the asynchronous nature of the API.

```javascript
/**
 * Generates a fingerprint based on the available speech synthesis voices.
 * Handles the asynchronous loading of the voice list.
 * @returns {Promise<string>} A promise that resolves to a string representing the voice fingerprint.
 */
function getSpeechSynthesisFingerprint() {
  return new Promise((resolve) => {
    const synth = window.speechSynthesis;
    if (!synth || typeof synth.getVoices !== 'function') {
      return resolve('api_not_supported');
    }

    let resolved = false;

    const getVoicesAndResolve = () => {
      if (resolved) return;
      const voices = synth.getVoices();

      if (voices.length > 0) {
        const voiceDetails = voices
          .map(voice => `${voice.name}:${voice.lang}:${voice.localService}`)
          .sort()
          .join(';');
        
        // In a real-world scenario, this string would be hashed (e.g., with SHA-256)
        // to produce a stable, fixed-length identifier.
        resolve(voiceDetails);
        resolved = true;
      }
    };

    // The 'onvoiceschanged' event fires when the voice list is ready.
    synth.onvoiceschanged = getVoicesAndResolve;

    // Attempt to get voices immediately in case they are already loaded.
    getVoicesAndResolve();

    // Set a timeout as a fallback for browsers that may not fire the event reliably.
    setTimeout(() => {
      if (!resolved) {
        getVoicesAndResolve(); // One last try
        if (!resolved) {
          resolve('no_voices_found_or_timeout');
        }
      }
    }, 500);
  });
}

// Usage:
getSpeechSynthesisFingerprint().then(fingerprint => {
  console.log('Speech Synthesis Fingerprint:', fingerprint);
  // Example Output (macOS): "Alex:en-US:true;Siri:en-US:true;..."
  // Example Output (Headless Chrome): "no_voices_found_or_timeout"
});
```