---
title: window.RTCEncodedVideoFrame
layout: default
---
# `window.RTCEncodedVideoFrame`
## Purpose
Represents a single encoded video frame within a WebRTC Insertable Streams context. It allows developers to access and manipulate the raw encoded video data (e.g., H.264, VP9, AV1) from a WebRTC stream before it is sent over the network or after it is received.

## Fingerprinting Usage
The `RTCEncodedVideoFrame` API, part of WebRTC Insertable Streams, is a powerful surface for fingerprinting and bot detection. It moves beyond simple API presence checks into deep hardware and software stack analysis.

1.  **Hardware Acceleration & Performance Probing:** The primary use is to measure the performance of the video encoding pipeline. A script can create a local `RTCPeerConnection`, stream a `<canvas>` element, and intercept the encoded frames.
    *   **Entropy Source:** The time it takes to encode a frame, the consistency (jitter) of frame delivery, and the maximum achievable throughput are highly dependent on the underlying hardware (CPU, GPU) and whether hardware acceleration is enabled and functioning correctly.
    *   **Detection Signal:** Bots running in virtualized or headless environments often lack GPU hardware acceleration, relying on slower, software-based encoders (e.g., libvpx). This results in significantly different performance profiles:
        *   **Real Users:** Exhibit variable but fast encoding times, influenced by GPU model, driver versions, and system load. Timings will have natural, minor fluctuations.
        *   **Bots:** Often show much slower and/or unnaturally consistent encoding times, characteristic of a software-only implementation running on a server CPU. They may also fail to initialize an encoder for certain codecs (like H.264) that often rely on hardware or licensed libraries.

2.  **Encoder Fingerprinting:** The exact byte output of a video encoder for a given input can vary.
    *   **Entropy Source:** Different operating systems (Windows, macOS, Linux), browser versions, and hardware (NVIDIA, AMD, Intel, Apple Silicon) use different underlying video encoding libraries and hardware blocks (e.g., NVENC, VideoToolbox, VA-API).
    *   **Detection Signal:** A script can draw a specific, complex pattern to a canvas and analyze the resulting `RTCEncodedVideoFrame`'s `data` buffer. The size of the frame, the `type` (`key` or `delta`), and even a hash of the data itself can create a detailed fingerprint of the user's encoding stack. A bot using a generic software encoder will produce a different output than a real user's hardware-accelerated encoder.

3.  **Behavioral Analysis:** The stability of the encoding stream under stress provides behavioral clues.
    *   **Detection Signal:** By running a computationally intensive task (e.g., a complex calculation) on the main thread, a fingerprinting script can observe its impact on the video encoding stream. On a real user's machine, this will likely cause dropped frames and increased latency. The specific pattern of this degradation can be a behavioral fingerprint, differing from how a powerful server running a bot would handle the concurrent load.

## Sample Code
This snippet demonstrates how to intercept encoded frames from a canvas stream to analyze their timing and size, which are key indicators of the underlying hardware and software stack.

```javascript
async function analyzeEncoder() {
  try {
    if (!window.RTCPeerConnection || !window.RTCRtpSender.prototype.createEncodedStreams) {
      console.log('Insertable Streams API not supported.');
      return;
    }

    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = 'blue';
    ctx.fillRect(0, 0, 100, 100);
    const stream = canvas.captureStream(30);
    const [videoTrack] = stream.getVideoTracks();

    const pc = new RTCPeerConnection();
    const sender = pc.addTrack(videoTrack, stream);
    
    // Access the encoded stream
    const encodedStreams = sender.createEncodedStreams();
    const readableStream = encodedStreams.readable;
    
    let frameCount = 0;
    let lastTimestamp = 0;
    const timings = [];
    const sizes = [];

    const reader = readableStream.getReader();
    while (frameCount < 50) {
      const { value: chunk, done } = await reader.read();
      if (done) break;

      const currentTime = performance.now();
      
      // 1. Measure timing jitter and throughput between frames.
      // Bots on headless servers often have slow but very consistent timings.
      if (lastTimestamp > 0) {
        const delta = currentTime - lastTimestamp;
        timings.push(delta);
      }
      
      // 2. Analyze frame size for a given input.
      // Hardware vs. software encoders produce different sized frames.
      sizes.push(chunk.data.byteLength);
      
      lastTimestamp = currentTime;
      frameCount++;
    }
    
    // This data is a rich fingerprint of the hardware/software encoder.
    console.log('Frame Timings (ms):', timings);
    console.log('Frame Sizes (bytes):', sizes);
    
    // Cleanup
    videoTrack.stop();
    pc.close();

  } catch (e) {
    console.error('Encoder analysis failed:', e);
  }
}

// Run the analysis
analyzeEncoder();
```