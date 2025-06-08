---
title: window.GPUCanvasContext
layout: default
---
# window.GPUCanvasContext
## Purpose
The `GPUCanvasContext` interface is the main entry point for the WebGPU API on a specific `<canvas>` element. It is obtained via `canvas.getContext('webgpu')` and is used to configure the canvas and present frames rendered by the GPU.

## Fingerprinting Usage
WebGPU provides extremely high-entropy data by exposing low-level details about the user's Graphics Processing Unit (GPU) and driver stack. It is one of the most powerful fingerprinting surfaces in modern browsers.

1.  **GPU Adapter Information**: The `navigator.gpu.requestAdapter()` method returns a `GPUAdapter` object. The subsequent call to `adapter.requestAdapterInfo()` reveals highly specific strings identifying the GPU vendor (NVIDIA, AMD, Intel, Apple), architecture (e.g., "turing", "ampere", "rdna2"), and device model. This information is a primary source of fingerprinting entropy. Bots or virtualized environments often expose generic software renderers (e.g., "Google SwiftShader") or virtual GPU adapters (e.g., "VMware SVGA II Adapter"), which are immediate red flags.

2.  **Supported Features & Limits**: The `GPUAdapter` object contains a `features` set and a `limits` object.
    *   `adapter.features`: A list of optional capabilities the GPU/driver supports, such as specific texture compression formats (`"texture-compression-bc"`, `"texture-compression-astc"`). The exact combination of supported features is highly unique.
    *   `adapter.limits`: A list of maximum values for various resources, like `maxStorageBufferBindingSize` or `maxComputeInvocationsPerWorkgroup`. These numeric constants are dictated by the hardware and driver, creating a detailed hardware fingerprint.

3.  **Rendered Output Hashing**: Similar to WebGL fingerprinting, a complex scene or computation can be rendered off-screen using WebGPU. The resulting pixel data is then read back and hashed. Subtle differences in GPU hardware, floating-point arithmetic, and driver-level optimizations will produce a different hash for different systems, creating a robust and stable identifier.

4.  **Performance Benchmarking**: The execution time of a standardized, complex compute shader can be measured. The performance profile is a strong behavioral signal that correlates directly with the underlying hardware. Bots running in overloaded virtual machines or using slow software renderers will have significantly different timings than real users on native hardware.

## Sample Code
This snippet demonstrates how to collect the most direct and high-entropy information from the WebGPU API.

```javascript
async function getGpuFingerprint() {
  if (!navigator.gpu) {
    return { error: "WebGPU not supported." };
  }

  try {
    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      // This is a strong signal for a bot or a misconfigured environment.
      return { error: "No GPU adapter found." };
    }

    // Requesting adapter info is a separate async operation.
    const adapterInfo = await adapter.requestAdapterInfo();

    // Collect high-entropy data points.
    const fingerprint = {
      // Direct hardware identifiers.
      vendor: adapterInfo.vendor,
      architecture: adapterInfo.architecture,
      device: adapterInfo.device,
      description: adapterInfo.description,
      // A list of supported optional features.
      features: [...adapter.features.values()],
      // A detailed map of hardware/driver limits.
      limits: {
        maxTextureDimension2D: adapter.limits.maxTextureDimension2D,
        maxStorageBufferBindingSize: adapter.limits.maxStorageBufferBindingSize,
        maxComputeWorkgroupSizeX: adapter.limits.maxComputeWorkgroupSizeX,
        // ... and many other limits
      }
    };

    return fingerprint;

  } catch (error) {
    return { error: error.message };
  }
}

// Usage:
getGpuFingerprint().then(fp => {
  console.log(fp);
  // An anti-bot system would send this JSON object to a server for analysis.
});
```