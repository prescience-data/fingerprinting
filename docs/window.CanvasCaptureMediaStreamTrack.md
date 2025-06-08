---
title: window.CanvasCaptureMediaStreamTrack
layout: default
---
# window.CanvasCaptureMediaStreamTrack
## Purpose
`CanvasCaptureMediaStreamTrack` is an interface representing a real-time video track whose content is sourced from an HTML `<canvas>` element. It is created by calling the `HTMLCanvasElement.captureStream()` method.

## Fingerprinting Usage
The primary value of this API in fingerprinting is not the interface itself, but the process of creating and consuming it, which provides a high-entropy surface and reveals behavioral anomalies.

1.  **Canvas Rendering Fingerprint**: This is the most potent use. The API provides a direct method to capture the pixel output of a canvas rendering operation. By drawing a specific scene (text with unique fonts, WebGL shapes, gradients), a script can capture a frame from the stream and hash its pixel data. This hash is a highly stable and unique identifier dependent on the user's GPU, graphics driver, OS-level font rendering, and anti-aliasing settings. Headless browsers like Puppeteer often use a software renderer (e.g., SwiftShader) by default, producing a completely different and easily identifiable hash compared to a standard browser using hardware acceleration.

2.  **Performance and Behavioral Analysis**: The performance of capturing the stream can be a behavioral signal.
    *   **Frame Rate Consistency**: A script can request a specific frame rate (e.g., `canvas.captureStream(60)`). By monitoring the actual output, it can detect discrepancies. A real browser on a loaded system might drop frames, while a bot environment might deliver them perfectly or fail in a predictable way.
    *   **`requestFrame()` Latency**: The `requestFrame()` method on the track forces a new frame to be captured. Measuring the time it takes to receive this frame can be a performance metric, differentiating between software and hardware rendering or exposing timing artifacts in virtualized environments.

3.  **Spoofing Detection**: Bots attempting to defeat canvas fingerprinting often override `canvas.toDataURL()` or `context.getImageData()` to return noisy or randomized data. `CanvasCaptureMediaStreamTrack` provides an alternative, less commonly spoofed pathway to the rendered pixels.
    *   **Deterministic Rendering Check**: A key anti-bot technique is to render the same canvas scene twice and capture a frame from the stream each time. On a real user's machine, the output is deterministic; the hashes will match. A bot injecting random noise to defeat fingerprinting will produce two different hashes, revealing itself.
    *   **API Integrity**: Scripts can check if `canvas.captureStream.toString()` returns `"[native code]"` or if the returned track is a genuine `instanceof window.CanvasCaptureMediaStreamTrack`, detecting crude prototype chain modifications.

## Sample Code
This snippet demonstrates capturing a canvas frame via the stream to generate a fingerprint hash. The logic implies a check for deterministic rendering to detect spoofing.

```javascript
async function getCanvasStreamFingerprint() {
  try {
    if (!('CanvasCaptureMediaStreamTrack' in window)) {
      return 'API not supported';
    }

    const canvas = document.createElement('canvas');
    canvas.width = 200;
    canvas.height = 50;
    const ctx = canvas.getContext('2d');

    // Draw a scene with high entropy
    ctx.fillStyle = 'rgb(123, 45, 67)';
    ctx.fillRect(10, 10, 180, 30);
    ctx.fillStyle = 'rgb(255, 255, 255)';
    ctx.font = '16px "Arial"';
    ctx.fillText('Browser Fingerprint Test! @#$', 15, 30);

    // Capture the stream and a single frame
    const stream = canvas.captureStream(0); // 0 fps means frames are captured on demand
    const track = stream.getVideoTracks()[0];
    
    // Use ImageCapture to get a bitmap from the track
    const imageCapture = new ImageCapture(track);
    const bitmap = await imageCapture.grabFrame();
    track.stop(); // Clean up the stream

    // Draw bitmap to a temporary canvas to get data URL for hashing
    const tempCanvas = document.createElement('canvas');
    tempCanvas.width = bitmap.width;
    tempCanvas.height = bitmap.height;
    tempCanvas.getContext('2d').drawImage(bitmap, 0, 0);
    
    const dataUrl = tempCanvas.toDataURL();

    // In a real scenario, you would hash this dataUrl.
    // To detect spoofing, you would run this function twice and
    // ensure the resulting hashes are identical.
    // For brevity, we return a slice of the data URL.
    return dataUrl.slice(0, 100);

  } catch (error) {
    return `Error: ${error.name}`;
  }
}

// Usage:
// getCanvasStreamFingerprint().then(fp => console.log(fp));
```