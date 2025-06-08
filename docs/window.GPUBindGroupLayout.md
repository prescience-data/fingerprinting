---
title: window.GPUBindGroupLayout
layout: default
---
# window.GPUBindGroupLayout
## Purpose
The `GPUBindGroupLayout` interface is a core component of the WebGPU API. It defines the structure, type, and accessibility (visibility to shader stages) of resources like buffers, textures, and samplers that a shader pipeline will use.

## Fingerprinting Usage
`GPUBindGroupLayout`, as part of the low-level WebGPU API, is a rich source of fingerprinting entropy by exposing hardware and driver-specific implementation details.

1.  **Hardware Limit Probing:** The most significant vector is testing the limits of what can be defined in a layout. While the limits themselves are exposed via `GPUDevice.limits` (e.g., `maxBindingsPerBindGroup`), bots may try to spoof these values. Actively trying to create a `GPUBindGroupLayout` that pushes or exceeds these limits and observing whether it succeeds or fails provides a more robust signal. For example, attempting to create a layout with 17 bindings when `maxBindingsPerBindGroup` is reported as 16 can verify the true hardware constraint. The combination of various limits (buffer sizes, binding counts) creates a high-entropy hardware fingerprint.

2.  **Error Message Fingerprinting:** The exact text of the `GPUValidationError` messages thrown when creating an invalid layout can differ based on the browser, OS, and underlying graphics driver (e.g., ANGLE on Windows, Metal on macOS, Vulkan on Linux/Android). By intentionally creating invalid layouts (e.g., with duplicate binding numbers, invalid resource combinations), fingerprinting scripts can capture the precise error string. These strings are often stable for a given software/hardware stack and can be used to identify it.

3.  **Feature Support and Quirks:** Different GPUs and drivers have varying levels of support for advanced WebGPU features that are configured within the bind group layout. This includes support for specific texture formats, read-write storage textures, or buffer-texture binding combinations. Creating layouts that use these features and checking for errors can reveal the capabilities of the underlying hardware, differentiating high-end discrete GPUs from low-power integrated ones.

4.  **API Availability:** The mere existence and correct functioning of `GPUDevice.prototype.createBindGroupLayout` can be a simple check. Many bot environments or older headless browsers lack WebGPU support entirely, causing an immediate failure that distinguishes them from modern browsers.

## Sample Code
This snippet demonstrates fingerprinting by capturing the error message from an intentionally invalid `GPUBindGroupLayout` descriptor.

```javascript
async function getBindGroupLayoutErrorFingerprint() {
  try {
    if (!navigator.gpu) {
      return "WebGPU not supported";
    }

    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return "No GPU adapter found";
    }

    const device = await adapter.requestDevice();

    // Intentionally create an invalid layout descriptor with a duplicate binding index.
    const invalidDescriptor = {
      entries: [{
        binding: 0, // First entry at binding 0
        visibility: GPUShaderStage.COMPUTE,
        buffer: { type: 'uniform' }
      }, {
        binding: 0, // DUPLICATE binding 0
        visibility: GPUShaderStage.COMPUTE,
        buffer: { type: 'storage' }
      }]
    };

    try {
      device.createBindGroupLayout(invalidDescriptor);
    } catch (e) {
      // The error message is the fingerprinting signal.
      // e.g., "Validation error: binding 0 is specified more than once."
      // The exact phrasing can vary by browser, OS, and driver.
      return e.message;
    }

    return "No error thrown for invalid layout"; // This would be an anomaly itself.

  } catch (error) {
    // Catches errors from requestAdapter/requestDevice
    return `Error during WebGPU setup: ${error.message}`;
  }
}

// Usage:
getBindGroupLayoutErrorFingerprint().then(fingerprint => {
  console.log("BindGroupLayout Fingerprint:", fingerprint);
});
```