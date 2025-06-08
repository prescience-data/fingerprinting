---
title: window.RTCDtlsTransport
layout: default
---
# window.RTCDtlsTransport
## Purpose
The `RTCDtlsTransport` interface provides details about the Datagram Transport Layer Security (DTLS) session established for an `RTCPeerConnection`. It represents the secure channel used for exchanging encrypted media and data in a WebRTC session.

## Fingerprinting Usage
`RTCDtlsTransport` is a high-entropy source for fingerprinting and a critical component in bot detection due to the characteristics of the auto-generated, self-signed certificates used in the DTLS handshake.

1.  **Certificate Analysis**: The primary fingerprinting vector is the `getRemoteCertificates()` method. When a WebRTC connection is established, each peer generates a temporary certificate. The properties of this certificate are not standardized and vary significantly based on:
    *   **Browser Engine & Version**: Chrome (using BoringSSL), Firefox (using NSS), and Safari (using libwebrtc with platform-specific crypto) generate certificates with distinct signature algorithms (e.g., ECDSA with P-256 vs. RSA), validity periods, and serial number formats. These details can pinpoint the browser family and often the specific version range.
    *   **Operating System**: The underlying OS cryptographic libraries can influence certificate generation, adding another layer of entropy.
    *   **Headless Browsers & Automation**: Automated browsers like Puppeteer or Playwright, when running in a standard configuration, will produce certificates consistent with the underlying Chrome version. However, custom WebRTC stacks or modified browser builds used by sophisticated bots may produce anomalous certificates (e.g., using different algorithms, having unusual validity dates, or lacking expected extensions). The complete failure to establish a DTLS transport is also a strong bot signal.

2.  **State Transition & Timing**: The `state` property (`connecting`, `connected`, `failed`) provides behavioral signals. Anti-bot systems can monitor the transitions. A legitimate browser will progress through the states predictably. A bot might fail immediately, get stuck in a state, or exhibit unusual timing patterns when establishing the connection, which can be flagged when analyzed at scale.

3.  **Consistency Check**: The certificate details serve as a powerful cross-validation signal. A browser might send a User-Agent string claiming to be Chrome on Windows, but if its DTLS certificate has the signature of Firefox on Linux, it is almost certainly a spoofed client. This inconsistency between a low-entropy, easily-spoofed signal (User-Agent) and a high-entropy, hard-to-spoof signal (DTLS certificate) is a definitive bot detection method.

## Sample Code
This snippet demonstrates how to establish a local loopback WebRTC connection to force the generation of a DTLS certificate, extract it, and create a hash from it for fingerprinting.

```javascript
async function getDtlsCertificateFingerprint() {
  try {
    // Use a loopback connection to force certificate generation locally.
    const pc = new RTCPeerConnection();
    
    // A data channel is needed to trigger the DTLS handshake.
    pc.createDataChannel('fingerprint');

    pc.onicecandidate = e => {
      if (e.candidate) {
        pc.addIceCandidate(e.candidate);
      }
    };

    const offer = await pc.createOffer();
    await pc.setLocalDescription(offer);
    
    // In a real scenario, this offer would be sent to a remote peer.
    // Here we simulate the remote peer by using the same peer connection.
    const answer = await pc.createAnswer();
    await pc.setRemoteDescription(answer);

    return new Promise((resolve, reject) => {
      // The 'connected' state indicates the DTLS handshake is complete.
      pc.addEventListener('connectionstatechange', async () => {
        if (pc.connectionState === 'connected') {
          const sender = pc.getSenders()[0]; // Or any sender/receiver
          if (!sender || !sender.transport) {
            reject('Could not get DTLS transport.');
            return;
          }
          
          const transport = sender.transport;
          const certificates = transport.getRemoteCertificates();

          if (certificates.length > 0) {
            // The certificate is an ArrayBuffer. Hash it for a stable fingerprint.
            const certBuffer = certificates[0];
            const hashBuffer = await crypto.subtle.digest('SHA-256', certBuffer);
            const hashArray = Array.from(new Uint8Array(hashBuffer));
            const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
            resolve(hashHex);
          } else {
            reject('No remote certificates found.');
          }
          pc.close();
        } else if (pc.connectionState === 'failed') {
            reject('WebRTC connection failed.');
            pc.close();
        }
      });
    });
  } catch (error) {
    // Errors here are a signal in themselves (e.g., WebRTC is disabled).
    console.error('RTCDtlsTransport fingerprinting failed:', error);
    return null;
  }
}

// Usage:
getDtlsCertificateFingerprint()
  .then(fingerprint => {
    console.log('DTLS Certificate Fingerprint (SHA-256):', fingerprint);
  })
  .catch(err => console.error(err));
```