---
title: window.NetworkInformation
layout: default
---
# window.NetworkInformation
## Purpose
The Network Information API (`navigator.connection`) provides information about the system's network connection. It exposes properties like the effective connection type (e.g., '4g', '3g'), estimated downlink speed, and round-trip time (RTT).

## Fingerprinting Usage
This API is a potent tool for bot detection, primarily through anomaly and consistency checks, rather than as a high-entropy fingerprinting source on its own.

1.  **Datacenter Detection:** This is the most critical use case. Bots running in cloud environments (AWS, Azure, GCP) exhibit network characteristics vastly different from real users.
    *   **Anomalously Low RTT:** Datacenter servers have near-zero latency to the target server, often resulting in an `rtt` value of `0` or `25` (V8 reports RTT in multiples of 25ms). A real user on a residential or mobile network will almost always have a higher RTT.
    *   **Maxed-Out Downlink:** These servers have extremely fast connections. V8 caps the reported `downlink` value at `10` (Mbps). A connection consistently reporting `rtt: 0` and `downlink: 10` is a massive red flag for a server-side bot.

2.  **Consistency and Plausibility Checks:** Anti-bot systems cross-reference network information with other data points.
    *   **IP Geolocation vs. RTT:** If a server-side IP lookup places the user in a remote country, but the client-side `rtt` is near zero, it strongly implies the user is behind a proxy located very close to the web server, not at their purported location.
    *   **Spoofing Artifacts:** A bot attempting to appear "slow" might spoof `effectiveType` to `'2g'`. However, if it fails to also spoof a correspondingly high `rtt` and low `downlink`, the inconsistency is easily detected. A `'2g'` connection with an `rtt` of `0` is impossible.

3.  **Behavioral Signals:**
    *   **Static Network:** A bot's network connection is typically static for its entire lifecycle. A real user, especially on a mobile device, might transition between Wi-Fi and cellular, triggering the `change` event on `navigator.connection`. The complete absence of this event over a long session can be a weak signal.

4.  **Entropy Contribution:** The combination of `effectiveType`, `rtt`, `downlink`, and `saveData` provides several bits of entropy. While not unique enough to identify a user alone, it helps differentiate populations and contributes to a larger device fingerprint.

## Sample Code
This snippet demonstrates a common heuristic for detecting bots running in datacenters.

```javascript
function checkNetworkAnomalies() {
  const connection = navigator.connection || navigator.mozConnection || navigator.webkitConnection;

  if (!connection) {
    // API not supported, cannot perform this check.
    return { isSuspicious: false, reason: "API not supported" };
  }

  const { rtt, downlink, effectiveType } = connection;

  // Heuristic: Datacenters often have extremely low latency and high bandwidth.
  // V8 reports RTT in 25ms increments and caps downlink at 10Mbps.
  // A value of rtt <= 25 and downlink >= 10 is highly indicative of a bot in a datacenter.
  if (rtt <= 25 && downlink >= 10) {
    return {
      isSuspicious: true,
      reason: `Datacenter-like network detected (RTT: ${rtt}ms, Downlink: ${downlink}Mbps)`
    };
  }

  // Heuristic: Check for inconsistent spoofing.
  // A 'slow' connection type should not have 'fast' RTT.
  if (effectiveType.includes('2g') && rtt < 100) {
    return {
        isSuspicious: true,
        reason: `Inconsistent network profile (Type: ${effectiveType}, RTT: ${rtt}ms)`
    };
  }

  return { isSuspicious: false, reason: "Network profile appears normal" };
}

const networkStatus = checkNetworkAnomalies();
if (networkStatus.isSuspicious) {
  console.warn("Suspicious network activity detected:", networkStatus.reason);
  // This signal would be sent to a server for correlation with other data.
}
```