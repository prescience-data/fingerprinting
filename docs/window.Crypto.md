---
title: window.crypto
layout: default
---
# window.crypto
## Purpose
The Web Cryptography API provides a set of low-level cryptographic primitives for web applications. Its primary functions include generating cryptographically strong random numbers (`getRandomValues`) and performing operations like hashing, signing, and encryption via the `crypto.subtle` interface.

## Fingerprinting Usage
The `window.crypto` API is a potent tool for fingerprinting and bot detection, primarily through performance analysis rather than the output of the cryptographic functions themselves.

1.  **Hardware-Accelerated Performance:** The `crypto.subtle` interface, particularly the `digest()` method for hashing, is often accelerated by underlying hardware instruction sets (e.g., AES-NI and SHA extensions on x86/ARM CPUs). By measuring the execution time of a hashing operation on a large block of data, a script can create a high-entropy fingerprint of the user's CPU. A bot running in a virtualized environment without hardware passthrough will exhibit significantly slower performance than a real user on bare metal, creating a clear anomaly. This is one of the most reliable signals for detecting VMs and emulators.

2.  **Implementation Consistency:** While the cryptographic results are standardized, the exact set of supported algorithms, curves, and key formats can vary slightly between browser engines (Blink, Gecko, WebKit) and versions. Anti-bot systems can check for the presence and correct functioning of specific, less common algorithms to verify the integrity of the browser environment. A bot attempting to spoof its user agent may fail these specific checks if its underlying engine doesn't match the claimed browser.

3.  **PRNG Analysis:** While `crypto.getRandomValues` is designed to be secure, it's possible (though difficult) to perform statistical analysis on large samples of its output to detect non-standard Pseudo-Random Number Generator (PRNG) implementations. A more common bot detection technique is to check for consistency: if a bot hooks `Math.random()` to provide deterministic values but fails to also hook `crypto.getRandomValues`, the discrepancy between the two random sources is a strong signal of tampering.

## Sample Code
This snippet demonstrates timing the `crypto.subtle.digest` function, a common technique to detect virtualized environments based on performance.

```javascript
async function getCryptoTimingFingerprint() {
  if (!window.crypto || !window.crypto.subtle) {
    return 'CryptoAPINotSupported';
  }

  try {
    // Prepare a reasonably large buffer to make timing differences more apparent.
    const data = new Uint8Array(1024 * 1024 * 10); // 10MB
    window.crypto.getRandomValues(data);

    const startTime = performance.now();
    // Hash the data using a common, often hardware-accelerated algorithm.
    await window.crypto.subtle.digest('SHA-256', data);
    const endTime = performance.now();

    const duration = endTime - startTime;

    // Anti-bot logic would compare this duration to a baseline.
    // e.g., A real Chrome on a modern CPU might take < 50ms.
    // A VM without hardware acceleration could take > 200ms.
    console.log(`SHA-256 hashing took ${duration.toFixed(2)} ms.`);
    return duration;
  } catch (error) {
    return 'CryptoOperationFailed';
  }
}

getCryptoTimingFingerprint();
```