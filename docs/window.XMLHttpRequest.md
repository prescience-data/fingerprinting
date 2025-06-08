---
title: window.XMLHttpRequest
layout: default
---
# window.XMLHttpRequest
## Purpose
The `XMLHttpRequest` (XHR) object is a browser API used to send HTTP or HTTPS requests to a web server and load the server response data back into the script. It is the foundation of AJAX (Asynchronous JavaScript and XML).

## Fingerprinting Usage
`XMLHttpRequest` is not a source of high-entropy static fingerprinting data itself. Instead, it is a powerful tool for **behavioral analysis** and **environment validation**. Anti-bot systems use it to probe the client's underlying networking stack and execution environment.

1.  **CORS Enforcement:** This is a primary detection method. A real browser strictly enforces the Cross-Origin Resource Sharing (CORS) policy. A script can attempt an XHR request to a cross-origin endpoint controlled by the server, which is intentionally configured *without* the proper `Access-Control-Allow-Origin` header.
    *   **Real Browser:** The request will fail, and the `onerror` event handler will be triggered. The response body will be inaccessible to the script.
    *   **Simple Bot (e.g., Node.js with `axios` or `request`):** These libraries do not enforce CORS. The request will succeed, and the bot will be able to read the response, revealing its non-browser nature.

2.  **HTTP Header Consistency:** When an XHR is sent, the browser automatically attaches a specific set of headers (`Accept`, `Accept-Language`, `Accept-Encoding`, `User-Agent`, `Referer`, `Sec-CH-UA`, etc.).
    *   Anti-bot systems check if the headers received on the server-side match the expected headers for the claimed `User-Agent`.
    *   Headless browsers like Puppeteer or Playwright often require manual configuration to perfectly replicate the header order and composition of a real Chrome browser, and inconsistencies are a strong bot signal. For example, a client claiming to be modern Chrome but missing the `Sec-CH-UA` (Client Hints) headers is highly suspicious.

3.  **Event and State Timing:** The timing and sequence of XHR events (`readystatechange`, `loadstart`, `progress`, `load`, `onerror`, `ontimeout`) can be measured.
    *   The micro-timing between `readyState` changes (from 1 to 4) can differ between a real browser's native network stack and an emulated environment.
    *   A bot may not perfectly replicate the firing of all events. Forcing a request to time out and checking if the `ontimeout` event fires correctly is a common test.

4.  **Object Properties and `toString()`:** While low-entropy, this can help identify the underlying JavaScript engine.
    *   `Object.prototype.toString.call(new XMLHttpRequest())` returns `"[object XMLHttpRequest]"` in Chrome. Any deviation can indicate a hooked or emulated object.
    *   The order and presence of properties on the XHR object instance can differ slightly between browser engines (V8 vs. SpiderMonkey vs. JavaScriptCore), providing a clue to the browser family.

## Sample Code
This snippet demonstrates a CORS enforcement check. A server would host `/cors-test` and deliberately omit the `Access-Control-Allow-Origin` header.

```javascript
function checkCORSBehavior() {
  return new Promise(resolve => {
    const xhr = new XMLHttpRequest();
    const testUrl = 'https://api.example.com/cors-test'; // A controlled, cross-origin endpoint

    // In a real browser, this handler will be called due to CORS policy violation.
    xhr.onerror = function() {
      console.log('Behavior: Correct. Request failed as expected.');
      resolve({ isBrowser: true, reason: 'CORS policy enforced' });
    };

    // In a simple bot, this handler might be called, indicating no CORS enforcement.
    xhr.onload = function() {
      console.log('Behavior: Suspicious. Cross-origin request succeeded.');
      resolve({ isBrowser: false, reason: 'CORS policy not enforced' });
    };

    // Set a timeout to handle cases where neither onload nor onerror fires.
    xhr.ontimeout = function() {
        resolve({ isBrowser: 'unknown', reason: 'Request timed out' });
    };
    xhr.timeout = 2000;

    try {
      xhr.open('GET', testUrl, true);
      xhr.send();
    } catch (e) {
      // Some environments might throw an exception immediately.
      resolve({ isBrowser: true, reason: 'Exception on send() due to CORS' });
    }
  });
}

checkCORSBehavior().then(result => {
  // Send this result back to the server for analysis.
  // A result of { isBrowser: false } is a very strong bot signal.
  console.log(result);
});
```