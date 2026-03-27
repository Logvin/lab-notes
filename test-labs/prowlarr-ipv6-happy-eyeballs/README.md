# Prowlarr IPv6 Happy Eyeballs Test Lab

Reproducing and validating the fix for [Prowlarr#2157](https://github.com/Prowlarr/Prowlarr/issues/2157), where a single IPv4-only indexer permanently disables IPv6 for the entire Prowlarr process.

## Documents

- [TEST_PLAN.md](TEST_PLAN.md) - Full test plan with hypotheses, environment design, and expected outcomes

## Quick Start

The test lab Docker environment lives in the `prowlarr-ipv6-testlab/` project directory (not checked in here since it contains runtime scripts and Docker configs). See the test plan for full reproduction steps.

## Related

- **Issue:** [Prowlarr#2157](https://github.com/Prowlarr/Prowlarr/issues/2157)
- **Fix PR:** [Prowlarr#2533](https://github.com/Prowlarr/Prowlarr/pull/2533)
- **Sonarr equivalent (merged):** [Sonarr#8153](https://github.com/Sonarr/Sonarr/pull/8153)
