---
title: window.GPURenderPipeline
layout: default
---
# window.GPURenderPipeline
## Purpose
The `GPURenderPipeline` object is a fundamental part of the WebGPU API, representing a pre-compiled and configured state for drawing operations. It encapsulates shader programs (written in WGSL) and fixed-function state like blending, depth testing, and rasterization, enabling high-performance graphics rendering.

## Fingerprinting Usage
WebGPU, and by extension `GPURenderPipeline`, is a high-entropy fingerprinting surface because it provides low-level access to the user's Graphics Processing Unit (GPU) and its driver. This exposes a wealth of hardware-specific details that are highly stable and unique.

1.  **Hardware & Driver Enumeration**: The primary fingerprinting vector is not the pipeline object itself, but the `GPUAdapter` used to create it. The adapter exposes explicit details about the underlying hardware.
    *   **`GPUAdapter.info`**: In supporting browsers like Chrome, this property (once requested via `adapter.requestAdapterInfo()`) can directly expose the GPU `vendor` (e.g., "NVIDIA Corporation"), `architecture` (e.g., "Turing"), and `description` (e.g., "NVIDIA GeForce RTX 2080 Ti"). This is a direct, high-entropy string, analogous to WebGL's `UNMASKED_RENDERER_WEBGL`.
    *   **`GPUAdapter.features`**: This is a set of optional capabilities the GPU supports (e.g., "texture-compression-bc", "depth-clip-control"). The specific combination of supported features is unique to the GPU model and driver version.
    *   **`GPUAdapter.limits`**: This object contains dozens of numerical limits of the hardware (e.g., `maxTextureDimension2D`, `maxStorageBufferBindingSize`). The complete set of these values forms a detailed numerical fingerprint of the GPU.

2.  **Performance Benchmarking**: The time it takes to create and use a `GPURenderPipeline` for a standardized, complex rendering task serves as a performance benchmark.
    *   **Signal**: High-end GPUs will complete the task significantly faster than integrated graphics or older cards.
    *   **Bot Detection**: Bots running in headless environments often use a software renderer (like SwiftShader) or have no GPU acceleration. A WebGPU benchmark will be orders of magnitude slower on such systems compared to real user hardware, making them easy to detect.

3.  **Shader Compilation Artifacts**: The WGSL shader compiler is part of the GPU driver. Different vendors (NVIDIA, AMD, Intel, Apple) have different compiler implementations.
    *   **Signal**: Feeding the system a complex or slightly malformed shader can produce vendor-specific error messages or subtle differences in compiled output. This technique can differentiate between driver families even if the `adapter.info` strings were spoofed.

4.  **Anomaly Detection**: Anti-bot systems cross-reference WebGPU data with other signals. For example, if a browser's user-agent claims to be on a MacBook Pro (which has an Apple, AMD, or Intel GPU), but the WebGPU fingerprint reveals an NVIDIA GPU via `adapter.info` or performance characteristics, this contradiction is a strong indicator of a bot or a spoofed environment.

## Sample Code
This snippet demonstrates how to extract the most potent fingerprinting information from the `GPUAdapter`, which is the prerequisite for creating any `GPURenderPipeline`.

```javascript
async function getGpuFingerprint() {
  if (!navigator.gpu) {
    console.log("WebGPU not supported. This is a fingerprinting signal.");
    return { error: "WebGPU not supported" };
  }

  try {
    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      console.log("No GPU adapter found. Likely a virtualized environment.");
      return { error: "No adapter found" };
    }

    // Request detailed adapter information (a powerful fingerprinting source)
    const adapterInfo = await adapter.requestAdapterInfo();

    const fingerprint = {
      // Direct hardware/driver strings (high entropy)
      adapterInfo: {
        vendor: adapterInfo.vendor,
        architecture: adapterInfo.architecture,
        device: adapterInfo.device,
        description: adapterInfo.description,
      },
      // A set of supported features, unique to the hardware/driver combo
      features: [...adapter.features.values()],
      // A detailed numerical fingerprint of the hardware's capabilities
      limits: {
        maxTextureDimension2D: adapter.limits.maxTextureDimension2D,
        maxStorageBufferBindingSize: adapter.limits.maxStorageBufferBindingSize,
        maxComputeWorkgroupSizeX: adapter.limits.maxComputeWorkgroupSizeX,
        // ... and many other limits
      }
    };

    console.log("GPU Fingerprint:", fingerprint);
    return fingerprint;

  } catch (e) {
    console.error("Error accessing GPU adapter:", e);
    return { error: e.message };
  }
}

// Execute the function to collect the fingerprint
getGpuFingerprint();
```