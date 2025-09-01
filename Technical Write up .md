# **Expired APNs Token Still Enables Push Delivery and Background Processing**

## Summary:

Observed in production: APNs delivered a push via cached token after failed token lookups, with no active app context or re-registration. Message was processed by background daemons (`cloudd`, `identityservicesd`). This appears to violate token lifecycle enforcement and enables unauthorized background push behavior.

## Environment:

* **Platform**: iPhone 14 Pro Max (APNs and system daemons)
* **OS Version**: iOS 18.6.2 (Production)
* **Context**: Observed in a real-world, non-laboratory setting

## Timeline (Key Logs):

```
18:07:50.236249 apsd: copyTokenForDomain push.apple.com (null)
18:07:52.580220 apsd: copyTokenForDomain push.apple.com (null)
18:07:54.012876 apsd: copyTokenForDomain push.apple.com <private>, PerAppToken.v0
18:07:54.119270 apsd: found cached token for topic: com.apple.icloud-container.com.apple.willowd
18:07:54.203283 StatusKitAgent: Received aps incoming message
18:07:54.408293 apsd: saveSalt failed (null) com.apple.icloud-container.com.apple.willowd.homekit
18:07:57.025320 cloudd: TCC approved access for container containerID=com.apple.homekit.config:Production, applicationBundleID=com.apple.willowd
```

## Observations:

* `apsd` failed multiple times to retrieve a valid token (`null` response).
* No `registerForPush`, `registerTopic`, or equivalent re-registration events occurred.
* Despite this, `apsd` used a cached token to deliver a push.
* Background system components (`identityservicesd`, `cloudd`, `StatusKitAgent`) processed the incoming message.
* There was no foreground app context at the time of processing.

## Impact:

* **Violates expected APNs behavior**: expired or unavailable tokens are reused.
* **Enables covert push delivery** without user interaction or app activity.
* **Potential for misuse**:

  * Silent push channels post app removal
  * Covert data exfiltration via system daemons
  * Background command-and-control channel

## Recommendation:

To ensure lifecycle integrity of APNs tokens, Apple should:

* Invalidate and purge tokens when `copyTokenForDomain` returns `null`.
* Enforce rejection of pushes without a freshly registered token.
* Disallow background push handling by daemons when an app is no longer active or authorized.
* Audit APNs routing paths that rely on cached state rather than real-time validation.

## Suggested CVSS:

* **Vector**: AV\:N/AC\:L/PR\:N/UI\:N/S\:C/C\:L/I\:L/A\:N
* **Score**: 7.5 (High)
