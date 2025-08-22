# Fail-Open Signing in AppleMediaServices (Zero-Day Disclosure)

**Summary**

This issue constitutes a Zero-Day vulnerability and reflects a systemic design flaw in `AppleMediaServices.framework`, rather than a regression or version-specific bug.

A critical fail-open flaw in Apple’s AppleMediaServices framework allows request signing to be silently disabled if a remote configuration file (the "Bag") fails to load. This affects iOS, macOS, tvOS, and watchOS.

When the Bag cannot be retrieved—due to DNS manipulation, timeouts, or network interference—AppleMediaServices daemons disable Mescal/Absinthe signing and send unsigned requests to Apple servers. These requests lack integrity protections and expose users to downgrade and replay attacks.

**Log Evidence:**

https://ia600207.us.archive.org/11/items/fail-open-log-evidence-in-apple-media-services/Fail%20Open%20Log%20Evidence%20in%20AppleMediaServices.mov


**Discovery**

* Date: August 20, 2025
* Type: Active Zero-Day
* Status: Unpatched
* CVSS (Preliminary): 9.1 Critical
* Vector: CVSS:3.1/AV\:N/AC\:L/PR\:N/UI\:N/S\:C/C\:L/I\:H/A\:N

**Affected Systems**

All Apple platforms that use `AppleMediaServices.framework` are affected.


Impacted daemons include:

* appstored (App Store services)
* amsengagementd (Media and preview endpoints)
* promotedcontentd (Advertising and personalization logic)

**Vulnerability Overview**

Apple devices fetch a dynamic configuration Bag from the following endpoint:

[https://bag.itunes.apple.com/bag.xml?deviceClass=...\&format=json](https://bag.itunes.apple.com/bag.xml?deviceClass=...&format=json)

This configuration includes flags like `useAMSMescal`, `mescalURL`, and `absintheURL`, which determine if outgoing requests must be signed.

If the Bag fails to load, AppleMediaServices logs the failure, disables signing logic, and sends unsigned requests. There is no signature validation, integrity check, or enforced fallback mechanism. The Bag is unauthenticated and unsigned, making the security state vulnerable to network interference.

**Proof of Concept**

Preconditions:

* The device must be connected to a network controlled or manipulated by an attacker (e.g., rogue access point or compromised DNS).

Exploit Steps:

1. Block or tamper with access to the Bag endpoint using DNS NXDOMAIN responses, dropped TCP handshakes, or delayed responses.
2. Observe system logs indicating the Bag failed to load and Mescal/Absinthe signatures are skipped.
3. Trigger system components (e.g., App Store, Music app) to send requests. Monitor network traffic and confirm the absence of signing headers:

   * X-Apple-Mescal-Signature
   * X-Apple-Mescal-Request-Digest
   * X-Apple-ID-Session
   * X-Apple-Absinthe-Signature

Result:

Unsigned traffic is transmitted to Apple endpoints without verification. This allows manipulation, replay, and other integrity risks.


Disclaimer:

This proof of concept was not executed against production Apple infrastructure. All observations are based on local logs and controlled network conditions. No unauthorized probing or exploitation was performed.

**Threat Models**

* Rogue public Wi-Fi access points that prevent Bag retrieval
* DNS poisoning or tampering that blocks access to `bag.itunes.apple.com`
* Frida or jailbreak modification of `AMSBagManager` to override security flags
* Replay or alteration of unsigned content requests to Apple’s CDNs

**Recommended Remediation**

1. **Signed Configuration**
   Sign the Bag using CMS, JWT, or HMAC and verify signatures client-side.

2. **Fail-Secure Defaults**
   AppleMediaServices should block signing-dependent traffic when the Bag cannot be retrieved or validated.

3. **Server Enforcement**
   Apple’s backend APIs should reject unsigned requests that require Mescal or Absinthe protection.

4. **Validated Caching**
   Bag content should only be cached when it passes integrity checks and falls within defined expiry constraints.

**Severity Justification**

* Exploitable remotely without user interaction
* No privileges required
* Affects foundational services across all major Apple platforms
* Bypasses authentication headers
* Enables replay and downgrade scenarios

CVSS Score (Preliminary): 9.1 Critical
Vector: CVSS:3.1/AV\:N/AC\:L/PR\:N/UI\:N/S\:C/C\:L/I\:H/A\:N

---
