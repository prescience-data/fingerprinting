---
title: window.AudioContext
layout: default
---
# window.AudioContext
## Purpose
The `AudioContext` interface is the primary entry point for the Web Audio API. It is used to create, manage, and manipulate audio processing graphs for generating and processing audio in the browser.

## Fingerprinting Usage
Audio fingerprinting is a high-entropy, stable, and widely used technique. It does not involve playing audible sound to the user. Instead, it leverages the subtle differences in how various hardware and software stacks process a standardized audio signal.

The core principle is to generate a specific waveform (e.g., a sine wave from an `OscillatorNode`) and pass it through one or more processing nodes (typically a `DynamicsCompressorNode`). The final rendered audio buffer is then analyzed. Variations in the output are caused by:

*   **Hardware:** CPU architecture, integrated sound card, and other hardware components affect floating-point calculations.
*   **Software Stack:** The browser's specific implementation of audio processing algorithms (e.g., in V8 or other engines), the operating system's audio libraries, and audio drivers all introduce minor variations.

The resulting floating-point values in the output buffer are summed or hashed (commonly with MurmurHash3) to produce a single, consistent fingerprint for that specific browser and device combination.

**Bot Detection Signals:**

1.  **Absence or Errors:** Many headless browsers or automation frameworks lack a complete audio stack. Attempting to create an `AudioContext` or render an audio graph may fail, throw an error, or return a null result, which is a strong indicator of a non-standard environment.
2.  **Zeroed or Constant Output:** Some bot environments return a buffer of all zeros or a known constant value, which is easily flagged.
3.  **Inconsistency:** A real user's audio fingerprint is extremely stable. If the value changes on subsequent page loads or checks, it suggests spoofing or a misconfigured bot.
4.  **Timing Analysis:** The time required to generate the fingerprint can be measured. A real browser takes a small but non-zero amount of time (e.g., 10-50ms). A bot might return a cached or hardcoded value instantly, or take an unusually long time, both of which are anomalous.

## Sample Code
This snippet demonstrates the standard method using `OfflineAudioContext`, which processes the audio graph without audible playback and is faster.

```javascript
async function getAudioFingerprint() {
  try {
    // Use OfflineAudioContext for faster, non-audible processing
    const context = new (window.OfflineAudioContext || window.webkitOfflineAudioContext)(1, 44100, 44100);

    // Create an oscillator
    const oscillator = context.createOscillator();
    oscillator.type = 'triangle';
    oscillator.frequency.setValueAtTime(10000, context.currentTime);

    // Create a compressor to introduce processing variations
    const compressor = context.createDynamicsCompressor();
    [
      ['threshold', -50], ['knee', 40], ['ratio', 12],
      ['attack', 0], ['release', 0.25]
    ].forEach(param => {
      if (compressor[param[0]] !== undefined && typeof compressor[param[0]].setValueAtTime === 'function') {
        compressor[param[0]].setValueAtTime(param[1], context.currentTime);
      }
    });

    // Connect the nodes and start processing
    oscillator.connect(compressor);
    compressor.connect(context.destination);
    oscillator.start(0);
    const renderedBuffer = await context.startRendering();

    // Extract the raw sample data
    const samples = renderedBuffer.getChannelData(0);

    // Calculate a simple sum as the fingerprint (real-world uses a hash like Murmur3)
    let fingerprint = 0;
    for (let i = 0; i < samples.length; i++) {
      fingerprint += Math.abs(samples[i]);
    }

    return fingerprint;
  } catch (error) {
    // Errors or inability to create context is a strong bot signal
    return 'error';
  }
}

// Usage:
getAudioFingerprint().then(fp => {
  console.log('Audio Fingerprint:', fp);
});
```