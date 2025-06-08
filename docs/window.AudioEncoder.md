---
title: window.AudioEncoder
layout: default
---
# window.AudioEncoder
## Purpose
Part of the WebCodecs API, `AudioEncoder` provides a low-level interface for encoding raw `AudioData` frames into a compressed format (e.g., Opus, AAC). It is designed for high-performance applications like real-time communication, cloud gaming, and in-browser video/audio editing.

## Fingerprinting Usage
`AudioEncoder` is a potent source of fingerprinting entropy due to its reliance on underlying OS and hardware capabilities. Anti-bot systems use it to verify the authenticity of the browser environment.

1.  **Codec & Configuration Support:** The primary fingerprinting vector is enumerating which codec configurations are supported. The static method `AudioEncoder.isConfigSupported(config)` can be queried with numerous combinations of codecs (`opus`, `aac`, `flac`, etc.), sample rates, bitrates, and channel counts. The resulting pattern of supported vs. unsupported configurations is highly specific to:
    *   **Browser & Version:** Chrome may add or deprecate support for codecs or specific profiles in different versions.
    *   **Operating System:** The underlying OS media frameworks (e.g., Media Foundation on Windows, Core Audio on macOS) provide the actual encoder libraries. The available set of encoders can differ significantly between Windows, macOS, Linux, and Android.
    *   **Hardware:** Some codecs may only be available or performant if hardware acceleration is present and enabled. A headless browser running in a datacenter VM will likely lack the hardware-accelerated H.264/AAC encoders common on consumer devices.
    *   **Build Flags:** Custom Chromium builds (often used in automation frameworks) may be compiled with a different set of codecs than official Google Chrome builds.

2.  **Implementation-Specific Errors:** Attempting to configure an encoder with invalid, unsupported, or edge-case parameters can trigger errors. The exact error message string and `DOMException` type are often specific to the V8/Blink implementation version. Bots attempting to spoof this API must replicate these error details perfectly, which is difficult.

3.  **Behavioral & Performance Analysis:**
    *   **Encoding Speed:** A script can generate a standardized raw audio buffer and measure the time it takes for the encoder to process it. This timing is a strong behavioral signal, sensitive to the underlying CPU performance and the presence of hardware acceleration. Bots running on shared, underpowered server CPUs will exhibit significantly different performance profiles than a real user on a modern laptop or desktop.
    *   **Encoded Output:** For a given input `AudioData` and configuration, the exact `byteLength` of the output `EncodedAudioChunk` can vary slightly between different encoder versions or implementations. Hashing the byte length of the first few encoded chunks can create a stable and high-entropy fingerprint component.

## Sample Code
This snippet demonstrates the primary fingerprinting technique: enumerating supported codec configurations to generate a unique identifier.

```javascript
async function getAudioEncoderFingerprint() {
  if (!window.AudioEncoder) {
    return 'NotSupported';
  }

  const configsToTest = [
    // Opus configs
    { codec: 'opus', sampleRate: 48000, numberOfChannels: 2, bitrate: 128000 },
    { codec: 'opus', sampleRate: 24000, numberOfChannels: 1, bitrate: 64000 },
    // AAC configs (often hardware-dependent)
    { codec: 'mp4a.40.2', sampleRate: 44100, numberOfChannels: 2, bitrate: 192000 },
    { codec: 'mp4a.40.5', sampleRate: 22050, numberOfChannels: 1, bitrate: 96000 },
    // FLAC config
    { codec: 'flac', sampleRate: 48000, numberOfChannels: 2, bitDepth: 16 },
  ];

  try {
    const supportResults = await Promise.all(
      configsToTest.map(config => AudioEncoder.isConfigSupported(config))
    );

    // The array of boolean/object results creates a unique signature.
    // In a real scenario, this string would be hashed (e.g., with SHA-256).
    const fingerprint = supportResults.map(res => res.supported ? '1' : '0').join('');
    
    return fingerprint;
  } catch (e) {
    // An unexpected error during probing is also a signal.
    return 'ProbingError';
  }
}

getAudioEncoderFingerprint().then(fp => {
  console.log('AudioEncoder Fingerprint:', fp);
  // Example Output: "11001"
});
```