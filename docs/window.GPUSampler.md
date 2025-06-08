---
title: window.GPUSampler
layout: default
---
# window.GPUSampler
## Purpose
The `GPUSampler` object is a core component of the WebGPU API. It defines the rules and parameters for how a GPU shader samples (reads) data from a `GPUTexture`, controlling aspects like filtering (e.g., linear, nearest-neighbor), addressing modes (e.g., clamp, repeat), and level of detail.

## Fingerprinting Usage
`GPUSampler` and the broader WebGPU API are high-entropy sources for fingerprinting due to their direct interaction with the user's graphics hardware and drivers.

1.  **Hardware Capability Limits:** The most potent fingerprinting vector is the set of limits imposed by the underlying GPU and driver. When creating a `GPUSampler`, the `maxAnisotropy` property is a key differentiator. Different GPUs support different maximum levels of anisotropic filtering (commonly values like 2, 4, 8, 16). A script can query the `GPUDevice.limits.maxAnisotropy` value directly, or attempt to create a sampler with a high value and see if it throws a validation error. This value is a stable and highly identifying piece of hardware information.

2.  **Implementation Inconsistencies:** Bots and automated browsers (e.g., Puppeteer, Playwright) often use software-based renderers (like SwiftShader on Windows/Android or Lavapipe on Linux) or have incomplete WebGPU implementations.
    *   **Renderer String:** The `GPUAdapter.requestAdapterInfo()` method can reveal the underlying renderer (e.g., "ANGLE (NVIDIA GeForce RTX 4090 Direct3D11 vs_5_0 ps_5_0)" for a real user vs. "Apple M1" vs. "SwiftShader" for a bot). A mismatch between this string and the User-Agent is a strong bot signal.
    *   **API Availability:** The mere presence or absence of `navigator.gpu` or the ability to successfully request a `GPUDevice` can filter out older browsers or non-compliant automation tools.
    *   **Behavioral Anomalies:** A bot might try to spoof its hardware limits. A detection script can verify these claims by attempting to create a resource (like a `GPUSampler`) that uses the claimed limit. If a bot spoofs a `maxAnisotropy` of 16 but its underlying renderer only supports 4, the creation will fail, exposing the lie.

3.  **Driver-Specific Behavior:** Minor differences in how various GPU drivers (NVIDIA, AMD, Intel, Apple) implement the WebGPU specification can lead to subtle behavioral variations. While harder to exploit, advanced fingerprinting libraries may test edge cases in sampler configuration to elicit driver-specific responses, further increasing entropy.

## Sample Code
This snippet demonstrates how to extract the `maxAnisotropy` limit and check for the presence of a software renderer, two common fingerprinting techniques using the WebGPU API.

```javascript
async function getGpuFingerprint() {
  const fingerprint = {
    webGpuSupported: false,
    maxAnisotropy: null,
    adapterVendor: null,
    adapterRenderer: null,
    isSoftwareRenderer: false,
  };

  if (!navigator.gpu) {
    return fingerprint;
  }

  try {
    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return fingerprint;
    }

    const device = await adapter.requestDevice();
    if (!device) {
      return fingerprint;
    }

    fingerprint.webGpuSupported = true;

    // 1. Extract the hardware limit for maxAnisotropy
    fingerprint.maxAnisotropy = device.limits.maxAnisotropy;

    // 2. Get adapter info to check for software renderers
    if ('requestAdapterInfo' in adapter) {
      const info = await adapter.requestAdapterInfo();
      fingerprint.adapterVendor = info.vendor;
      fingerprint.adapterRenderer = info.architecture; // Or info.description
      
      // Known software renderer substrings
      const softwareRenderers = ['swiftshader', 'lavapipe', 'llvmpipe'];
      const rendererInfo = (info.description || '').toLowerCase();
      fingerprint.isSoftwareRenderer = softwareRenderers.some(sr => rendererInfo.includes(sr));
    }
    
    device.destroy();
    return fingerprint;

  } catch (error) {
    console.error("WebGPU fingerprinting failed:", error);
    return fingerprint; // Return partial or default data on error
  }
}

// Usage:
getGpuFingerprint().then(fp => {
  console.log("GPU Fingerprint:", fp);
  // Example Output on a real device:
  // {
  //   webGpuSupported: true,
  //   maxAnisotropy: 16,
  //   adapterVendor: "NVIDIA",
  //   adapterRenderer: "NVIDIA GeForce RTX 4090",
  //   isSoftwareRenderer: false
  // }
});
```