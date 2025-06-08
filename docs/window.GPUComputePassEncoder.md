---
title: window.GPUComputePassEncoder
layout: default
---
# window.GPUComputePassEncoder
## Purpose
Part of the WebGPU API, `GPUComputePassEncoder` is an interface for recording commands for a compute pass. This allows web applications to leverage the GPU for high-performance, general-purpose parallel computation, distinct from rendering graphics.

## Fingerprinting Usage
WebGPU, and `GPUComputePassEncoder` by extension, is a high-entropy source for fingerprinting and a powerful tool for bot detection. Its usage exposes deep hardware and driver-level characteristics that are difficult to spoof.

1.  **Hardware & Driver Fingerprint:** The primary use is to execute a standardized computation (a "compute shader") on the GPU and read back the result. The final output can vary subtly due to:
    *   **GPU Architecture:** Different GPU vendors (NVIDIA, AMD, Intel, Apple) and even different models from the same vendor have unique floating-point unit implementations.
    *   **Driver Version:** The GPU driver that translates WebGPU commands into hardware instructions can introduce minor variations. Updates to drivers can change the output.
    *   **Method:** A script runs a complex mathematical function on an array of numbers. The resulting array is read back from the GPU memory, converted to a string, and hashed. This hash becomes a highly stable and unique fingerprint for that specific hardware/driver combination.

2.  **Performance Benchmarking:** The time it takes to complete a compute pass is a strong behavioral signal.
    *   A script can measure the duration of a `dispatchWorkgroups()` call for a fixed workload.
    *   This differentiates high-end gaming PCs, low-end integrated graphics, and mobile devices.
    *   Crucially, it unmasks bots. Headless browsers often use software-based rendering (like SwiftShader) or run in virtualized data center environments with standardized, non-consumer GPUs. Their performance profile will be drastically different from a real user's machine and inconsistent with their claimed User-Agent.

3.  **Feature and Limit Probing:** While not using `GPUComputePassEncoder` directly, its availability is part of a larger WebGPU fingerprint. Scripts can query `navigator.gpu.limits` to get specific hardware limits, such as `maxComputeWorkgroupsPerDimension` or `maxComputeInvocationsPerWorkgroup`. The combination of these limits provides a detailed hardware profile.

4.  **Proof-of-Work:** Anti-bot systems can issue a unique WebGPU compute challenge. A real user's browser can solve it quickly, while a bot farm would find the GPU computation prohibitively expensive to run at scale, thus acting as a throttling mechanism.

## Sample Code
This snippet demonstrates the core logic of running a compute shader, reading the result, and creating a hash. It abstracts the full setup for clarity.

```javascript
async function getGpuComputeFingerprint() {
  if (!navigator.gpu) {
    return { error: "WebGPU not supported" };
  }

  try {
    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return { error: "No GPU adapter found" };
    }
    const device = await adapter.requestDevice();

    // 1. A simple compute shader that performs floating-point math.
    // The specific operations are chosen to expose hardware/driver differences.
    const shaderCode = `
      @group(0) @binding(0) var<storage, read_write> data: array<f32>;

      @compute @workgroup_size(64)
      fn main(@builtin(global_invocation_id) global_id: vec3<u32>) {
        let i = global_id.x;
        // Non-trivial math to produce subtle variations
        data[i] = pow(data[i], 2.051) * sin(data[i] * 0.987);
      }
    `;
    const shaderModule = device.createShaderModule({ code: shaderCode });

    // 2. Prepare data and buffers
    const input = new Float32Array(Array.from({ length: 128 }, (_, i) => i + 0.1));
    const outputBuffer = device.createBuffer({
      size: input.byteLength,
      usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC | GPUBufferUsage.COPY_DST,
    });
    device.queue.writeBuffer(outputBuffer, 0, input);

    // 3. Create pipeline and bind group
    const pipeline = device.createComputePipeline({
      layout: 'auto',
      compute: { module: shaderModule, entryPoint: 'main' },
    });
    const bindGroup = device.createBindGroup({
      layout: pipeline.getBindGroupLayout(0),
      entries: [{ binding: 0, resource: { buffer: outputBuffer } }],
    });

    // 4. Record and submit compute commands using GPUComputePassEncoder
    const commandEncoder = device.createCommandEncoder();
    const passEncoder = commandEncoder.beginComputePass(); // The API in question
    passEncoder.setPipeline(pipeline);
    passEncoder.setBindGroup(0, bindGroup);
    passEncoder.dispatchWorkgroups(Math.ceil(input.length / 64));
    passEncoder.end();

    // 5. Read back the result from the GPU
    const readbackBuffer = device.createBuffer({
      size: input.byteLength,
      usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
    });
    commandEncoder.copyBufferToBuffer(outputBuffer, 0, readbackBuffer, 0, input.byteLength);
    device.queue.submit([commandEncoder.finish()]);

    await readbackBuffer.mapAsync(GPUMapMode.READ);
    const result = new Float32Array(readbackBuffer.getMappedRange());
    
    // 6. Create a hash from the result array
    const fingerprint = btoa(result.toString()); // In reality, a better hash like SHA-256 would be used.
    readbackBuffer.unmap();

    return { fingerprint };

  } catch (e) {
    return { error: e.message };
  }
}

// Usage:
// getGpuComputeFingerprint().then(console.log);
// Example output: { fingerprint: "MC4wMDAwMDc4NjM4NjM..." }
```