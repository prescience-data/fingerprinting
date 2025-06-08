---
title: window.AudioBuffer
layout: default
---
# window.AudioBuffer
## Purpose
Represents a short audio asset in memory, containing Pulse-Code Modulation (PCM) data organized by channels. It is a core component of the Web Audio API, used for storing, manipulating, and playing audio data generated or decoded by an `AudioContext`.

## Fingerprinting Usage
The `AudioBuffer` itself is not the direct source of entropy. Instead, it is the container for the *result* of a process known as **AudioContext Fingerprinting**, a highly effective and stable fingerprinting technique.

The method involves using an `OfflineAudioContext` to render a specific audio signal through a series of processing nodes (typically an `OscillatorNode` and a `DynamicsCompressorNode`). The final rendered PCM data, stored in an `AudioBuffer`, is then analyzed.

Key sources of entropy include:
1.  **Hardware and OS Variations:** Subtle differences in floating-point unit (FPU) architecture on the CPU, as well as OS-level audio libraries and drivers, cause minute variations in the output of audio processing algorithms.
2.  **Browser Implementation:** The specific implementation of the audio processing algorithms (especially the complex `DynamicsCompressorNode`) differs between browser vendors (Blink, Gecko, WebKit) and even between different versions of the same browser.
3.  **Anomalies in Headless Browsers:**
    *   **Lack of Implementation:** Older or simpler headless browsers may not have implemented the Web Audio API, causing the context creation to fail.
    *   **Standardized Output:** Modern headless browsers (e.g., headless Chrome) often produce a standardized, non-unique audio fingerprint, which can be flagged if it appears too frequently.
    *   **Inconsistency:** A bot might spoof its User-Agent to appear as Chrome on Windows but produce an audio fingerprint characteristic of headless Linux, creating a detectable mismatch.

The fingerprint is typically derived by calculating a sum or a hash (e.g., SHA-256) of the floating-point values within the final `AudioBuffer`. This produces a highly stable and unique identifier for the user's device/software stack.

## Sample Code
This snippet demonstrates the core logic of AudioContext fingerprinting. It uses an `OfflineAudioContext` to render a simple waveform and then calculates a sum from the resulting `AudioBuffer` data.

```javascript
async function getAudioFingerprint() {
  try {
    // Use OfflineAudioContext to render audio without playing it.
    // Parameters are chosen to be common in fingerprinting scripts.
    const audioCtx = new (window.OfflineAudioContext || window.webkitOfflineAudioContext)(1, 44100, 44100);

    // Create a simple oscillator
    const oscillator = audioCtx.createOscillator();
    oscillator.type = 'triangle';
    oscillator.frequency.setValueAtTime(10000, audioCtx.currentTime);

    // Use a DynamicsCompressorNode, a major source of entropy
    const compressor = audioCtx.createDynamicsCompressor();
    compressor.threshold.setValueAtTime(-50, audioCtx.currentTime);
    compressor.knee.setValueAtTime(40, audioCtx.currentTime);
    compressor.ratio.setValueAtTime(12, audioCtx.currentTime);
    compressor.attack.setValueAtTime(0, audioCtx.currentTime);
    compressor.release.setValueAtTime(0.25, audioCtx.currentTime);

    // Connect the nodes and start rendering
    oscillator.connect(compressor);
    compressor.connect(audioCtx.destination);
    oscillator.start(0);
    
    const renderedBuffer = await audioCtx.startRendering();

    // The renderedBuffer is the AudioBuffer instance.
    // We extract its data to generate the fingerprint.
    const pcmData = renderedBuffer.getChannelData(0);
    let fingerprint = 0;
    for (let i = 0; i < pcmData.length; i++) {
      fingerprint += Math.abs(pcmData[i]);
    }

    return fingerprint;
  } catch (error) {
    // If the API is not supported or blocked, this is also a signal.
    return 'error';
  }
}

// Usage:
getAudioFingerprint().then(fp => {
  console.log('Audio Fingerprint:', fp);
  // Example output: 124.04347527516074
});
```