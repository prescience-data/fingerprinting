---
title: window.Worker
layout: default
---
# window.Worker
## Purpose
The `Worker` interface enables running JavaScript in a background thread, separate from the main user interface (UI) thread. This allows for long-running, computationally intensive tasks to be performed without freezing the page and degrading the user experience.

## Fingerprinting Usage
Web Workers are a powerful tool for both fingerprinting and bot detection due to their performance characteristics and isolated execution environment.

1.  **Hardware Concurrency Probing:** The most common use is to verify the value of `navigator.hardwareConcurrency`. While a bot can easily spoof the `hardwareConcurrency` property on the main `navigator` object, it cannot fake the actual performance of the underlying hardware. A script can create a number of workers equal to the reported `hardwareConcurrency` and assign them all a CPU-intensive task (e.g., cryptographic hashing, prime number calculation). By measuring the total time to completion, the script can detect inconsistencies.
    *   **Signal:** If `hardwareConcurrency` is reported as `16`, but running 16 parallel workers is not significantly faster than running 8, it suggests the reported value is spoofed and the machine has fewer physical cores. The performance curve across a varying number of workers is a high-entropy fingerprint of the CPU's architecture.

2.  **Implementation & Anomaly Detection:** The mere existence and correct functioning of Workers can be a check.
    *   **Bot Weakness:** Some less sophisticated headless browsers or bot frameworks may not implement the Worker API correctly, or may disable it to conserve resources. A simple `try...catch` block around `new Worker()` can instantly flag these environments.
    *   **Environment Discrepancies:** The global scope inside a worker (`WorkerGlobalScope`) is different from the main thread's `window`. A bot attempting to emulate a browser might incorrectly expose properties like `window` or `document` inside the worker's scope. Checking for the absence of these APIs within a worker can reveal an imperfect emulation.

3.  **JIT Engine & Timing Analysis:** Each Worker thread runs in its own V8 Isolate (or equivalent in other engines), with its own memory heap and JIT compiler instance.
    *   **Behavioral Signal:** By running the exact same complex JavaScript function on the main thread and inside a worker, and measuring their execution times with `performance.now()`, subtle differences in JIT compilation and optimization can be observed. These micro-timing variations can contribute to a device fingerprint and may differ between real Chrome and automated frameworks like Puppeteer/Playwright, which manage the V8 engine differently.

## Sample Code
This snippet demonstrates how to verify `navigator.hardwareConcurrency` by measuring the time it takes to complete a task with a corresponding number of workers. A significant mismatch between the expected and actual performance is a strong bot signal.

```javascript
// This code would be part of a larger fingerprinting script.

async function probeHardwareConcurrency() {
  const reportedCores = navigator.hardwareConcurrency;
  console.log(`navigator.hardwareConcurrency reports: ${reportedCores} cores.`);

  // Worker script content created as a string, then a Blob.
  // This avoids needing a separate .js file.
  const workerScript = `
    self.onmessage = () => {
      // A simple, CPU-bound task.
      let result = 0;
      for (let i = 0; i < 10e6; i++) {
        result += Math.sqrt(i);
      }
      self.postMessage(result);
    };
  `;
  const blob = new Blob([workerScript], { type: 'application/javascript' });
  const workerUrl = URL.createObjectURL(blob);

  const workers = [];
  const promises = [];

  for (let i = 0; i < reportedCores; i++) {
    const worker = new Worker(workerUrl);
    workers.push(worker);
    promises.push(new Promise(resolve => {
      worker.onmessage = () => {
        worker.terminate(); // Clean up the worker
        resolve();
      };
    }));
  }

  const startTime = performance.now();

  // Start all workers
  workers.forEach(worker => worker.postMessage('start'));

  // Wait for all workers to finish
  await Promise.all(promises);

  const endTime = performance.now();
  const duration = endTime - startTime;

  console.log(`Time to complete task with ${reportedCores} workers: ${duration.toFixed(2)}ms`);

  // Cleanup the Blob URL
  URL.revokeObjectURL(workerUrl);

  // Anti-bot logic would compare this duration against an expected baseline
  // for the reported number of cores. A very high duration suggests spoofing.
  return duration;
}

probeHardwareConcurrency();
```