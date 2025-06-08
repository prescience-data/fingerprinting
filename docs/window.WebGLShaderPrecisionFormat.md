---
title: window.WebGLShaderPrecisionFormat
layout: default
---
# WebGLRenderingContext.getShaderPrecisionFormat()
## Purpose
This method, part of the WebGL API, queries the graphics driver for the supported range and precision of numeric data types (integers and floating-point numbers) within WebGL shaders. This information is crucial for developers to write shaders that execute correctly and efficiently across different hardware.

## Fingerprinting Usage
The values returned by `getShaderPrecisionFormat` are a direct reflection of the user's Graphics Processing Unit (GPU) and its associated driver. This provides a highly stable and high-entropy fingerprinting vector.

*   **Hardware & Driver Signature:** The combination of `rangeMin`, `rangeMax`, and `precision` for vertex and fragment shaders across different float/int types creates a detailed signature. This signature varies significantly between GPU vendors (NVIDIA, AMD, Intel, Apple, Qualcomm), specific GPU models, and even driver versions. This allows for fine-grained device identification.

*   **Bot Detection Anomaly:** This is one of the most powerful signals for detecting headless browsers and virtualized environments.
    *   **Inconsistency:** A bot may spoof its User-Agent to claim it's Chrome on Windows with an NVIDIA GPU. However, if it's running in a cloud environment without a physical GPU, it will likely use a software renderer like Google's SwiftShader or the Mesa 3D Graphics Library (`llvmpipe`). These software renderers have their own unique and well-documented precision format values, which will not match any real consumer hardware. This mismatch between the claimed environment and the actual WebGL signature is a definitive indicator of a non-human user.
    *   **Missing Values:** A poorly configured bot environment might lack WebGL support entirely, causing this API call to fail. This absence of data is itself a suspicious signal.

## Sample Code
This snippet collects the precision data for various shader and data types, creating a composite signature that can be used for fingerprinting.

```javascript
function getWebGLPrecisionFingerprint() {
  try {
    const canvas = document.createElement('canvas');
    const gl = canvas.getContext('webgl');
    if (!gl) {
      return 'NO_WEBGL';
    }

    const shaderTypes = ['VERTEX_SHADER', 'FRAGMENT_SHADER'];
    const precisionTypes = ['LOW_FLOAT', 'MEDIUM_FLOAT', 'HIGH_FLOAT', 'LOW_INT', 'MEDIUM_INT', 'HIGH_INT'];
    
    const signature = [];

    shaderTypes.forEach(shader => {
      precisionTypes.forEach(precision => {
        const format = gl.getShaderPrecisionFormat(gl[shader], gl[precision]);
        // The format object contains rangeMin, rangeMax, and precision
        signature.push(format.rangeMin, format.rangeMax, format.precision);
      });
    });

    // The resulting array of numbers is a stable hardware signature.
    // Real-world scripts often join and hash this value.
    // e.g., "127,127,23,-127,-127,23,..."
    return signature.join(',');

  } catch (e) {
    return 'WEBGL_ERROR';
  }
}

// A fingerprinting script would collect and likely hash this value.
const precisionFingerprint = getWebGLPrecisionFingerprint();
// console.log(precisionFingerprint);
```