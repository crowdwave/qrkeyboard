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

The [starter](examples/index.html) explicitly requests the keyboard and handles
live `key` events, so it uses the earlier API. With an earlier deployment, the
phone may say **Connect** and require tapping **keyboard**. The new flow says
**Start typing**. The starter's `finish` listener takes effect after the new
service is deployed; it does not provide cleanup on the old service.

Message and presentation examples describe the new release. Deploy the Worker,
SDK, and phone assets together before presenting those examples as ready on the
hosted service. Hosting your own receiving page does not update the service.

Maintainers should update this file with the deployed version/date and actual
live checks after rollout. Also remove the pending-release notes from the README
and quick start once those checks pass. Do not change status merely because local
tests passed. Service rollout instructions are in the private implementation
repository; application developers only need [Deploy your page](DEPLOY.md).
