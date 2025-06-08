---
title: window.GPUComputePipeline
layout: default
---
# window.GPUComputePipeline
## Purpose
`GPUComputePipeline` is a core interface of the WebGPU API. It represents a pre-compiled, reusable pipeline for performing general-purpose computations on the GPU, encapsulating a compute shader written in WGSL (WebGPU Shading Language).

## Fingerprinting Usage
`GPUComputePipeline`, and the WebGPU API in general, is a high-entropy source for fingerprinting and a powerful tool for bot detection. Its low-level access to the GPU exposes hardware and driver characteristics that are highly specific to a user's machine.

1.  **Performance Benchmarking:** This is the primary vector. A standardized, complex computation (e.g., matrix multiplication, cryptographic hashing, physics simulation) is executed via a `GPUComputePipeline`. The time taken to complete the task is measured with high precision (`performance.now()` or WebGPU's own timestamp queries). This timing is a direct reflection of:
    *   **GPU Hardware:** Different GPU models (NVIDIA, AMD, Intel, Apple M-series, mobile GPUs) have vastly different architectures and performance profiles. An RTX 4090 will complete a task orders of magnitude faster than an integrated Intel GPU.
    *   **Driver Version:** The GPU driver contains the shader compiler. Different driver versions can have different compiler optimizations, leading to measurable performance variations for the same hardware.
    *   **Operating System & Power State:** The OS scheduler and the device's power management (e.g., laptop on battery vs. plugged in) can influence GPU clock speeds and performance.

2.  **Bot & VM Detection:** Bots and virtualized environments often exhibit clear anomalies:
    *   **Software Renderer:** Headless browsers often fall back to a software renderer like Google's SwiftShader or LLVMpipe. A WebGPU compute task will be dramatically slower (100x-1000x) on a software renderer compared to real hardware. This performance cliff is a near-certain indicator of a non-standard environment. The adapter information (`navigator.gpu.requestAdapter()`) can also explicitly name the software renderer.
    *   **Lack of GPU:** The `navigator.gpu.requestAdapter()` call may return `null`, indicating no compatible GPU is available.
    *   **Virtualized GPU (vGPU):** Environments like VMWare or VirtualBox expose virtualized GPUs. These have unique performance characteristics and often lack support for advanced WebGPU features, causing pipeline creation to fail.

3.  **Shader Compiler Quirks:** The process of creating the pipeline itself (`device.createComputePipelineAsync`) involves compiling the WGSL shader. Fingerprinting scripts can supply complex, esoteric, or intentionally malformed WGSL code to probe the driver's compiler.
    *   **Success/Failure:** Whether the pipeline compiles successfully is a binary signal. A shader might work on an NVIDIA driver but fail on an AMD driver due to subtle differences in spec interpretation.
    *   **Error Messages:** The specific validation error message thrown upon failure can be unique to the browser and driver stack.

The combination of performance timings, supported features/limits (queried from the `GPUAdapter`), and shader compilation behavior creates a highly stable and unique fingerprint that is very difficult to spoof without possessing the actual hardware.

## Sample Code
This snippet demonstrates the core logic of using a compute pipeline for performance-based fingerprinting. It times a simple compute task that squares numbers in a buffer.

```javascript
async function getGpuComputeFingerprint() {
  try {
    if (!navigator.gpu) {
      return "WEBGPU_UNSUPPORTED";
    }

    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return "WEBGPU_NO_ADAPTER";
    }
    const device = await adapter.requestDevice();

    // WGSL shader code for a simple compute task
    const shaderModule = device.createShaderModule({
      code: `
        @group(0) @binding(0) var<storage, read_write> data: array<f32>;

        @compute @workgroup_size(64)
        fn main(@builtin(global_invocation_id) global_id: vec3<u32>) {
          let index = global_id.x;
          data[index] = data[index] * data[index];
        }
      `,
    });

    // Create the pipeline. Failure here is a fingerprinting signal.
    const pipeline = await device.createComputePipelineAsync({
      layout: 'auto',
      compute: {
        module: shaderModule,
        entryPoint: 'main',
      },
    });

    // Prepare data and buffers
    const input = new Float32Array(Array.from({ length: 1024 * 1024 }, (_, i) => i));
    const workBuffer = device.createBuffer({
      size: input.byteLength,
      usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC | GPUBufferUsage.COPY_DST,
      mappedAtCreation: true,
    });
    new Float32Array(workBuffer.getMappedRange()).set(input);
    workBuffer.unmap();

    const bindGroup = device.createBindGroup({
      layout: pipeline.getBindGroupLayout(0),
      entries: [{ binding: 0, resource: { buffer: workBuffer } }],
    });

    // --- The Core Fingerprinting Logic ---
    const commandEncoder = device.createCommandEncoder();
    const passEncoder = commandEncoder.beginComputePass();
    passEncoder.setPipeline(pipeline);
    passEncoder.setBindGroup(0, bindGroup);
    passEncoder.dispatchWorkgroups(Math.ceil(input.length / 64));
    passEncoder.end();
    
    const startTime = performance.now();
    device.queue.submit([commandEncoder.finish()]);
    await device.queue.onSubmittedWorkDone();
    const endTime = performance.now();
    // --- End of Core Logic ---

    const executionTime = endTime - startTime;

    // The fingerprint combines the adapter name and the precise execution time.
    // A real script would use a more complex task and hash the result.
    const adapterInfo = await adapter.requestAdapterInfo();
    return `${adapterInfo.vendor}:${adapterInfo.architecture}:${executionTime.toFixed(4)}ms`;

  } catch (error) {
    // Errors during setup or compilation are also a fingerprint.
    return `WEBGPU_ERROR:${error.name}`;
  }
}

// Usage:
getGpuComputeFingerprint().then(fingerprint => {
  console.log("GPU Compute Fingerprint:", fingerprint);
  // Example output on real hardware: "nvidia:unknown:1.8520ms"
  // Example output on SwiftShader: "google:cpu:157.4310ms"
});
```