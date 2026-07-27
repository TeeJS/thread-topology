# PROJECT CHARTER — thread_topology / modern OTBR JSON:API

Status: **delivered and verified in production** (created and completed 2026-07-27)

## 1. What is the one thing this must do?

Home Assistant must expose **live Thread link-quality sensor entities** — one per
mesh node, whose *state* is that node's LQI (0–3) — sourced from the OTBR at
`192.168.1.144:8081`, updating on a schedule, usable in automations.

## 2. What would be wrong if we shipped "working" software without it?

The integration loading without error, entities appearing, and every
`link_quality` reading **0** — because the JSON:API translator we are porting
from the JS visualizer never emits `Connectivity`. Green config entry, dead
sensors. That is the specific failure mode this charter exists to prevent.

Second failure of the same kind: the network sensor showing
`network_name: Unknown`, `router_count: 0`, and no node flagged as leader,
because `/node` now returns camelCase and `_process_topology` reads PascalCase.

## 3. What is explicitly off-limits as a workaround?

- Shipping with `link_quality` hardcoded, defaulted, or derived from anything
  other than real OTBR diagnostic data.
- Dropping the legacy `/diagnostics` path. Auto-detect must keep working on
  older OTBR builds.
- Requiring the user to hand-edit files inside the HA container to deploy.
  Deployment is HACS.
- Widening scope into the SVG generator, sensor platform, or Matter matching.
  The change is scoped to the **fetch layer + response key handling**.

## 4. Deployment target and backup location

- **Target:** HA Container `core-2026.7.4` at `192.168.1.25`, integration
  installed via **HACS** from `TeeJS/thread-topology` (currently pinned to
  commit `ddd858c`, tracking default branch). Deploy = commit → push → HACS
  update → restart HA.
- **Backup:** source is covered by git in this repo. Before the HACS update +
  restart, take a **full HA backup** so the live `/config` state is recoverable.

## 5. How will we verify it is done?

1. Config entry `01KSBAQ2H0JE4QWQGR899FYM0B` reaches state `loaded`
   (it has never once loaded since it was created 2026-05-23).
2. `sensor.thread_network` reports `network_name = ha-thread-0d68`,
   `router_count = 2` (not `Unknown` / `0`).
3. At least one `ThreadNodeSensor` exists with a **non-zero** numeric LQI state.
4. Exactly one node is flagged `role: leader`; others `router` / `end_device` —
   not everything collapsing to `end_device`.
5. Update cycles complete without overlapping (task flow measured ~48 s).
6. `pytest` green, including new tests driven by a real captured OTBR response.

---

## Outcome (2026-07-27)

Delivered in `TeeJS/thread-topology` PR #2, merged as `622d30a`, deployed via
HACS (`ddd858c` -> `622d30a`) after a full HA backup, and verified live.

| Criterion | Result |
|---|---|
| 1. Entry `loaded` | Yes - first successful load since it was created |
| 2. Network sensor | `ha-thread-0d68`, `router_count: 2` |
| 3. Non-zero LQI | Both node sensors report `3` |
| 4. One leader | `6a57f823187e197b`, the other node a `router` |
| 5. No overlapping updates | Scan interval auto-raised 30s -> 180s |
| 6. Tests green | 130 passing |

The failure mode named in section 2 was real and would have shipped: the JS
reference implementation this was ported from never requests the `mode` or
`connectivity` diagnostic TLVs, because the visualizer draws edges from route
data instead. Porting it faithfully would have produced loaded entities
reporting LQI 0 for every node. `/node` had also silently moved to camelCase
and renamed `NumOfRouter` to `routerCount`, which had been breaking leader
detection, network name and router count independently of the 404.

Also fixed along the way: the polled border router now honours
`custom_routers.yaml` instead of being hardcoded to "SkyConnect (OTBR)", and
the SVG write moved off the event loop.

### Fixed after the first deployment

Restarting Home Assistant put the entry into `setup_retry` with
`list index out of range`. `get_matter()` indexes into `hass.data["matter"]`,
which is still empty if the Matter integration has not finished setting up, and
`_get_matter_devices` caught `KeyError, StopIteration, AttributeError,
ImportError` but not `IndexError` - so a race over what is only optional
enrichment failed the entire update. It self-healed on retry, but recurred
whenever Matter lost the race. `IndexError` is now caught, with a test that
reproduces the original failure.

### Known follow-ups, deliberately out of scope

- `_match_end_device` still matches children positionally. The JSON:API
  `children[]` array carries each child's real extended address, which would
  make that matching exact.
- Home Assistant knows 13 Matter Thread devices while the mesh crawl reports
  3 nodes. Unexplained; may need the `deviceCount` / `maxAge` crawl parameters
  revisited, or those devices may simply be offline.
