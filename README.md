# Ghost Push: Background Delivery via Expired APNs Tokens

## Overview

This repository documents a vulnerability observed in **iOS 18.6.2**, where **expired or unavailable Apple Push Notification service (APNs) tokens** are still reused by system daemons to deliver silent push messages — even without an active app context or token re-registration.

This violates the expected APNs token lifecycle and enables unauthorized background processing. It introduces risks such as covert communication, post-removal message delivery, and potential data leakage.

---

## Affected Platform

* **Operating System**: iOS 18.6.2 (Production)
* **Components**: `apsd`, `cloudd`, `identityservicesd`, `StatusKitAgent`
* **Context**: Observed in a real-world, non-laboratory production environment

---

## Technical Summary

* The system repeatedly invoked `copyTokenForDomain` to retrieve a push token, but received `null` responses.
* No evidence of `registerForPush`, `registerTopic`, or re-registration events.
* Despite this, `apsd` retrieved a previously cached token and delivered a push message.
* System daemons processed the push, even though:

  * The corresponding app was not active
  * The app had not freshly registered for push notifications
* This behavior bypasses lifecycle validation checks and introduces a persistent, unauthorized communication path.

---

## Timeline (Sample Logs)

```
18:07:50.236249 apsd: copyTokenForDomain push.apple.com (null)
18:07:54.012876 apsd: copyTokenForDomain push.apple.com <private>, PerAppToken.v0
18:07:54.119270 apsd: found cached token for topic: com.apple.icloud-container.com.apple.willowd
18:07:57.025320 cloudd: TCC approved access for container com.apple.homekit.config
```
**Log Evidence**: https://ia601600.us.archive.org/28/items/ghost-push/Ghost%20Push.mov

---

## Impact

* Violation of expected APNs token lifecycle enforcement
* Silent push delivery without user interaction or app activity
* Persistence of push capability even after app removal
* Potential for covert communication, data exfiltration, or command-and-control (C2) channels via system daemons

---

## Recommendations

Apple should consider the following mitigations:

* Invalidate cached tokens when `copyTokenForDomain` returns `null`
* Enforce rejection of push messages that lack a recently registered token
* Prevent background system daemons from processing pushes associated with unregistered or removed applications
* Audit APNs routing logic to ensure real-time token validation and eliminate reliance on stale cached data

---

## Suggested CVSS

* **Vector**: AV\:N/AC\:L/PR\:N/UI\:N/S\:C/C\:L/I\:L/A\:N
* **Base Score**: 7.5 (High)

---

## References

* Apple Push Notification Service (APNs) Documentation
* Affected system daemons: `apsd`, `cloudd`, `identityservicesd`, `StatusKitAgent`

---

## Disclaimer

This repository is provided for educational and security research purposes only. Findings should be responsibly disclosed to Apple or relevant platform maintainers through official channels.

---


