---
title: window.GPUBuffer
layout: default
---
# window.GPUBuffer
## Purpose
Represents a block of memory (a buffer) accessible by the GPU. It is a core component of the WebGPU API, used for storing data for rendering (e.g., vertex data, uniforms) and general-purpose compute operations (GPGPU).

## Fingerprinting Usage
`GPUBuffer` itself is a data container, but its usage within the broader WebGPU API is a high-entropy source for fingerprinting and a strong signal for bot detection. The interaction with the GPU provides hardware-specific information that is difficult for bots to forge consistently.

1.  **Hardware & Driver Identification**: The primary fingerprinting vector is not the buffer but the process of acquiring the `GPUDevice` needed to create it. The `navigator.gpu.requestAdapter()` call returns a `GPUAdapter` object. While the `adapter.info` property is being deprecated for privacy, the adapter's capabilities, limits, and performance characteristics are still exposed. These are often cross-referenced with the `UNMASKED_RENDERER_WEBGL` string from the WebGL API to precisely identify the user's GPU model and driver stack (e.g., "NVIDIA GeForce RTX 3080" vs. "Apple M1 Pro" vs. "Mesa Intel(R) HD Graphics 620"). A mismatch or a generic renderer name is a red flag.

2.  **Performance Benchmarking**: The speed of GPU operations is a powerful behavioral fingerprint. A script can create `GPUBuffer` objects, write data to them, execute a standardized compute shader (e.g., calculating hashes, matrix multiplication), and measure the time it takes to read the result back.
    *   **Signal**: This timing is highly specific to the GPU hardware, driver version, OS, and even current system load.
    *   **Bot Detection**: Headless browsers often use a software renderer (like Google's SwiftShader) or a virtualized GPU. A software renderer will be orders of magnitude slower than native hardware, providing a clear signal that the environment is not a typical user's browser. Virtualized GPUs in data centers also have distinct and often less variable performance profiles compared to consumer hardware.

3.  **Hardware Limits**: The `GPUDevice` object, obtained before creating a buffer, has a `limits` property (`GPUSupportedLimits`). This exposes dozens of hardware-specific values like `maxBufferSize`, `maxStorageBufferBindingSize`, and `maxComputeWorkgroupsPerDimension`. The combination of these values creates a highly unique fingerprint of the underlying hardware. Forging these values is difficult because they are interdependent and must align with the performance benchmarks.

4.  **API Availability**: The mere presence of `navigator.gpu` and the successful creation of a `GPUDevice` and `GPUBuffer` can be used to segment users. Many bots or privacy-focused browser configurations may disable WebGPU, causing these calls to fail.

## Sample Code
This snippet demonstrates how to access GPU adapter information and hardware limits, and outlines a performance test—key components of a WebGPU fingerprinting script.

```javascript
async function getGpuFingerprint() {
  if (!navigator.gpu) {
    return { error: "WebGPU not supported." };
  }

  try {
    // 1. Request adapter - its existence and type is a signal
    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return { error: "No GPU adapter found." };
    }

    // 2. Request device and extract hardware limits for fingerprinting
    const device = await adapter.requestDevice();
    const limits = {};
    for (const key in device.limits) {
      limits[key] = device.limits[key];
    }

    // 3. Perform a benchmark to measure GPU compute performance
    const startTime = performance.now();

    // (Simplified for brevity)
    // In a real scenario, you would:
    // - Create a compute shader (e.g., in WGSL)
    // - Create GPUBuffer for input and output
    // - Create a GPUBindGroup and GPUComputePipeline
    // - Encode, submit, and await commands
    // - Read data back from the output buffer
    // For this example, we'll just simulate a delay.
    await new Promise(resolve => setTimeout(resolve, 50)); // Placeholder for actual GPU work

    const executionTime = performance.now() - startTime;

    return {
      // The adapter itself doesn't expose much, but its performance and limits do.
      // This is often correlated with WebGL's renderer string for a full ID.
      adapterFeatures: [...adapter.features.values()],
      deviceLimits: limits,
      benchmarkTimeMs: executionTime,
    };
  } catch (e) {
    return { error: e.message };
  }
}

getGpuFingerprint().then(fp => {
  console.log("GPU Fingerprint:", fp);
  // A bot detector would analyze fp.deviceLimits and fp.benchmarkTimeMs
  // to check for inconsistencies or known bot signatures (e.g., very slow times).
});
```