---
title: window.GPUQueue
layout: default
---
# window.GPUQueue
## Purpose
The `GPUQueue` interface, a core component of the WebGPU API, represents a command queue on a logical GPU device. Its primary purpose is to accept and execute command buffers, which contain instructions for rendering, computation, and data copying.

## Fingerprinting Usage
`GPUQueue` is a high-entropy source for fingerprinting and a powerful tool for bot detection due to its direct interaction with the system's graphics hardware.

1.  **Performance Benchmarking:** The most significant use is measuring the precise execution time of a standardized compute or rendering workload. By submitting a command buffer with a complex shader and using a `timestamp-query`, a script can measure the GPU's processing time in nanoseconds. This timing is highly specific to the GPU model, driver version, system temperature, and current load, creating a very stable and unique identifier. Bots running in headless environments often use a software renderer (like SwiftShader) or a virtualized GPU, which yields performance results that are drastically different and easily distinguishable from real user hardware.

2.  **Behavioral Analysis & Error Probing:** The way a `GPUQueue` handles various commands can reveal the underlying driver and hardware. Submitting malformed, complex, or edge-case command buffers can trigger specific errors or produce subtly different outputs on different GPU vendor stacks (NVIDIA, AMD, Intel, Apple). A bot environment might fail to initialize WebGPU entirely, or its `GPUQueue` might throw predictable errors when certain advanced features are attempted, flagging it as a non-standard client.

3.  **Consistency Anomaly:** While high performance is expected from real hardware, *perfectly consistent* performance can be an anomaly. A real user's system has background processes and OS-level GPU scheduling, causing minor "jitter" or variations in repeated performance tests. A bot in a sterile, isolated virtual environment may produce timings that are too consistent, lacking the natural noise of a real system.

## Sample Code
This snippet demonstrates how to measure the execution time of a simple compute shader, a core technique for GPU fingerprinting. The resulting `durationNs` is a high-entropy value.

```javascript
async function getGpuExecutionTime() {
  if (!navigator.gpu) {
    return { error: "WebGPU not supported." };
  }

  try {
    const adapter = await navigator.gpu.requestAdapter();
    const device = await navigator.gpu.requestDevice();
    const queue = device.queue;

    // 1. Create a timestamp query set to measure GPU-side execution time
    const querySet = device.createQuerySet({
      type: "timestamp",
      count: 2,
    });
    const queryBuffer = device.createBuffer({
      size: querySet.count * 8, // 8 bytes per timestamp (64-bit)
      usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC,
    });
    const resultBuffer = device.createBuffer({
      size: querySet.count * 8,
      usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
    });

    // 2. Create a simple compute workload (e.g., a loop)
    const shaderModule = device.createShaderModule({
      code: `
        @compute @workgroup_size(1)
        fn main() {
          for (var i: u32 = 0u; i < 1000000u; i = i + 1u) {
            // Intensive work simulation
          }
        }
      `,
    });
    const pipeline = device.createComputePipeline({
      layout: 'auto',
      compute: { module: shaderModule, entryPoint: "main" },
    });
    const bindGroup = device.createBindGroup({ layout: pipeline.getBindGroupLayout(0) });

    // 3. Encode and submit the commands to the GPUQueue
    const commandEncoder = device.createCommandEncoder();
    commandEncoder.writeTimestamp(querySet, 0); // Start timestamp
    const passEncoder = commandEncoder.beginComputePass();
    passEncoder.setPipeline(pipeline);
    passEncoder.setBindGroup(0, bindGroup);
    passEncoder.dispatchWorkgroups(1);
    passEncoder.end();
    commandEncoder.writeTimestamp(querySet, 1); // End timestamp
    commandEncoder.resolveQuerySet(querySet, 0, 2, queryBuffer, 0);
    commandEncoder.copyBufferToBuffer(queryBuffer, 0, resultBuffer, 0, resultBuffer.size);

    queue.submit([commandEncoder.finish()]);

    // 4. Read back the results and calculate duration
    await resultBuffer.mapAsync(GPUMapMode.READ);
    const times = new BigInt64Array(resultBuffer.getMappedRange());
    const durationNs = Number(times[1] - times[0]);

    resultBuffer.unmap();
    querySet.destroy();
    queryBuffer.destroy();
    resultBuffer.destroy();

    // This duration is a powerful fingerprinting signal
    return { durationNs };

  } catch (e) {
    return { error: e.message };
  }
}

// Usage:
getGpuExecutionTime().then(result => {
  console.log(result);
  // An anti-bot system would send this result to a server for analysis.
  // e.g., { durationNs: 134578 } for a real GPU
  // e.g., { durationNs: 8976543 } for a software renderer
});
```