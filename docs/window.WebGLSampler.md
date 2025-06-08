---
title: window.WebGLSampler
layout: default
---
# window.WebGLSampler
## Purpose
The `WebGLSampler` interface, part of the WebGL 2.0 API, represents an opaque object that stores sampling parameters for a WebGL texture. It defines how a GPU reads from a texture, controlling aspects like filtering (e.g., linear, nearest-neighbor) and wrapping (e.g., clamp, repeat).

## Fingerprinting Usage
While the `WebGLSampler` object itself is opaque and offers little direct entropy, its associated parameters and extensions are a significant source for fingerprinting. Anti-bot systems query the WebGL context for implementation-specific limits and supported features related to texture sampling, which are dictated by the user's GPU hardware and driver.

1.  **Anisotropic Filtering Support**: The `EXT_texture_filter_anisotropic` extension is a primary vector. Scripts can check for the presence of this extension and, more importantly, query its maximum supported level (`MAX_TEXTURE_MAX_ANISOTROPY_EXT`). This value (typically a power of two like 2, 4, 8, or 16) varies directly with the GPU hardware, providing a strong hardware identifier.
2.  **Hardware Limits**: The WebGL 2.0 specification defines minimum values for various parameters, but the actual maximums are hardware-dependent. A key parameter related to samplers is `MAX_TEXTURE_LOD_BIAS`. The exact value returned by `gl.getParameter(gl.MAX_TEXTURE_LOD_BIAS)` can differ between GPU vendors (NVIDIA, AMD, Intel, Apple) and even driver versions, adding to the fingerprint.
3.  **Inconsistency Detection**: Bots often run in virtualized or headless environments (e.g., Linux servers with Mesa/LLVMpipe for software rendering) while spoofing a common user agent (e.g., Chrome on Windows with an NVIDIA GPU). A fingerprinting script can detect this lie by comparing the user agent's claimed hardware with the actual WebGL sampler parameters. A high-end NVIDIA GPU should report `MAX_TEXTURE_MAX_ANISOTROPY_EXT` as 16, whereas a software renderer will report a much lower value (e.g., 2) or not support the extension at all. This mismatch is a high-confidence bot signal.
4.  **Absence of API**: Older or incomplete headless browser implementations may lack full WebGL 2.0 support. The simple failure to create a `WebGL2RenderingContext` or the absence of the `gl.createSampler` function is a strong indicator of a non-standard, potentially automated environment.

## Sample Code
This snippet demonstrates how to extract fingerprinting data related to texture sampling capabilities.

```javascript
function getSamplerFingerprint() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl2');

  if (!gl) {
    // WebGL 2.0 not supported, a signal in itself.
    return { error: 'WebGL 2 not supported' };
  }

  // 1. Query for the anisotropic filtering extension
  const anisotropicExt = gl.getExtension('EXT_texture_filter_anisotropic');
  const maxAnisotropy = anisotropicExt 
    ? gl.getParameter(anisotropicExt.MAX_TEXTURE_MAX_ANISOTROPY_EXT) 
    : null;

  // 2. Query for a hardware-dependent sampler parameter
  const maxLodBias = gl.getParameter(gl.MAX_TEXTURE_LOD_BIAS);

  return {
    maxAnisotropy, // e.g., 16 for high-end GPUs, 2 for Mesa, null if unsupported
    maxLodBias,    // e.g., 15 for NVIDIA, different for others
  };
}

// Example usage:
const samplerFP = getSamplerFingerprint();
console.log(samplerFP);
// Real user (NVIDIA GPU): { maxAnisotropy: 16, maxLodBias: 15 }
// Bot (headless w/ Mesa): { maxAnisotropy: 2, maxLodBias: 2 }
```