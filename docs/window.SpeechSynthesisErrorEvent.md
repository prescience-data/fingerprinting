---
title: window.SpeechSynthesisErrorEvent
layout: default
---
# window.SpeechSynthesisErrorEvent
## Purpose
An event interface for the Web Speech API. It represents an error that occurs during the processing of a `SpeechSynthesisUtterance` (a text-to-speech request).

## Fingerprinting Usage
The `SpeechSynthesisErrorEvent` is primarily used in bot detection to probe for inconsistencies and incomplete implementations in headless browsers or emulated environments. Real browsers have deep integration with the underlying operating system's Text-to-Speech (TTS) engine, whereas bots often provide incomplete or mocked APIs.

1.  **Implementation Gaps:** Headless browsers like Puppeteer or Playwright, in their default configurations, often lack a functional TTS engine. Attempting to use `speechSynthesis.speak()` will not produce audio and may fail in specific, detectable ways. The script can check if an `onerror` event fires, and the `error` property of the event (`synthesis-unavailable`, `synthesis-failed`) can signal a non-standard environment. A complete lack of any event (`onend` or `onerror`) is also a strong indicator of a non-functional, stubbed API.

2.  **Spoofing Inconsistency:** A common bot evasion technique is to spoof the `speechSynthesis.getVoices()` method to return a list of fake, realistic-looking voices. A detection script can then try to use one of these supposedly "available" voices. If the attempt immediately fails and triggers a `SpeechSynthesisErrorEvent` with an error code of `voice-unavailable`, it reveals a clear inconsistency between the advertised capabilities and the actual implementation. This is a high-confidence signal of a spoofed environment.

3.  **Error Property Entropy:** For a given invalid input (e.g., an unsupported language or an extremely long text string), the specific `error` code (`language-unavailable`, `text-too-long`, etc.) and the `elapsedTime` before the error is thrown can differ between browser engines (Blink, Gecko, WebKit) and the underlying OS TTS service (Windows SAPI, macOS Speech Synthesis, etc.). This provides a small amount of entropy that can contribute to a larger device fingerprint.

## Sample Code
This snippet demonstrates how to detect spoofing by checking for an error when using a voice that the browser claims is available.

```javascript
function checkSpeechSynthesis() {
  return new Promise(resolve => {
    const synth = window.speechSynthesis;
    if (!synth || typeof synth.getVoices !== 'function' || typeof synth.speak !== 'function') {
      return resolve('API_MISSING');
    }

    // getVoices() can be async, onvoiceschanged is the robust way to wait.
    let voices = synth.getVoices();
    if (voices.length === 0) {
      synth.onvoiceschanged = () => {
        voices = synth.getVoices();
        if (voices.length > 0) {
          runTest(voices);
        } else {
          resolve('NO_VOICES_AVAILABLE');
        }
      };
    } else {
      runTest(voices);
    }

    // Set a timeout in case onvoiceschanged never fires
    setTimeout(() => resolve('TIMEOUT'), 1000);

    function runTest(voices) {
      const utterance = new SpeechSynthesisUtterance('test');
      // Use a voice the browser claims to have
      utterance.voice = voices[0];

      utterance.onerror = (event) => {
        // This is the key detection point.
        // If a listed voice is "unavailable", it's a strong bot signal.
        if (event.error === 'voice-unavailable') {
          resolve('SPOOFED_VOICES');
        } else {
          resolve(`ERROR: ${event.error}`);
        }
      };

      utterance.onend = () => {
        resolve('SUCCESS');
      };

      synth.speak(utterance);
    }
  });
}

// Usage:
checkSpeechSynthesis().then(result => {
  console.log('Speech Synthesis Test Result:', result);
  // An anti-bot system would flag 'SPOOFED_VOICES' with a high score.
});
```