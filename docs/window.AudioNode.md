---
title: window.AudioNode
layout: default
---
# window.AudioNode
## Purpose
`AudioNode` is the base interface for any audio processing module within the Web Audio API. It represents a single node in an audio processing graph, such as an audio source (e.g., `OscillatorNode`), a modification effect (e.g., `DynamicsCompressorNode`), or an audio destination (e.g., `AudioDestinationNode`).

## Fingerprinting Usage
`AudioNode` is a cornerstone of the highly effective **Audio Fingerprinting** (or AudioContext Fingerprinting) technique. This method does not involve playing or recording sound but instead measures the unique way a user's hardware and software stack processes a generated audio signal.

The core principle is that floating-point calculations performed by audio processing nodes can have minute, yet consistent, variations across different combinations of:
*   **Hardware:** CPU architecture, integrated sound card.
*   **Software:** Operating system, browser version, and specific audio drivers.

The process involves:
1.  Creating an `OfflineAudioContext` to process audio silently and quickly in the background.
2.  Generating a known, consistent signal using an `OscillatorNode`.
3.  Passing this signal through one or more complex processing nodes, most commonly a `DynamicsCompressorNode`, whose algorithm is sensitive to subtle computational differences.
4.  Rendering the output to an `AudioBuffer` and extracting a metric from the resulting samples (e.g., the sum of all sample values).
5.  Hashing this metric to produce a stable, high-entropy fingerprint.

**Anomalies and Bot Detection:**
*   **Missing Implementation:** Many headless browsers (like older versions of Puppeteer or Playwright) or minimal environments may not have a complete or functional implementation of the Web Audio API. A script failing to create an `AudioContext` or its nodes is a strong bot signal.
*   **Inconsistent Fingerprints:** A bot might spoof its User-Agent string to appear as "Chrome on Windows," but its audio fingerprint may match a known value for "Headless Chrome on Linux," revealing the deception.
*   **Null or Zero Output:** Some unsophisticated bots or browser automation frameworks may return `0` or `null` for the audio buffer data to avoid the computational cost, which is a trivial anomaly to detect.

## Sample Code
This snippet demonstrates the core logic of generating an audio fingerprint by processing a signal through a compressor.

```javascript
async function getAudioFingerprint() {
  try {
    // Use OfflineAudioContext for silent, fast rendering.
    // Fixed parameters are crucial for a consistent fingerprint.
    const context = new (window.OfflineAudioContext || window.webkitOfflineAudioContext)(1, 44100, 44100);

    // Create a simple oscillator to generate a sine wave.
    const oscillator = context.createOscillator();
    oscillator.type = 'triangle';
    oscillator.frequency.setValueAtTime(10000, context.currentTime);

    // Use a DynamicsCompressorNode, as its processing is sensitive to hardware/software differences.
    const compressor = context.createDynamicsCompressor();
    compressor.threshold.setValueAtTime(-50, context.currentTime);
    compressor.knee.setValueAtTime(40, context.currentTime);
    compressor.ratio.setValueAtTime(12, context.currentTime);
    compressor.attack.setValueAtTime(0, context.currentTime);
    compressor.release.setValueAtTime(0.25, context.currentTime);

    // Build the audio graph: oscillator -> compressor -> destination
    oscillator.connect(compressor);
    compressor.connect(context.destination);

    // Start the oscillator and render the audio.
    oscillator.start(0);
    const renderedBuffer = await context.startRendering();

    // Extract the raw sample data from the buffer.
    const channelData = renderedBuffer.getChannelData(0);

    // Calculate a sum of the samples as a simple metric.
    // In a real-world scenario, this would be hashed (e.g., SHA-256).
    let sum = 0;
    for (let i = 0; i < channelData.length; i++) {
      sum += Math.abs(channelData[i]);
    }

    return sum.toString();
  } catch (error) {
    // If the API is not supported or fails, it's a strong bot signal.
    return 'AudioAPI_Not_Supported';
  }
}

// Usage:
getAudioFingerprint().then(fingerprint => {
  console.log('Audio Fingerprint:', fingerprint);
});
```