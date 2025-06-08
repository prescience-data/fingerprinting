---
title: window.OfflineAudioCompletionEvent
layout: default
---
# window.OfflineAudioCompletionEvent
## Purpose
This event is dispatched by an `OfflineAudioContext` when its audio processing and rendering are complete. It signals that the `renderedBuffer` containing the final, processed audio data is now available for inspection or use.

## Fingerprinting Usage
The `OfflineAudioCompletionEvent` is not the source of entropy itself, but it is a critical component of the powerful and widely used **Audio Fingerprinting** technique. Anti-bot systems use this event as the trigger to analyze the rendered audio data.

1.  **Rendered Buffer Analysis:** The primary use is to signal when the audio buffer is ready. The script listens for this event, and upon its firing, it accesses the `event.renderedBuffer`. It then reads the raw floating-point sample data from the buffer. Slight variations in CPU architecture, browser rendering engine, and OS-level math libraries cause subtle differences in the final rendered samples. A hash (like MurmurHash3 or SHA-256) of this data produces a highly stable and unique identifier for the device/browser combination.

2.  **Behavioral Timing Analysis:** The time elapsed between calling `OfflineAudioContext.startRendering()` and the firing of the `OfflineAudioCompletionEvent` is a valuable behavioral signal.
    *   **Real Users:** On a real machine, the rendering process takes a small but measurable amount of time (typically a few milliseconds), depending on the complexity of the audio graph and the CPU speed.
    *   **Bots/Spoofed Environments:** Bots attempting to evade detection by hooking the API and returning a pre-calculated, "clean" value often do so synchronously or with a near-zero delay. An unnaturally fast completion time is a strong indicator of a non-human or spoofed environment.

3.  **API Integrity Check:** The presence and correct behavior of this event are used to validate the browser environment.
    *   **Missing API:** Simple headless browsers or sandboxed environments may not have a complete Web Audio API implementation. If the event never fires, it indicates a non-standard or incomplete browser.
    *   **Inconsistent Results:** If the fingerprint derived from the rendered buffer does not match the expected fingerprint for the claimed User-Agent string, it's a major red flag for spoofing.

## Sample Code
This snippet demonstrates the core logic of audio fingerprinting. The `complete` event handler is where the fingerprint is extracted and processed.

```javascript
async function getAudioFingerprint() {
  try {
    // The rendering context processes audio as fast as possible, not in real-time.
    const audioCtx = new window.OfflineAudioContext(1, 44100, 44100);

    // Create a simple oscillator and compressor to process the audio.
    // The compressor's processing is highly sensitive to internal floating-point math.
    const oscillator = audioCtx.createOscillator();
    oscillator.type = 'triangle';
    oscillator.frequency.setValueAtTime(10000, audioCtx.currentTime);

    const compressor = audioCtx.createDynamicsCompressor();
    compressor.threshold.setValueAtTime(-50, audioCtx.currentTime);
    compressor.knee.setValueAtTime(40, audioCtx.currentTime);
    compressor.ratio.setValueAtTime(12, audioCtx.currentTime);
    compressor.attack.setValueAtTime(0, audioCtx.currentTime);
    compressor.release.setValueAtTime(0.25, audioCtx.currentTime);

    oscillator.connect(compressor);
    compressor.connect(audioCtx.destination);
    oscillator.start(0);

    // Start rendering and wait for the 'complete' event.
    const renderedBuffer = await audioCtx.startRendering();

    // The OfflineAudioCompletionEvent has fired, and its 'renderedBuffer' is now available.
    // We extract the raw sample data to generate a fingerprint.
    const samples = renderedBuffer.getChannelData(0);
    let fingerprint = 0;
    for (let i = 0; i < samples.length; i++) {
      // A simple sum is used for demonstration; real-world code uses a robust hash.
      fingerprint += Math.abs(samples[i]);
    }

    console.log('Audio Fingerprint:', fingerprint);
    return fingerprint;

  } catch (error) {
    console.error('Audio fingerprinting failed:', error);
    return null; // Indicates a broken or missing API.
  }
}

getAudioFingerprint();
// Example Output on a real browser: 124.0434752702713
// This value will be consistent on the same machine/browser but different on others.
```