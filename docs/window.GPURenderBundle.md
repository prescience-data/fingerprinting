---
title: window.GPURenderBundle
layout: default
---
# window.GPURenderBundle
## Purpose
`GPURenderBundle` is an interface within the WebGPU API. It represents a pre-recorded, reusable sequence of rendering commands that can be executed efficiently multiple times, reducing CPU overhead by avoiding redundant command encoding.

## Fingerprinting Usage
`GPURenderBundle` and its associated encoder (`GPURenderBundleEncoder`) are powerful tools for fingerprinting and bot detection, primarily through performance benchmarking and error analysis. As WebGPU interacts directly with the system's GPU driver, its behavior exposes significant details about the underlying hardware and software stack.

1.  **Existence and Availability**: The mere presence of `window.GPURenderBundle` and the parent `navigator.gpu` object signals a modern, capable browser. Many headless browsers or older browser versions lack WebGPU support, making its absence a strong indicator of a non-standard or outdated environment.

2.  **Performance Benchmarking (Timing Attack)**: The time required to create a render bundle is a high-entropy fingerprinting vector. The process involves the browser's WebGPU implementation (e.g., Chrome's Dawn) translating commands and the GPU driver compiling them into a native command list. This duration is highly sensitive to:
    *   **GPU Hardware**: Different GPU vendors (NVIDIA, AMD, Intel, Apple) and models have distinct performance profiles.
    *   **GPU Driver**: The driver version and its specific optimizations directly impact compilation time.
    *   **Operating System**: The OS's graphics subsystem (e.g., DirectX 12 on Windows, Metal on macOS, Vulkan on Linux) influences performance.
    *   **Browser Implementation**: Subtle differences between browser versions affect the process.
    By measuring the time to encode a standardized, complex set of rendering commands, a script can generate a stable and unique hardware identifier.

3.  **Error Message Analysis**: Intentionally creating a `GPURenderBundle` with invalid parameters (e.g., incorrect buffer bindings, invalid pipeline states) will trigger a `GPUValidationError`. The exact text and structure of the resulting error message can vary significantly between different GPU drivers, operating systems, and browser versions. This provides a fingerprint similar to well-known WebGL error fingerprinting techniques.

4.  **Behavioral Analysis**: For bot detection, scripts can repeatedly create render bundles and analyze the timing consistency. Virtualized environments or overloaded cloud servers often exhibit greater performance jitter than a typical user's machine running on bare metal. An unusually high standard deviation in encoding times can be a flag for a bot.

## Sample Code
This snippet demonstrates a timing attack by measuring the duration of `encoder.finish()`, which creates the `GPURenderBundle`. The resulting `duration` is a potent fingerprinting value.

```javascript
async function getRenderBundleFingerprint() {
  try {
    if (!navigator.gpu) {
      return "WebGPU not supported";
    }

    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return "No GPU adapter found";
    }
    const device = await adapter.requestDevice();

    // The format here is arbitrary for the test, but consistency is key.
    const bundleEncoder = device.createRenderBundleEncoder({
      colorFormats: ['bgra8unorm'],
    });

    // A series of arbitrary commands to create a workload for the driver.
    // In a real scenario, this would be more complex and standardized.
    // For this example, we'll keep it empty to measure base overhead.
    // bundleEncoder.draw(3, 1, 0, 0);

    // --- FINGERPRINTING LOGIC ---
    const startTime = performance.now();
    
    // finish() triggers the compilation of the command list by the driver.
    const renderBundle = bundleEncoder.finish(); 
    
    const endTime = performance.now();
    const duration = endTime - startTime;
    // --- END FINGERPRINTING LOGIC ---

    // The 'duration' is a high-entropy value reflecting the GPU/driver stack.
    // It can be combined with other metrics for a robust fingerprint.
    console.log(`Render bundle creation time: ${duration.toFixed(4)} ms`);
    return duration;

  } catch (error) {
    // The error message itself is a fingerprinting vector.
    return `Error: ${error.message}`;
  }
}

getRenderBundleFingerprint().then(fingerprint => {
  // This fingerprint value would be sent to a server for analysis.
  console.log("Fingerprint Value:", fingerprint);
});
```