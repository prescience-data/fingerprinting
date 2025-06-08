---
title: window.fetch
layout: default
---
# window.fetch
## Purpose
Provides a modern, promise-based interface for making asynchronous network requests to retrieve resources across the network. It is the standard mechanism for AJAX and API communication in modern web applications.

## Fingerprinting Usage
While `fetch` itself doesn't have high entropy, its implementation and behavior are critical for bot detection. Anti-bot systems focus less on what `fetch` *is* and more on whether it has been tampered with or behaves abnormally.

1.  **Implementation Integrity Check:** This is the most common and powerful use in bot detection. Automation frameworks like Puppeteer or Playwright, and malicious scripts, often override or "hook" `window.fetch` to intercept, monitor, or modify network traffic. A detection script can check for this tampering.
    *   **`toString()` method:** Calling `window.fetch.toString()` on a native, unmodified function in Chrome/V8 returns the string `'function fetch() { [native code] }'`. Any other result, such as a full function definition, indicates that the function has been overridden by a JavaScript polyfill or a wrapper, which is a massive red flag for automation.
    *   **Prototype Chain:** Verifying that `window.fetch`'s prototype and constructor are the expected native `Function` objects.

2.  **Behavioral Analysis:** The manner and context in which `fetch` is called can reveal automation.
    *   **CORS and Opaque Responses:** A `fetch` request with `mode: 'no-cors'` to a cross-origin resource results in an "opaque" response in real browsers. This means the script receives a response object with `status: 0`, empty headers, and an unreadable body. Poorly implemented bots or non-browser environments (like Node.js) may fail to replicate this specific behavior, either by throwing an error or returning an accessible response. This can be used as a test for environment authenticity.
    *   **Request Timing:** Measuring the latency between a user input event (e.g., `click`) and the subsequent `fetch` call. Bots often execute this with zero or unnaturally consistent delays, whereas human interaction has a small, variable delay.

3.  **Header Consistency Probing:** `fetch` is used to send requests to a server-side analysis endpoint. The server then inspects the incoming HTTP headers for signs of spoofing. For example, it checks if the `User-Agent` string is consistent with the JavaScript-accessible `navigator.userAgent` and, more importantly, with modern `Sec-CH-UA` (Client Hints) headers. Headless browsers or simple HTTP clients often fail to send a consistent set of headers that a real Chrome browser would.

## Sample Code
This snippet demonstrates the most critical technique: checking if `window.fetch` has been modified from its native implementation.

```javascript
/**
 * Checks if the global fetch function is the native browser implementation.
 * Overridden fetch is a strong indicator of an automation tool or bot script
 * attempting to intercept or modify network requests.
 * @returns {boolean} True if fetch appears to be native, false otherwise.
 */
function isFetchNative() {
  if (typeof window.fetch !== 'function') {
    return false; // Fetch doesn't exist or is not a function.
  }
  
  // In genuine Chrome/V8, the string representation is highly specific.
  // Any deviation suggests it has been polyfilled, wrapped, or is running
  // in a non-standard environment.
  const fetchString = window.fetch.toString();
  
  return fetchString === 'function fetch() { [native code] }';
}

// Example usage by a detection script:
if (!isFetchNative()) {
  // Flag this user as suspicious. The fetch API has been tampered with.
  console.warn('Suspicious activity detected: window.fetch is not native.');
  // An anti-bot system would send a report here.
  // navigator.sendBeacon('/analytics/flag', JSON.stringify({ reason: 'fetch_tampered' }));
} else {
  console.log('window.fetch appears to be the native implementation.');
}
```