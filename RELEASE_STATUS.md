# Release status

Status recorded **2026-10-11**: the simplified implementation is in the local
checkout and has not been deployed. The hosted URL is `https://qrkeyboard.com`;
its reachability/version could not be verified during this documentation review.

| Behaviour | Checkout | Hosted deployment status |
|---|---|---|
| Live `key` events with `caps: ['keyboard']` | Implemented | Existing integration API; current availability not reverified |
| Optional PIN pairing | Implemented | Present in the previous implementation |
| Keyboard-only default; Start typing opens a visible editor | Implemented | Updated flow pending deployment |
| Compose-and-send `text` mode | Implemented | Pending deployment |
| Named `action` buttons | Implemented | Pending deployment |
| Application delivery acknowledgements | Implemented | Pending deployment |
| Finish, old-link invalidation, and fresh visitor sessions | Implemented | Pending deployment |
| Protocol 2: per-connection salts, counter nonces, replay rejection | Implemented | Pending deployment |
| PIN attempts counted when pairing starts | Implemented | Pending deployment |
| Dropped phone keeps its place for 5 minutes; Disconnect releases it | Implemented | Pending deployment |
| Phone asks older screens to update | Implemented | Pending deployment |
| Versioned SDK files with SRI (`/sdk/manifest.json`) | Implemented | Pending deployment |
| Message-length budget shown while typing | Implemented | Pending deployment |
| Server QR and provisioning QR only for PIN instances | Implemented | Pending deployment |
| Protocol 3 controls: `layout`, remote / teletext / gamepad presets, key down/up | Implemented | Pending deployment |

The [starter](examples/index.html) explicitly requests the keyboard and handles
live `key` events, so it uses the earlier API. With an earlier deployment, the
phone may say **Connect** and require tapping **keyboard**. The new flow says
**Start typing**. The starter's `finish` listener takes effect after the new
service is deployed; it does not provide cleanup on the old service.

Message and presentation examples describe the new release. Deploy the Worker,
SDK, and phone assets together before presenting those examples as ready on the
hosted service. Protocol 2 is not compatible with the deployed protocol: once it
ships, screens still running the old SDK are asked to refresh by phones that
scan them, and reloading the page picks up the new SDK. Hosting your own receiving page does not update the service.

Maintainers should update this file with the deployed version/date and actual
live checks after rollout. Also remove the pending-release notes from the README
and quick start once those checks pass. Do not change status merely because local
tests passed. Service rollout instructions are in the private implementation
repository; application developers only need [Deploy your page](DEPLOY.md).
