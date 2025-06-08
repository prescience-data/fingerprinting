---
title: window.speechSynthesis
layout: default
---
# `window.SpeechSynthesis`
## Purpose
The `SpeechSynthesis` interface of the Web Speech API is the controller for text-to-speech (TTS) functionality. It allows web applications to generate synthesized speech from text content, using voices available on the user's operating system or browser.

## Fingerprinting Usage
The `SpeechSynthesis` API is a high-entropy source for fingerprinting due to its reliance on the host operating system's configuration.

1.  **Voice List (`getVoices()`):** This is the primary vector. The list of available `SpeechSynthesisVoice` objects returned by `speechSynthesis.getVoices()` is highly specific to the user's environment.
    *   **OS and Version:** The set of default voices differs significantly between Windows, macOS, Linux, Android, and iOS. For example, macOS includes voices like "Alex" and "Samantha," while Windows provides "Microsoft David" and "Microsoft Zira."
    *   **Language Packs:** Users who have installed additional language packs or accessibility features will have a larger and more diverse list of voices.
    *   **Browser-Specific Voices:** Some browsers, like Chrome, may bundle their own voices (e.g., "Google US English"), which can vary by browser version.
    *   **Voice Properties:** Each voice object contains properties like `name`, `lang`, `voiceURI`, and `localService` (boolean). The combination of these properties across the entire list creates a highly unique signature.

2.  **Asynchronous Behavior and Implementation Quirks:**
    *   The voice list is populated asynchronously. A script must wait for the `onvoiceschanged` event to fire before `getVoices()` returns a complete list. Naive bots or automation frameworks that call `getVoices()` synchronously will often receive an empty array `[]`, immediately flagging them as non-standard clients.
    *   Headless browsers may provide a stubbed or non-functional implementation. For instance, a headless Chrome instance running on a server without audio hardware might return an empty voice list or fail to produce audio, which can be detected by checking properties like `speechSynthesis.speaking` after calling `speak()`. This tests the fidelity of the browser environment.

## Sample Code
This snippet demonstrates how to robustly collect the list of speech synthesis voices, handle the asynchronous loading, and generate a fingerprint from the results.

```javascript
async function getSpeechSynthesisFingerprint() {
  // The getVoices() list is populated asynchronously. We must wait for the
  // 'voiceschanged' event. This is a common bot detection check.
  const getVoices = () => {
    return new Promise((resolve, reject) => {
      let voices = window.speechSynthesis.getVoices();
      if (voices.length > 0) {
        return resolve(voices);
      }
      window.speechSynthesis.onvoiceschanged = () => {
        voices = window.speechSynthesis.getVoices();
        resolve(voices);
      };
      // Timeout to prevent hanging on non-compliant browsers
      setTimeout(() => reject('SpeechSynthesis voices timed out'), 1000);
    });
  };

  try {
    const voices = await getVoices();
    // Extract key properties from each voice object
    const voiceAttributes = voices.map(voice => `${voice.name}:${voice.lang}:${voice.localService}`);
    
    // Join the attributes into a single string for hashing
    const signature = voiceAttributes.join(',');
    
    // In a real library, this signature would be hashed (e.g., with MurmurHash3)
    // For demonstration, we'll just log it.
    console.log('Speech Synthesis Signature:', signature);
    return signature;
    
  } catch (error) {
    // An error or timeout is also a fingerprinting signal (e.g., API disabled or stubbed)
    console.error('Speech Synthesis Fingerprint Error:', error);
    return 'error';
  }
}

// Example usage:
getSpeechSynthesisFingerprint();
// Example Output on macOS: "Alex:en-US:true,Alice:it-IT:true,Alva:sv-SE:true..."
// Example Output on Windows: "Microsoft David Desktop - English (United States):en-US:true,Microsoft Zira Desktop - English (United States):en-US:true..."
// Example Output on a naive bot: "error" or ""
```