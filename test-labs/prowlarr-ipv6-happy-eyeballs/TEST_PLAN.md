# Prowlarr IPv6 Happy Eyeballs Test Plan

## Related

- **Issue:** [Prowlarr#2157](https://github.com/Prowlarr/Prowlarr/issues/2157) - IPv6 permanently disabled after single IPv4-only connection
- **Fix PR:** [Prowlarr#2533](https://github.com/Prowlarr/Prowlarr/pull/2533) - Use Happy Eyeballs for HTTP socket address selection
- **Sonarr equivalent:** [Sonarr#8153](https://github.com/Sonarr/Sonarr/pull/8153) - Same fix, merged January 2026
- **Upstream .NET issue:** [dotnet/runtime#26177](https://github.com/dotnet/runtime/issues/26177) - .NET lacks native Happy Eyeballs

---

## 1. Background

Prowlarr's HTTP dispatcher (`ManagedHttpDispatcher.cs`) contains a static boolean flag `useIPv6` that controls whether IPv6 connections are attempted. When the application starts, this flag is set to `true` if the OS reports IPv6 support (`Socket.OSSupportsIPv6`).

On the first outbound connection attempt, Prowlarr tries IPv6 with a 2-second timeout. If IPv6 fails and the system has a routable IPv4 address, the flag is permanently set to `false` for the lifetime of the process. Every subsequent connection skips IPv6 entirely, regardless of destination.

This is a process-wide, one-way kill switch. It cannot be toggled by the user. It resets only on process restart.

### Why this matters

- An indexer that simply lacks AAAA DNS records (normal, not broken) triggers the global disable.
- IPv6-only indexers and services become unreachable for the rest of the session.
- FlareSolverr users experience cookie invalidation because cookies are bound to the IPv6 address, but Prowlarr switches to IPv4 mid-session.
- Kubernetes dual-stack clusters and IPv6-only environments are disproportionately affected.

### The proposed fix

PR #2533 replaces the global flag with a proper implementation of the **Happy Eyeballs algorithm (RFC 8305)**. This is the industry standard for dual-stack connection handling, used by curl, Chrome, Firefox, Safari, Jellyfin, and soon .NET 11 natively. It works by:

1. Resolving both A and AAAA DNS records
2. Interleaving addresses: IPv6, IPv4, IPv6, IPv4...
3. Racing connection attempts with a 250ms delay between each
4. First successful connection wins; all others are cancelled
5. Never globally disabling either protocol

---

## 2. Hypothesis

### H1: Global IPv6 Disable (Baseline)

> If Prowlarr (current develop branch) connects to an IPv4-only host before connecting to an IPv6-only host, IPv6 will be globally disabled for the process, and the IPv6-only host will be unreachable.

### H2: Happy Eyeballs Fix (PR #2533)

> If Prowlarr (with PR #2533 applied) connects to an IPv4-only host before connecting to an IPv6-only host, IPv6 will NOT be globally disabled, and the IPv6-only host will remain reachable.

### H3: Per-Connection Racing

> With PR #2533, a dual-stack host will receive connections via whichever address family responds first, independent of previous connection outcomes to other hosts.

### H4: Connection Attempt Delay

> With PR #2533, if IPv6 is reachable but slow (>250ms), the algorithm will fall back to IPv4 for that specific host without affecting IPv6 availability for other hosts.

---

## 3. Test Environment

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Docker Compose Stack (dual-stack network)                      │
│                                                                 │
│  ┌──────────┐    DNS     ┌──────────────────────────────────┐   │
│  │ dnsmasq  │◄───────────│         Prowlarr                 │   │
│  │ (DNS)    │            │    (develop or PR #2533)          │   │
│  └──────────┘            └──┬──────┬──────┬──────┬──────────┘   │
│                             │      │      │      │              │
│                     ┌───────┘      │      │      └───────┐      │
│                     ▼              ▼      ▼              ▼      │
│              ┌────────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐  │
│              │ v4only     │ │ v6only   │ │ dualstack│ │v6slow│  │
│              │ (A only)   │ │(AAAA only│ │ (A+AAAA) │ │(slow)│  │
│              └────────────┘ └──────────┘ └──────────┘ └──────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ tcpdump sidecar (captures Prowlarr's network traffic)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Image | Role | Network Config |
|-----------|-------|------|----------------|
| `dnsmasq` | `jpillora/dnsmasq` | Controlled DNS. Returns specific A/AAAA records per mock indexer hostname. | Dual-stack |
| `indexer-v4only` | Custom Python | Mock Torznab indexer. Bound to IPv4 only. DNS returns A record only. | IPv4 only |
| `indexer-v6only` | Custom Python | Mock Torznab indexer. Bound to dual-stack socket. DNS returns AAAA record only. | Dual-stack |
| `indexer-dualstack` | Custom Python | Mock Torznab indexer. DNS returns both A and AAAA records. | Dual-stack |
| `indexer-v6slow` | Custom Python | Mock Torznab indexer with 500ms artificial IPv6 latency via `tc netem`. | Dual-stack |
| `prowlarr` | `linuxserver/prowlarr:develop` | The application under test. | Dual-stack |
| `tcpdump` | `nicolaka/netshoot` | Packet capture sidecar sharing Prowlarr's network namespace. | Shares Prowlarr's network |

### Network

- **IPv4 subnet:** `172.30.0.0/24`
- **IPv6 subnet:** `fd00:dead:beef::/64`
- Docker's `enable_ipv6: true` flag is set on the bridge network.
- Prowlarr's DNS is pointed at the dnsmasq container.

### Mock Indexer Design

Each mock indexer is a Python HTTP server that:

1. Responds to Torznab API calls (`?t=caps`, `?t=search`, etc.) with valid XML
2. Logs every incoming request with:
   - Timestamp (UTC)
   - Client IP address
   - Address family classification (`IPv4`, `IPv6`, or `IPv4-mapped-IPv6`)
   - Requested action
3. Writes structured JSON logs to `/tmp/indexer-access.log` inside the container

The address family classification is the primary observable. If a client connects from an IPv4 address, it logs `IPv4`. If from a native IPv6 address, it logs `IPv6`. If from an IPv4-mapped IPv6 address (`::ffff:x.x.x.x`), it logs `IPv4-mapped-IPv6` (which indicates the client used IPv4 but the server is on a dual-stack socket).

### DNS Records

| Hostname | A Record | AAAA Record | Purpose |
|----------|----------|-------------|---------|
| `indexer-v4only.testlab.local` | `172.30.0.11` | (none) | Forces IPv4. Triggers the global disable in baseline. |
| `indexer-v6only.testlab.local` | (none) | `fd00:dead:beef::12` | Only reachable via IPv6. Canary for the global disable. |
| `indexer-dualstack.testlab.local` | `172.30.0.10` | `fd00:dead:beef::10` | Both available. Shows which family Prowlarr chooses. |
| `indexer-v6slow.testlab.local` | `172.30.0.13` | `fd00:dead:beef::13` | Both available, but IPv6 has 500ms added latency. |

---

## 4. Variables

### Independent Variable

The Prowlarr build under test:

- **Baseline:** Current `develop` branch (contains the global `useIPv6` static flag)
- **Fixed:** PR #2533 branch (contains the Happy Eyeballs implementation)

### Dependent Variables

| Variable | How Measured | Source |
|----------|-------------|--------|
| Address family per connection | `address_family` field in indexer access logs | Mock indexer JSON logs |
| Whether v6-only indexer is reachable | Presence or absence of log entries for `indexer-v6only` | Mock indexer JSON logs |
| Address family chosen for dual-stack hosts | `address_family` in `indexer-dualstack` logs | Mock indexer JSON logs |
| Connection-level packet data | Source/destination IP per TCP SYN | tcpdump pcap files |
| Prowlarr's internal connection logging | Debug-level log entries | Prowlarr application logs |

### Controlled Variables

- All tests use the same Docker network configuration
- Prowlarr is restarted between tests to reset the static `useIPv6` flag (baseline only; the flag doesn't exist in the fix)
- Indexer access logs are cleared between tests
- Indexer priority order is set via Prowlarr API to control which indexer is contacted first
- The same search queries and Torznab responses are used across all scenarios

---

## 5. Test Scenarios

### Test 1: IPv6 Global Disable After IPv4-Only Connection

**Purpose:** Prove that connecting to an IPv4-only host disables IPv6 for all subsequent connections.

**Preconditions:**
- Prowlarr freshly started (static `useIPv6 = true`)
- `indexer-v4only` configured as highest priority (contacted first)
- `indexer-v6only` configured as lower priority (contacted second)

**Procedure:**
1. Restart Prowlarr to reset the `useIPv6` static flag
2. Trigger a search via Prowlarr API (`/api/v1/search?query=...&type=search`)
3. Prowlarr contacts all configured indexers in priority order
4. Wait 5 seconds for all connections to complete
5. Collect access logs from all mock indexers

**Expected Result (Baseline):**
- `indexer-v4only` log shows connections (IPv4)
- `indexer-v6only` log shows **zero connections** (IPv6 was globally disabled after v4only was hit)
- This confirms H1

**Expected Result (Fixed):**
- `indexer-v4only` log shows connections (IPv4)
- `indexer-v6only` log shows connections (IPv6)
- This confirms H2

**Pass Criteria (Baseline):** Test FAILS (confirming the bug exists). `indexer-v6only` receives no connections.
**Pass Criteria (Fixed):** Test PASSES. `indexer-v6only` receives connections.

---

### Test 2: IPv6 Succeeds First, Then IPv4

**Purpose:** Verify that when IPv6 succeeds on the first connection, subsequent IPv4-only hosts still work.

**Preconditions:**
- Prowlarr freshly started
- `indexer-v6only` configured as highest priority (contacted first)
- `indexer-v4only` configured as lower priority (contacted second)

**Procedure:**
1. Swap indexer priorities via Prowlarr API so v6only has the highest priority
2. Restart Prowlarr to reset state
3. Trigger a search
4. Wait 5 seconds
5. Collect access logs

**Expected Result (Both Baseline and Fixed):**
- `indexer-v6only` log shows IPv6 connections
- `indexer-v4only` log shows IPv4 connections
- Both indexers are reachable because IPv6 succeeded first (the flag was never flipped in baseline)

**Pass Criteria:** Both indexers receive connections.

---

### Test 3: Dual-Stack Address Family After IPv4-Only Hit

**Purpose:** Determine which address family Prowlarr uses for a dual-stack host after IPv6 has been globally disabled.

**Preconditions:**
- Prowlarr freshly started
- `indexer-v4only` configured as highest priority
- `indexer-dualstack` configured as lower priority

**Procedure:**
1. Restore original priorities (v4only first, dualstack second)
2. Restart Prowlarr
3. Trigger a search
4. Wait 5 seconds
5. Collect access logs from `indexer-dualstack`
6. Inspect the `address_family` field

**Expected Result (Baseline):**
- `indexer-dualstack` shows only `IPv4` or `IPv4-mapped-IPv6` connections
- Even though the host has an AAAA record, Prowlarr skips it because `useIPv6 = false`

**Expected Result (Fixed):**
- `indexer-dualstack` may show `IPv6` connections (Happy Eyeballs races both and IPv6 typically wins on a local network)

**Pass Criteria (Baseline):** Test FAILS (confirming the bug). Dual-stack host forced to IPv4.
**Pass Criteria (Fixed):** Test PASSES. Dual-stack host receives IPv6 connections.

---

### Test 4: Slow IPv6 / Fast IPv4 (Connection Attempt Delay)

**Purpose:** Verify that the Happy Eyeballs 250ms Connection Attempt Delay correctly falls back to IPv4 when IPv6 is slow, without affecting other hosts.

**Preconditions:**
- Prowlarr freshly started
- `indexer-v6slow` has 500ms artificial latency on IPv6 traffic (via `tc netem`)
- Other indexers have no artificial latency

**Procedure:**
1. Apply `tc netem` latency rules to the `indexer-v6slow` container
2. Restart Prowlarr
3. Trigger a search
4. Wait 5 seconds
5. Collect access logs from `indexer-v6slow` and other indexers

**Expected Result (Baseline):**
- `indexer-v6slow` triggers IPv6 timeout, which globally disables IPv6 for everything else
- Same cascading failure as Test 1

**Expected Result (Fixed):**
- `indexer-v6slow` receives an IPv4 connection (Happy Eyeballs falls back after 250ms)
- Other indexers with working IPv6 still receive IPv6 connections
- This confirms H4: per-host fallback without global side effects

**Pass Criteria (Fixed):** `indexer-v6slow` shows IPv4 connections AND other dual-stack indexers still show IPv6 connections.

---

### Test 5: IPv6-Only Environment

**Purpose:** Verify behavior when no routable IPv4 address exists.

**Note:** This test requires a modified Docker Compose configuration that removes IPv4 from the Prowlarr container. It is documented here for completeness but marked as a manual test.

**Preconditions:**
- Prowlarr container has only an IPv6 address (no routable IPv4)
- All indexers have AAAA records

**Expected Result (Both Baseline and Fixed):**
- All connections use IPv6
- In baseline, the `HasRoutableIPv4Address()` check returns `false`, so the global flag stays `true`
- In the fixed version, Happy Eyeballs naturally selects IPv6 as the only available option

**Pass Criteria:** All indexers receive IPv6 connections.

---

### Test 6: IPv6 Recovery After Restart

**Purpose:** Confirm that restarting Prowlarr re-enables IPv6 after it was globally disabled.

**Preconditions:**
- Prowlarr freshly started
- `indexer-v4only` is highest priority (triggers the global disable)

**Procedure:**
1. Trigger a search to disable IPv6 (v4only hit first)
2. Collect `indexer-v6only` logs (expect: no connections)
3. Clear all indexer logs
4. Restart Prowlarr
5. Trigger another search
6. Collect `indexer-v6only` logs again

**Expected Result (Baseline):**
- Before restart: `indexer-v6only` has zero connections (IPv6 disabled)
- After restart: `indexer-v6only` has connections (static flag reset to `Socket.OSSupportsIPv6`)
- This confirms the disable is runtime-only, not persisted to config

**Expected Result (Fixed):**
- Both before and after restart: `indexer-v6only` has connections (no global flag to disable)

**Pass Criteria (Baseline):** v6-only indexer is unreachable before restart, reachable after.
**Pass Criteria (Fixed):** v6-only indexer is reachable in both phases.

---

## 6. Procedure

### Prerequisites

- Docker and Docker Compose (v2+)
- `git` for cloning the test lab
- A system with IPv6 support enabled (check: `sysctl net.ipv6.conf.all.disable_ipv6` should return `0` on Linux, or verify via `ifconfig` on macOS)
- Port 9696 available (Prowlarr web UI)

### Running the Baseline

```bash
# Clone and enter the test lab
cd prowlarr-ipv6-testlab

# Launch the full stack and run the baseline test suite
./launch.sh baseline
```

The launch script will:
1. Build the mock indexer Docker image
2. Start all containers (DNS, 4 indexers, Prowlarr develop, tcpdump)
3. Verify DNS resolution from the Prowlarr container
4. Verify mock indexers are responding
5. Configure all indexers in Prowlarr via API
6. Run all 6 test scenarios with restarts between each
7. Output a results summary

### Running the Fix

To test PR #2533, modify `docker-compose.yml` to build Prowlarr from the PR branch instead of using the `linuxserver/prowlarr:develop` image. Then:

```bash
./launch.sh fixed
```

### Comparing Results

```bash
# Analyze a specific run
./scripts/check-results.sh results/baseline-YYYYMMDD-HHMMSS/
./scripts/check-results.sh results/fixed-YYYYMMDD-HHMMSS/
```

### Cleanup

```bash
./scripts/teardown.sh
```

---

## 7. Recording Results

Each test run creates a timestamped directory under `results/` containing:

| File | Contents |
|------|----------|
| `metadata.json` | Run label, timestamp, Prowlarr version |
| `results.csv` | Pass/fail/skip/info per test scenario |
| `testN-indexer-*.log` | Mock indexer access logs (JSON, one entry per connection) |
| `testN-prowlarr.log` | Prowlarr application logs |
| `*.pcap` | Packet captures from tcpdump sidecar |

### Access Log Schema

Each line in the indexer access logs is a JSON object:

```json
{
    "timestamp": "2026-03-27T12:00:00.000000+00:00",
    "indexer": "v4only-indexer",
    "client_ip": "172.30.0.100",
    "address_family": "IPv4",
    "action": "caps",
    "path": "/api?t=caps&apikey=testlab-mock-key"
}
```

The `address_family` field is the primary observable:

| Value | Meaning |
|-------|---------|
| `IPv4` | Client connected via IPv4 |
| `IPv6` | Client connected via native IPv6 |
| `IPv4-mapped-IPv6` | Client used IPv4, server is on a dual-stack socket (`::ffff:x.x.x.x`) |

---

## 8. Expected Outcome Summary

| Test | Baseline (develop) | Fixed (PR #2533) |
|------|-------------------|------------------|
| 1. v4-only then v6-only | **FAIL** - v6-only unreachable | **PASS** - both reachable |
| 2. v6-only then v4-only | PASS - both reachable | PASS - both reachable |
| 3. Dual-stack after v4-only hit | **FAIL** - forced to IPv4 | **PASS** - IPv6 available |
| 4. Slow IPv6 fallback | Global disable triggered | Per-host IPv4 fallback only |
| 5. IPv6-only environment | PASS (no IPv4 to fall back to) | PASS |
| 6. Recovery after restart | PASS - restart resets flag | PASS - no flag to reset |

If baseline results match the "Baseline" column, hypotheses H1 is confirmed and the bug is proven.
If fixed results match the "Fixed" column, hypotheses H2, H3, and H4 are confirmed and the fix is validated.
