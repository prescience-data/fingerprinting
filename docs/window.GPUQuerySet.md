---
title: window.GPUQuerySet
layout: default
---
# window.GPUQuerySet
## Purpose
The `GPUQuerySet` interface, part of the WebGPU API, is used to create objects that store the results of GPU queries. These queries can measure rendering occlusion (how many pixels were drawn) or, more importantly for fingerprinting, capture high-precision timestamps of GPU command execution.

## Fingerprinting Usage
`GPUQuerySet` is a high-entropy source for fingerprinting and a powerful tool for bot detection. Its primary use is through `'timestamp'` queries, which enable micro-benchmarking of the user's GPU.

1.  **Hardware & Driver Fingerprint:** By executing a standardized set of compute or render commands and measuring the execution time with nanosecond precision via `writeTimestamp()`, a script can generate a highly specific performance signature. This signature is determined by the GPU model, VRAM speed, core clock speed, and, crucially, the graphics driver version. The exact time taken to complete a complex shader is a very stable and unique identifier for a specific hardware/software stack.

2.  **Software Renderer Detection:** A primary goal of bot detection is to identify headless browsers or virtual machines running without dedicated GPU hardware. These environments often fall back to a software renderer (e.g., Google's SwiftShader, Mesa/LLVMpipe on Linux). A timestamp query will immediately reveal this:
    *   **Performance:** Software renderers are orders of magnitude slower than actual hardware. A simple rendering task that takes 500ns on an NVIDIA GPU might take 50,000ns on SwiftShader. This difference is trivial to detect.
    *   **Timing Characteristics:** The statistical distribution of timings from a software renderer is different from that of a hardware GPU, providing another signal.

3.  **API Availability & Quirks:**
    *   **Presence:** The mere absence of `navigator.gpu` or the failure to create a `GPUQuerySet` in a browser that claims to be a modern Chrome version is a strong indicator of a non-standard environment (e.g., an older automation framework or a browser with WebGPU disabled).
    *   **Timestamp Precision:** The resolution of the timestamp counter can vary between GPU vendors (NVIDIA, AMD, Intel, Apple) and their drivers. While the spec aims for nanosecond precision, the actual reported values might be quantized differently, leaking information about the underlying hardware.

4.  **Behavioral Probing:** A script can issue a series of queries with varying workloads. A real GPU's response times will scale predictably with the workload's complexity. Bots attempting to spoof these values by returning random or static numbers will fail this consistency check.

## Sample Code
This snippet demonstrates the core logic of using a timestamp query to measure GPU performance, which forms the basis of a fingerprint.

```javascript
async function getGpuBenchmark() {
  try {
    if (!navigator.gpu) {
      return { error: "WebGPU not supported." };
    }

    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return { error: "No GPU adapter found." };
    }
    const device = await adapter.requestDevice();

    // 1. Create a query set to store two timestamps (start and end).
    const querySet = device.createQuerySet({
      type: 'timestamp',
      count: 2,
    });

    // 2. Create a buffer to resolve the query results into.
    const resolveBuffer = device.createBuffer({
      size: querySet.count * 8, // 2 timestamps * 8 bytes/timestamp (64-bit)
      usage: GPUBufferUsage.QUERY_RESOLVE | GPUBufferUsage.COPY_SRC,
    });

    // 3. Create a buffer to read the results back to the CPU.
    const resultBuffer = device.createBuffer({
      size: resolveBuffer.size,
      usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
    });

    const encoder = device.createCommandEncoder();

    // 4. Write start timestamp, perform a workload, write end timestamp.
    encoder.writeTimestamp(querySet, 0); // Start timestamp
    // In a real scenario, a standardized compute or render pass would go here.
    encoder.writeTimestamp(querySet, 1); // End timestamp

    // 5. Resolve the queries into the resolveBuffer.
    encoder.resolveQuerySet(querySet, 0, querySet.count, resolveBuffer, 0);

    // 6. Copy the results to a readable buffer.
    encoder.copyBufferToBuffer(resolveBuffer, 0, resultBuffer, 0, resultBuffer.size);

    device.queue.submit([encoder.finish()]);

    // 7. Map the buffer and read the results.
    await resultBuffer.mapAsync(GPUMapMode.READ);
    const times = new BigInt64Array(resultBuffer.getMappedRange());
    const durationNanoseconds = times[1] - times[0];

    resultBuffer.unmap();

    // This duration is a high-entropy fingerprinting value.
    console.log(`GPU task duration: ${durationNanoseconds} ns`);
    return { fingerprint: Number(durationNanoseconds) };

  } catch (e) {
    return { error: e.message };
  }
}

getGpuBenchmark().then(result => {
  if (result.fingerprint) {
    // This value is sent to a server for analysis.
    // A value > 20000 might indicate a software renderer (bot).
    // A specific value like 834 might correlate to a specific GPU/driver.
  }
});
```