---
title: window.SpeechSynthesisEvent
layout: default
---
# window.SpeechSynthesisEvent
## Purpose
This API represents the event object for events dispatched by the Web Speech API's speech synthesis service. It provides information about the state of a `SpeechSynthesisUtterance` as it is being spoken, such as the character index and elapsed time.

## Fingerprinting Usage
While the `SpeechSynthesisEvent` object itself has low entropy, its generation is a critical part of a larger, high-value fingerprinting and bot detection process involving the `window.speechSynthesis` API.

1.  **Voice List Enumeration:** The primary fingerprinting vector is `speechSynthesis.getVoices()`. The list of available voices is highly specific to the user's Operating System (Windows, macOS, Linux), OS version, installed language packs, and browser. The combination of voice `name`, `lang`, and `voiceURI` properties provides a high-entropy fingerprint. Triggering speech synthesis (which generates `SpeechSynthesisEvent`) is often done to ensure the voice list is fully populated, as some browsers load it asynchronously.

2.  **Behavioral & Timing Analysis:** Bots and headless browsers often have a deficient or non-existent audio stack. Anti-bot systems exploit this by measuring the behavior of the synthesis process:
    *   **Latency:** The time between calling `speechSynthesis.speak()` and the `start` event firing can be measured. Inconsistent or zero latency is a strong anomaly signal.
    *   **Duration Mismatch:** The `elapsedTime` property of the `end` event can be compared against a wall-clock timer (`performance.now()`). In a headless environment like Puppeteer, the audio engine may not be fully implemented, causing `elapsedTime` to be `0` or wildly inaccurate, while a real browser provides a realistic duration.
    *   **Event Firing:** A spoofed or incomplete browser environment may fail to fire the `start`, `boundary`, or `end` events altogether, or fire them in an incorrect order.

3.  **Error Analysis:** Calling `speechSynthesis.speak()` in a restricted or improperly configured environment (common in bots) may immediately trigger an `error` event. The presence and type of this error (`synthesis-failed`, `audio-busy`) is a strong signal that the environment is not a standard user browser.

## Sample Code
This snippet demonstrates how to collect a fingerprint by enumerating voices and measuring the timing of the synthesis process, which relies on `SpeechSynthesisEvent`.

```javascript
function getSpeechFingerprint() {
  return new Promise((resolve) => {
    if (!window.speechSynthesis || !window.SpeechSynthesisUtterance) {
      return resolve({ error: "SpeechSynthesisNotSupported" });
    }

    const fingerprint = {
      voices: [],
      timing: {
        startTime: 0,
        endTime: 0,
        duration: 0,
        eventElapsedTime: 0,
        error: null,
      },
    };

    // 1. Get the list of available voices (high-entropy source)
    const voices = window.speechSynthesis.getVoices();
    if (voices.length > 0) {
        fingerprint.voices = voices.map(v => `${v.name} (${v.lang})`);
    } else {
        // On some browsers, voices load async. We'll get them in the utterance.
        // A persistently empty list is a red flag.
    }

    const utterance = new SpeechSynthesisUtterance("test");
    const startTime = performance.now();

    utterance.onstart = (event) => {
      fingerprint.timing.startTime = performance.now() - startTime;
    };

    utterance.onerror = (event) => {
      fingerprint.timing.error = event.error;
      resolve(fingerprint); // Resolve immediately on error
    };

    utterance.onend = (event) => {
      fingerprint.timing.endTime = performance.now();
      fingerprint.timing.duration = fingerprint.timing.endTime - (startTime + fingerprint.timing.startTime);
      // 2. Capture elapsedTime from the event - a key behavioral signal
      fingerprint.timing.eventElapsedTime = event.elapsedTime;

      // Re-check voices in case they loaded asynchronously
      fingerprint.voices = window.speechSynthesis.getVoices().map(v => `${v.name} (${v.lang})`);
      
      resolve(fingerprint);
    };

    // 3. Trigger the process that generates SpeechSynthesisEvents
    window.speechSynthesis.speak(utterance);

    // Timeout to handle cases where no events fire (common bot behavior)
    setTimeout(() => {
        if (fingerprint.timing.endTime === 0) {
            fingerprint.timing.error = "timeout";
            resolve(fingerprint);
        }
    }, 1000);
  });
}

// Usage:
// getSpeechFingerprint().then(fp => console.log(fp));
```