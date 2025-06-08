---
title: window.GPUShaderModule
layout: default
---
# window.GPUShaderModule
## Purpose
A `GPUShaderModule` is an object within the WebGPU API that represents a compiled shader program. It takes shader code written in WebGPU Shading Language (WGSL) and prepares it for execution by the GPU, holding the compiled result which can then be used in a `GPURenderPipeline` or `GPUComputePipeline`.

## Fingerprinting Usage
`GPUShaderModule` and its creation process (`device.createShaderModule`) are a high-entropy source for fingerprinting, evolving the concepts from WebGL fingerprinting to a much more detailed level. The primary vector is not the successful execution of a shader, but the nuances of the compilation process itself.

*   **Compilation Errors and Warnings:** This is the most potent vector. Different GPU vendors (NVIDIA, AMD, Intel, Apple), operating systems, and driver versions have unique shader compilers. By feeding the compiler a series of complex, non-standard, or slightly malformed WGSL shaders, a script can observe the resulting compilation messages. The `getCompilationInfo()` method returns a `GPUCompilationInfo` object containing an array of messages, each with a `type` (`error`, `warning`, `info`), `message` text, `lineNum`, and `linePos`. The exact text and structure of these error/warning messages are highly specific to the underlying driver stack, creating a detailed and stable fingerprint.

*   **Performance and Timing:** The time required to compile a complex shader can be a behavioral signal. A high-end discrete GPU will compile significantly faster than an integrated GPU or a software-based renderer (like SwiftShader or Mesa's Lavapipe) often found in headless/bot environments. Measuring the duration of the `createShaderModule` promise resolution can help classify the underlying hardware.

*   **Supported Features:** While not part of the module itself, the features supported by the `GPUDevice` dictate what kind of WGSL code can be successfully compiled. A script can attempt to compile shaders that use optional features (e.g., `texture-compression-bc`, `shader-f16`) to probe the capabilities of the GPU, adding bits of entropy to the fingerprint.

*   **Bot Detection Anomalies:**
    *   **Software Renderer:** The most significant red flag. If the `GPUAdapter` identifies as using a software renderer (e.g., "SwiftShader", "llvmpipe"), it is almost certainly a virtualized or bot environment, as real users have hardware acceleration.
    *   **Inconsistency:** A bot might spoof its User-Agent to claim it's Chrome on Windows with an NVIDIA GPU, but the shader compilation errors match those of the Mesa drivers on Linux. This contradiction is a strong signal of deception.
    *   **Absence or Failure:** If `navigator.gpu` is undefined, or if `requestAdapter()` / `requestDevice()` fails, it indicates an older browser or a simplified bot environment that lacks WebGPU support. A bot might also intentionally break `createShaderModule` to prevent fingerprinting, which is itself a detectable behavior.

## Sample Code
This snippet demonstrates how to probe the shader compiler for unique error messages, which form the basis of a fingerprint.

```javascript
async function getShaderFingerprint() {
  if (!navigator.gpu) {
    return "WebGPU not supported";
  }

  try {
    const adapter = await navigator.gpu.requestAdapter();
    const device = await adapter.requestDevice();

    // This WGSL code is intentionally slightly malformed or uses advanced features
    // to trigger unique compiler warnings or errors based on the driver stack.
    // A real fingerprinting script would use a battery of different shaders.
    const shaderCode = `
      @vertex
      fn main(@builtin(vertex_index) vi : u32) -> @builtin(position) vec4<f32> {
        var pos = array<vec2<f32>, 3>(
          vec2<f32>(0.0, 0.5),
          vec2<f32>(-0.5, -0.5),
          vec2<f32>(0.5, -0.5)
        );
        // The following line might produce different behavior or warnings on different compilers
        let x: f32 = 1; 
        return vec4<f32>(pos[vi], x, 1.0);
      }
    `;

    const shaderModule = device.createShaderModule({
      code: shaderCode,
    });

    const info = await shaderModule.getCompilationInfo();

    // The fingerprint is derived from the compilation messages.
    let fingerprint = "";
    if (info.messages.length > 0) {
      fingerprint = info.messages.map(m => `${m.type}:${m.lineNum}:${m.message}`).join(';');
    } else {
      fingerprint = "Compilation successful with no messages.";
    }
    
    // A hash of this string would be used as the final fingerprint.
    console.log("Shader Fingerprint String:", fingerprint);
    return fingerprint;

  } catch (e) {
    // Errors during device creation or compilation are also a signal.
    return `Error: ${e.message}`;
  }
}

getShaderFingerprint();
```