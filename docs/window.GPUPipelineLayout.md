---
title: window.GPUPipelineLayout
layout: default
---
# window.GPUPipelineLayout
## Purpose
`GPUPipelineLayout` is an interface within the WebGPU API. It defines the interface between a shader program and the resources it uses (e.g., buffers, textures, samplers), specifying how these resources are organized into bind groups for a render or compute pipeline.

## Fingerprinting Usage
`GPUPipelineLayout` is a high-entropy source for fingerprinting and a strong signal for bot detection. Its utility comes from the complex validation logic that browsers must perform, which is tightly coupled to the underlying operating system, graphics driver, and hardware.

1.  **Validation Error Fingerprinting**: The primary method involves creating deliberately invalid or complex `GPUPipelineLayout` descriptors. The WebGPU implementation in the browser validates this descriptor against the specification and hardware/driver limits. The resulting `GPUValidationError` message is highly specific. The text of this error message can vary significantly between:
    *   **Browsers**: Chrome, Firefox, and Safari have different WebGPU implementations (Dawn, wgpu, WebKit-WebGPU).
    *   **Operating Systems**: The underlying graphics API (Direct3D 12 on Windows, Metal on macOS, Vulkan on Linux/ChromeOS) influences validation logic and error reporting.
    *   **GPU Drivers**: Different driver versions for the same GPU can produce unique error strings or exhibit different validation behaviors for edge cases.
    *   **Bots vs. Real Browsers**: Automated browsers often use software renderers (e.g., SwiftShader) or have incomplete/buggy WebGPU implementations. They may fail to throw an error, throw a generic `TypeError` instead of a `GPUValidationError`, or produce a known, non-hardware-accelerated error message.

2.  **Hardware Limit Probing**: The `GPUPipelineLayout` is constrained by limits reported in `navigator.gpu.limits` (e.g., `maxBindGroups`, `maxDynamicUniformBuffersPerPipelineLayout`). A script can create layouts that test these boundaries. The exact behavior when a limit is reached or slightly exceeded (e.g., the specific error thrown) can differentiate hardware and driver combinations. This is effective at unmasking headless browsers that report spoofed limits but fail to enforce them correctly.

3.  **Behavioral Signals**: The mere ability to successfully create a valid `GPUPipelineLayout` is a strong positive signal for a real, modern browser. Many botting frameworks lack WebGPU support entirely. A script that attempts to initialize a WebGPU device and create a layout can quickly filter out less sophisticated bots. If `navigator.gpu.requestAdapter()` returns `null`, or if `device.createPipelineLayout()` fails unexpectedly, it strongly suggests an automated or outdated environment.

## Sample Code
This snippet attempts to create a pipeline layout with an invalid `visibility` property for a buffer. The resulting error message is captured and can be used as a fingerprint.

```javascript
async function getPipelineLayoutFingerprint() {
  try {
    if (!navigator.gpu) {
      return "WebGPU not supported";
    }
    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return "WebGPU adapter not available";
    }
    const device = await adapter.requestDevice();

    // Create a deliberately invalid layout descriptor.
    // A valid visibility flag is a bitwise OR of GPUShaderStage flags.
    // Using an arbitrary number like 999 is invalid.
    const invalidLayoutDescriptor = {
      bindGroupLayouts: [
        device.createBindGroupLayout({
          entries: [{
            binding: 0,
            visibility: 999, // Invalid value
            buffer: { type: 'uniform' }
          }]
        })
      ]
    };

    device.createPipelineLayout(invalidLayoutDescriptor);
    // If it doesn't throw, that's a fingerprint itself.
    return "No error thrown for invalid layout";

  } catch (e) {
    // The error name and message are high-entropy fingerprinting vectors.
    // e.g., "GPUValidationError: Stage visibility (999) is not a valid combination of GPUShaderStage flags."
    // This string differs across browsers, OS, and drivers.
    return `${e.name}: ${e.message}`;
  }
}

getPipelineLayoutFingerprint().then(fingerprint => {
  console.log("WebGPU Pipeline Layout Fingerprint:", fingerprint);
});
```