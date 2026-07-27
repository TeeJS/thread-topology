# PROJECT CHARTER — thread_topology / modern OTBR JSON:API

Status: **awaiting sign-off** (created 2026-07-27)

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
