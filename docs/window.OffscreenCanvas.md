---
title: window.OffscreenCanvas
layout: default
---
# window.OffscreenCanvas
## Purpose
The `OffscreenCanvas` API provides a canvas that can be rendered off-screen, decoupled from the DOM. Its primary purpose is to move graphics rendering into a Web Worker, preventing UI blocking and improving performance for complex animations or visualizations.

## Fingerprinting Usage
`OffscreenCanvas` is a powerful tool for fingerprinting and bot detection because it probes the browser's rendering pipeline, hardware configuration, and concurrency model in a context separate from the main thread.

1.  **Canvas/WebGL Fingerprinting in a Worker:** It serves as a new surface for classic canvas fingerprinting. By rendering text, shapes, and WebGL scenes in a worker thread, a script can generate a pixel buffer. The hash of this buffer is a high-entropy fingerprint derived from the GPU, graphics driver, and font rendering engine. This is especially effective for detecting headless browsers (like Puppeteer/Playwright) that may use a software renderer (e.g., SwiftShader) instead of a real GPU, producing a different visual output and a different hash.

2.  **Hardware & Driver Inconsistencies:** The rendering output of an `OffscreenCanvas` in a worker *should* be identical to an on-screen `<canvas>` on the same machine. Anti-bot systems can render the same scene on both surfaces and compare the resulting hashes. Any discrepancy is a strong signal of an anomaly, potentially indicating a spoofed environment, virtualization, or a browser with a flawed graphics stack implementation.

3.  **Performance & Concurrency Profiling:** The time it takes to create the canvas, transfer it to a worker, render a complex scene, and receive the result (`ImageBitmap`) back on the main thread is a behavioral metric. This timing is influenced by:
    *   CPU core count and thread scheduling.
    *   GPU performance and driver efficiency.
    *   Memory bandwidth for transferring the bitmap.
    A bot running in a resource-constrained container or VM will exhibit different timing characteristics than a real user's machine.

4.  **Feature Support & Quirks:** The mere existence and correct functioning of `OffscreenCanvas` is a data point.
    *   **Lack of Support:** Older browsers or non-compliant automation frameworks may not implement the API, immediately flagging them.
    *   **Implementation Errors:** A bot might polyfill the API on the `window` object but fail when a script attempts to use `transferControlToOffscreen()` or `postMessage` the canvas to a worker. These failures are clear indicators of a non-genuine environment.

## Sample Code
This snippet demonstrates generating a canvas fingerprint hash using `OffscreenCanvas` inside a Web Worker. The main thread creates the worker and canvas, while the worker performs the rendering and hashing, isolating the work from the main UI thread.

```javascript
async function getOffscreenCanvasFingerprint() {
  // 1. Check for API support. A strong negative signal if missing in a modern browser.
  if (!('OffscreenCanvas' in window)) {
    return "Error: OffscreenCanvas not supported";
  }

  try {
    // 2. Create the worker from a Blob to keep the code self-contained.
    const workerCode = `
      self.onmessage = async (e) => {
        const offscreenCanvas = e.data.canvas;
        const ctx = offscreenCanvas.getContext('2d');

        // 3. Perform standard canvas rendering operations.
        const text = "Browser Fingerprint 💯";
        ctx.textBaseline = "top";
        ctx.font = "14px 'Arial'";
        ctx.textBaseline = "alphabetic";
        ctx.fillStyle = "#f60";
        ctx.fillRect(125, 1, 62, 20);
        ctx.fillStyle = "#069";
        ctx.fillText(text, 2, 15);
        ctx.fillStyle = "rgba(102, 204, 0, 0.7)";
        ctx.fillText(text, 4, 17);

        // 4. Get the image data and hash it.
        const blob = await offscreenCanvas.convertToBlob();
        const arrayBuffer = await blob.arrayBuffer();
        const hashBuffer = await crypto.subtle.digest('SHA-256', arrayBuffer);
        
        // Convert hash to hex string
        const hashArray = Array.from(new Uint8Array(hashBuffer));
        const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
        
        // 5. Post the hash back to the main thread.
        self.postMessage(hashHex);
      };
    `;
    const workerBlob = new Blob([workerCode], { type: 'application/javascript' });
    const workerUrl = URL.createObjectURL(workerBlob);
    const worker = new Worker(workerUrl);

    // 6. Create an OffscreenCanvas and transfer control to the worker.
    const offscreen = new OffscreenCanvas(200, 50);
    worker.postMessage({ canvas: offscreen }, [offscreen]);

    // 7. Wait for the result from the worker.
    return new Promise((resolve) => {
      worker.onmessage = (e) => {
        resolve(e.data);
        worker.terminate();
        URL.revokeObjectURL(workerUrl);
      };
    });

  } catch (err) {
    return `Error: ${err.message}`;
  }
}

// Usage:
getOffscreenCanvasFingerprint().then(hash => {
  console.log("OffscreenCanvas Fingerprint:", hash);
  // Example output: "a4e1a8e2c3b4d5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0"
});
```