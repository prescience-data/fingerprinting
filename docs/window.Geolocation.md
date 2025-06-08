---
title: window.Geolocation
layout: default
---
# window.Geolocation
## Purpose
The Geolocation API provides a high-level interface for web applications to programmatically access a device's geographical position. Access is explicitly gated by a user permission prompt for privacy.

## Fingerprinting Usage
While the high-entropy location data itself is only available after user consent, the API and the user's interaction with it provide powerful signals for bot detection and user classification.

1.  **Permission State & Interaction Latency:** The primary use in bot detection is not the location itself, but the interaction with the permission prompt.
    *   **State:** The permission status (`granted`, `denied`, `prompt`) is a basic signal. Bots are often configured to `deny` permissions instantly.
    *   **Latency:** A human user takes time (hundreds of milliseconds to several seconds) to read the prompt and make a decision. A bot using a headless browser framework (like Puppeteer or Playwright) that is pre-configured to grant or deny the permission will resolve the promise almost instantaneously (<50ms). This near-zero latency is a strong indicator of automation.

2.  **Data Consistency & Spoofing Detection:** If a location is provided, it is cross-referenced with other browser and network signals to detect spoofing.
    *   **IP vs. Geolocation API:** The location derived from the user's IP address (via a server-side lookup) is compared to the coordinates from the Geolocation API. A significant mismatch (e.g., IP in Virginia, USA; Geolocation API in London, UK) is a major red flag.
    *   **Timezone Consistency:** The browser's reported timezone (`Intl.DateTimeFormat().resolvedOptions().timeZone`) must be consistent with the geographical coordinates. A device reporting coordinates in Los Angeles but a timezone of `Europe/Paris` is highly suspicious.
    *   **Data Perfection:** Real-world GPS/Wi-Fi positioning data has inherent noise and variability. Spoofed locations provided by bots are often unnaturally "perfect" (e.g., exact coordinates of a city center, an accuracy value of exactly `1`, or an altitude of `0`).

3.  **API Availability & Errors:** The presence or absence of the `navigator.geolocation` object can be a minor signal. More importantly, the specific error code returned on failure (e.g., `PERMISSION_DENIED` vs. `POSITION_UNAVAILABLE`) can differentiate between a user's choice and a technical limitation, which can be correlated with specific bot environments.

## Sample Code
This snippet demonstrates measuring interaction latency and flagging a common bot behavior (instant denial).

```javascript
async function checkGeolocationBehavior() {
  if (!navigator.geolocation) {
    console.log('Bot signal: Geolocation API not supported.');
    return;
  }

  const requestTime = performance.now();

  try {
    const position = await new Promise((resolve, reject) => {
      navigator.geolocation.getCurrentPosition(resolve, reject, {
        timeout: 10000 // Set a timeout
      });
    });

    const responseTime = performance.now();
    const latency = responseTime - requestTime;

    console.log(`Geolocation granted after ${latency.toFixed(2)} ms.`);
    if (latency < 50) {
      console.log('High confidence bot signal: Permission granted too quickly.');
    }
    
    // Further checks for spoofing would go here, e.g.:
    // checkConsistency(position.coords, userIpInfo, browserTimezone);

  } catch (error) {
    const responseTime = performance.now();
    const latency = responseTime - requestTime;

    if (error.code === error.PERMISSION_DENIED) {
      console.log(`Geolocation denied after ${latency.toFixed(2)} ms.`);
      if (latency < 50) {
        console.log('High confidence bot signal: Permission denied too quickly.');
      }
    } else {
      console.log(`Geolocation error: ${error.message}`);
    }
  }
}

checkGeolocationBehavior();
```