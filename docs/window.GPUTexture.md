---
title: window.GPUTexture
layout: default
---
# window.GPUTexture
## Purpose
`GPUTexture` is a core interface of the WebGPU API. It represents a multidimensional array of data (texels) that resides in GPU memory, used for storing image data, serving as render targets, or holding data for general-purpose GPU (GPGPU) computations.

## Fingerprinting Usage
`GPUTexture` is not a direct source of entropy but is a critical tool for probing the underlying GPU hardware and drivers, making it a cornerstone of advanced WebGPU fingerprinting. Its usage falls into three main categories:

1.  **Capability and Limit Probing:** Anti-bot systems attempt to create `GPUTexture` objects with various, often extreme, configurations to test the limits of the GPU and its driver. The success or failure of these creation attempts reveals specific hardware capabilities. Key probed parameters include:
    *   **`size`**: Testing the maximum supported texture dimensions (e.g., `width`, `height`, `depthOrArrayLayers`). A high-end discrete GPU supports much larger textures than an integrated GPU or a virtualized GPU in a headless environment.
    *   **`format`**: Iterating through the list of available `GPUTextureFormat` values. The set of supported formats (e.g., `rgba8unorm`, `depth24plus-stencil8`, `bc7-rgba-unorm-srgb`) is highly dependent on the GPU hardware and driver.
    *   **`sampleCount`**: Probing supported multisample anti-aliasing (MSAA) levels (e.g., 1, 4, 8). Support for higher sample counts is a feature of more powerful GPUs.
    *   **`usage`**: Combining different `GPUTextureUsage` flags. Some combinations may be invalid or unsupported on certain hardware, triggering a `Validation Error` which serves as a fingerprinting signal.

2.  **Render-to-Texture Hashing:** This is the WebGPU equivalent of canvas fingerprinting, but with a much higher potential for entropy.
    *   A `GPUTexture` is created with the `RENDER_ATTACHMENT` usage flag.
    *   A complex, deterministic scene is rendered into this texture using a specific shader program.
    *   The resulting pixel data is copied from the GPU texture back to the CPU.
    *   A hash (e.g., SHA-256) is computed from this pixel data.
    *   Subtle differences in floating-point calculations, anti-aliasing algorithms, and shader compilation across different GPU models, driver versions, and operating systems result in unique pixel-level variations, producing a highly stable and unique hash.

3.  **Performance Benchmarking:** The time it takes to perform operations on a `GPUTexture` is a strong behavioral signal. Scripts can measure the duration of tasks like:
    *   Writing data to a large texture using `queue.writeTexture`.
    *   Copying data between two textures using `commandEncoder.copyTextureToTexture`.
    *   Using the texture as a source in a complex compute shader.
    The resulting timings provide a precise performance profile of the GPU and its memory bus, which is extremely difficult for bots or virtualized environments to spoof accurately.

## Sample Code
This snippet demonstrates the "Capability and Limit Probing" technique by attempting to create a texture with an extremely large dimension. The success or failure of this operation becomes a data point in the user's fingerprint.

```javascript
async function checkGpuTextureLimits() {
  if (!navigator.gpu) {
    console.log("WebGPU not supported.");
    return { supported: false, error: "WebGPU not available" };
  }

  try {
    const adapter = await navigator.gpu.requestAdapter();
    const device = await adapter.requestDevice();

    // Probe by attempting to create a texture with an extreme size.
    // Most consumer hardware will fail, but the specific error or success
    // is a fingerprinting signal. High-end GPUs might succeed where others fail.
    device.createTexture({
      size: { width: 32768, height: 32768 }, // 16384 is a common limit
      format: 'rgba8unorm',
      usage: GPUTextureUsage.TEXTURE_BINDING,
    });

    // If this line is reached, the device supports very large textures.
    return { supported: true, detail: "Supports 32k texture" };
  } catch (e) {
    // The type of error (e.g., ValidationError, OutOfMemoryError) and its
    // message are valuable fingerprinting data.
    return { supported: false, error: e.name, message: e.message };
  }
}

checkGpuTextureLimits().then(result => {
  // The 'result' object contributes to the overall browser fingerprint.
  // For example: { supported: false, error: "ValidationError", message: "..." }
  console.log("Texture Limit Probe:", result);
});
```