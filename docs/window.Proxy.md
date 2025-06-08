---
title: window.Proxy
layout: default
---
# window.Proxy
## Purpose
The `Proxy` object is a standard, built-in ES6 feature that allows for the creation of a wrapper around another object (the "target"). This wrapper can intercept and redefine fundamental operations for the target, such as property lookup (`get`), assignment (`set`), and function invocation (`apply`), via handler functions known as "traps".

## Fingerprinting Usage
`Proxy` is not a direct source of entropy. Instead, its primary relevance to bot detection is in **detecting API tampering and behavioral anomalies**. Automation frameworks (like Puppeteer/Playwright) and malicious scripts often modify or spoof browser APIs to hide their presence (e.g., `navigator.webdriver`, `window.chrome`). `Proxy` is a powerful tool for both sides of this arms race.

1.  **Detecting Proxied APIs (Defensive):** Anti-bot scripts can check if sensitive objects like `navigator` or `window` have been wrapped in a `Proxy` by an automation script. While there is no standard `isProxy()` method, detection relies on behavioral side effects. A common technique is to check the `toString()` representation of an object's methods. A native function will typically return `"[native code]"`, whereas a function on a proxied object might not, revealing the tampering.

2.  **Honeypotting (Offensive):** This is the most powerful use case. An anti-bot script can wrap a sensitive object (e.g., `navigator`) in its own `Proxy`. It then sets a `get` trap on properties that bots frequently check, such as `webdriver`. When the bot's script attempts to read `navigator.webdriver`, the trap is triggered, providing a high-confidence signal that the user agent is automated. The proxy can still return the expected `false` value to the bot, making the detection invisible.

3.  **Detecting `Proxy` Tampering:** A sophisticated bot might try to disable this detection method by overwriting `window.Proxy` itself. Anti-bot scripts can counter this by checking the integrity of the `Proxy` constructor, for example, by verifying that `window.Proxy.toString()` returns `"[native code]"`. If it doesn't, it's a strong indication that the environment has been manipulated.

4.  **Feature Support (Low Entropy):** Simply checking for the existence of `window.Proxy` confirms a modern (ES6+) JavaScript environment. While most modern bots run in such environments, its absence can be a flag for an old or non-standard client, which might be suspicious in contexts where modern browsers are expected.

## Sample Code
This snippet demonstrates the "honeypot" technique. It creates a proxy for the `navigator` object to detect when any script attempts to access the `webdriver` property.

```javascript
// A flag to be set upon detection.
let botDetectionSignal = false;

// Define the handler with a 'get' trap.
const navigatorHandler = {
  get: function(target, property, receiver) {
    // If any script requests the 'webdriver' property...
    if (property === 'webdriver') {
      console.warn('FINGERPRINTING: "navigator.webdriver" was accessed.');
      botDetectionSignal = true;
      // This could trigger a report to a server.
    }
    // Crucially, forward the original operation to the real navigator object.
    // This ensures functionality is not broken and the detection is stealthy.
    return Reflect.get(target, property, receiver);
  }
};

// Create the proxy. In a real implementation, this would be done in a
// way that intercepts all checks, though you cannot directly overwrite window.navigator.
// This code demonstrates the core trapping mechanism.
const proxiedNavigator = new Proxy(window.navigator, navigatorHandler);

// --- Simulation ---
// 1. A normal script might access userAgent, which passes through without issue.
console.log('Normal access:', proxiedNavigator.userAgent);

// 2. A bot script checks the webdriver flag.
console.log('Bot access simulation:', proxiedNavigator.webdriver);

// 3. The detection signal is now true.
if (botDetectionSignal) {
  console.log('High-confidence bot signal detected.');
}
```