---
title: window.GPUDevice
layout: default
---
# navigator.gpu (WebGPU API)
## Purpose
The WebGPU API, accessed via `navigator.gpu`, provides modern, low-level control over the system's Graphics Processing Unit (GPU). It is the successor to WebGL, designed for high-performance graphics rendering and general-purpose GPU (GPGPU) computations directly within the browser.

## Fingerprinting Usage
WebGPU is a high-entropy source for fingerprinting due to the detailed hardware and driver information it exposes. Unlike WebGL, which primarily exposed a single renderer string, WebGPU provides a structured and extensive set of properties.

*   **Hardware & Driver Identification:** The `GPUAdapterInfo` object (obtained via `adapter.requestAdapterInfo()`) directly exposes the GPU `vendor` (e.g., "NVIDIA Corporation"), `architecture` (e.g., "turing"), and `device` (e.g., "NVIDIA GeForce RTX 2080"). This is a highly stable and unique identifier for a user's hardware setup.

*   **Software Renderer Detection:** A primary method for bot detection is identifying headless browsers or virtual machines. These environments often lack dedicated GPUs and fall back to software renderers. WebGPU will report vendors like "Google Inc." and devices like "SwiftShader", which is a strong signal that the environment is not a typical user's machine.

*   **Device Limits and Features:** The `GPUDevice` object contains a wealth of specific, hardware-dependent values.
    *   `device.limits`: An object containing dozens of numeric limits like `maxTextureDimension2D`, `maxStorageBufferBindingSize`, and `maxComputeWorkgroupSizeX`. The exact combination of these values is highly specific to the GPU model and driver version.
    *   `device.features`: A `GPUSupportedFeatures` set listing optional capabilities the hardware supports, such as `"texture-compression-bc"` or `"depth-clip-control"`. The specific set of supported features adds significant entropy.

*   **Behavioral & Performance Analysis:**
    *   **Initialization Time:** The time it takes for `requestAdapter()` and `requestDevice()` to resolve can be a behavioral signal.
    *   **Compute Benchmarking:** A small, standardized compute shader can be executed, and its completion time measured. This creates a performance benchmark that can differentiate between high-end, low-end, and software-emulated GPUs, exposing inconsistencies (e.g., a user agent claiming to have a powerful machine that performs poorly).

*   **Inconsistency Detection:** The detailed GPU information can be cross-referenced with other data points, like the User-Agent string. A User-Agent for macOS reporting an NVIDIA GPU (which Apple no longer uses in modern Macs) is a clear anomaly indicating spoofing.

## Sample Code
This snippet demonstrates how to collect key fingerprinting data from the WebGPU API.

```javascript
async function getGpuFingerprint() {
  if (!navigator.gpu) {
    return { error: "WebGPU not supported" };
  }

  try {
    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return { error: "No GPU adapter found" };
    }

    // Get structured hardware and driver info (high entropy)
    const adapterInfo = await adapter.requestAdapterInfo();

    const device = await adapter.requestDevice();

    // Collect specific hardware limits (high entropy)
    const limits = {};
    for (const key in device.limits) {
      limits[key] = device.limits[key];
    }

    // Collect supported features (adds more entropy)
    const features = [...device.features.values()];

    return {
      adapterInfo: {
        vendor: adapterInfo.vendor,
        architecture: adapterInfo.architecture,
        device: adapterInfo.device,
        description: adapterInfo.description,
      },
      features,
      limits,
    };
  } catch (error) {
    return { error: error.message };
  }
}

// Usage:
getGpuFingerprint().then(fp => {
  console.log(fp);
  // A bot detector would send this JSON object to a server for analysis.
  // Example check: if (fp.adapterInfo.vendor === 'Google Inc.') { /* likely a bot */ }
});
```