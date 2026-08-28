# 0002 — Agent → tenant binding via `--cloud-account`

- **Status:** Accepted (2026-08-24) — **agent half implemented in this repo
  (2026-08-28), shipped as an EXPERIMENTAL feature.** Cloud-server + `common`
  half was already done in `screenguard-cloud`.
- **Date:** 2026-08-24 (updated 2026-08-28)
- **Follows:** [0001](0001-multi-tenancy.md) (schema-per-tenant multi-tenancy)

> ⚠️ **Experimental.** Cloud mode (`--cloud-account`) is unfinished and unsupported.
> The wire field, the endpoint, and the enrolment UX may all change. With **no**
> `--cloud-account` the agent behaves exactly as it always has — that path is not
> affected by anything in this document.

> This entry is half decision record, half **porting checklist for whoever works
> in the community agent repo** [`adambie/screenguard`](https://github.com/adambie/screenguard).
> The cloud server already routes agents by tenant; the agent just needs to *tell*
> it which tenant, and only when the user opted into cloud mode.

---

## Context

ADR 0001 made the cloud server multi-tenant (one Postgres schema per tenant). An
agent must belong to exactly one tenant. The mechanism decided in 0001 §5: the
agent carries a **`cloud_account` email**, and the server maps it to a tenant.

**Already done in the private `screenguard-cloud` repo** (not this one — this is
the community repo, whose bundled `crates/server` is the single-tenant edition
and needs no change):
- `common`: `PairingRequest` and `AgentHello` each gained an optional
  `cloud_account: Option<String>`, `#[serde(default, skip_serializing_if = "Option::is_none")]`.
- `server/src/ws.rs`: `resolve_tenant()` reads `cloud_account` on the first frame
  (pairing or hello), maps it via the control-plane `public.accounts` table to a
  tenant + its schema-scoped pool, and scopes the whole connection to it. No
  `cloud_account` (or empty) ⇒ `DEFAULT_TENANT` / `public` — the existing
  single-tenant path, unchanged.

**Verified end-to-end on real Postgres** with a scripted agent: cloud agent pairs
into its tenant's schema (visible only to that tenant's admin); a self-hosted agent
with no `cloud_account` pairs into `public` exactly as before; reconnect via
`agent_hello` + `cloud_account` finds the agent in the right schema.

## Hard requirement: **backwards compatibility**

Self-hosted / community users must see **zero behavioural change**. Concretely:
- With **no** `--cloud-account`, the agent must do exactly what it does today
  (mDNS discovery or a hardcoded local URL) and must **not** put `cloud_account`
  on the wire at all (the `skip_serializing_if` on the server side already does
  this; the agent side must mirror it so the field is simply absent).
- The cloud path is strictly **opt-in** via the new flag.

## What was implemented in the agent (2026-08-28)

### 1. `common` wire field — done
`crates/common/src/messages.rs`: `PairingRequest` and `AgentHello` each gained

```rust
#[serde(default, skip_serializing_if = "Option::is_none")]
pub cloud_account: Option<String>,
```

`common` bumped `0.10.1 → 0.10.2`. **Only this field was mirrored** — this repo's
`PairingRequest` is `{machine_id, hostname, pairing_code}` and has no
`agent_version`, so the structs are *not* byte-identical to `screenguard-cloud`;
the version bump tracks the new field, not whole-struct parity. `skip_serializing_if`
means a self-hosted agent still puts nothing on the wire, and an old server
ignores the field if a cloud agent ever talks to one.

### 2. `--cloud-account <email>` flag + config — done
- CLI: `--cloud-account <email>` or `--cloud-account=<email>` (hand-parsed;
  `main.rs::arg_value`). Also `--server-url <url>` / `--server-url=<url>`.
- Config: `cloud_account` and `cloud_url` keys in `agent.toml`.
- Precedence: CLI `--cloud-account` > `agent.toml` `cloud_account`. The value is
  **trimmed only** — case is preserved on disk and on the wire (the cloud
  account lookup may or may not fold case; don't assume). Equality checks for
  re-bind detection (§5) fold case via `main.rs::account_key`.
- Rejected at startup if it has no `@` (it names a tenant, not a host).
- **Persisted in the agent DB** (`server_connection.cloud_account`). The value
  put on every `agent_hello` is `effective_account = CLI/config account, else the
  account saved with the pairing` — so a plain `systemctl restart` (unit file
  carries no flag, `agent.toml` maybe not edited) still reconnects into the right
  tenant instead of silently falling back to `public`.

### 3. Server selection — done
- `cloud_account` set ⇒ mDNS/local discovery is **skipped**; endpoint is
  `--server-url` > `agent.toml` `cloud_url` > compiled-in
  `config::DEFAULT_CLOUD_URL` (`wss://api.screenguard.cc/ws`).
- `cloud_account` unset ⇒ `discovery::resolve_server_url` exactly as before.

### 4. `cloud_account` on outbound frames — done
Threaded into `pairing::run_pairing` (on `pairing_request`) and held on
`HeartbeatLoop` so every `agent_hello` carries it (including the re-send after a
`config_reload`).

### 5. Re-bind on account change — **decided: wipe + re-pair, but only on an explicit *different* account**
`main.rs::should_rebind(paired, stored, requested)` — the agent wipes and
re-pairs only when it is already paired **and** an explicit `--cloud-account`
(or `agent.toml` value) is present **and** it differs (case-folded) from the
account stored with the pairing.

| paired | stored | requested (CLI/config) | result |
|---|---|---|---|
| no  | –     | anything            | no wipe — normal first-time pairing |
| yes | `a@x` | `a@x` (any case/space) | no wipe — normal reconnect |
| yes | `a@x` | *absent*            | no wipe — reconnect; `effective_account` falls back to `a@x` |
| yes | none  | *absent*            | no wipe — plain local agent |
| yes | `a@x` | `b@y`               | **wipe**, pair into `b@y` |
| yes | none  | `a@x`               | **wipe**, pair into cloud tenant `a@x` |

- **Absence never wipes.** The agent runs under systemd and the unit file has no
  `--cloud-account`; if "flag dropped ⇒ wipe" were the rule, every `systemctl
  restart` without a matching `agent.toml` edit would destroy a live cloud
  pairing and drop the machine to unmanaged. Leaving cloud mode is therefore an
  explicit `--reset`, exactly like moving to a different server.
- **Why account-only (not URL):** comparing the resolved server URL would force
  mDNS on every startup (the reconnect path never runs discovery today) and would
  make a DHCP move of a LAN server look like a re-bind. Server-URL moves stay
  served by `--reset`. Agents that never set an account never reach this branch.
- **Why wipe, not refuse:** non-interactive under systemd ⇒ the only options are
  auto-wipe or fail-to-start. Pairing still needs tenant-admin approval, so an
  accidental switch leaks nothing — it lands in a pending queue. A
  `tracing::warn!` names old → new and says the machine is now unmanaged.
- `Db::wipe_for_rebind()` clears `server_connection`, `config_meta` (else a stale
  `last_config_version` suppresses the new tenant's first `config_push` —
  `server/src/ws.rs` only pushes when `hello.last_config_version < server_version`),
  `managed_users` + all `cached_*`, `usage_log` (un-synced rows would upload into
  the wrong tenant), `server_remaining`, `active_sessions`; keeps
  `agent_capabilities` (describes the machine). `reset_pairing()` (`--reset`) is
  left narrow and unchanged — a regression test guards this.
- **Consequences to know:** the wipe empties `managed_users`, so
  `get_managed_uids()` returns nothing and **the machine enforces nothing** until
  the new account approves it and the first `config_push` lands — whereas a
  self-hosted agent that merely loses its server keeps enforcing cached rules for
  `cache_ttl_hours`. Today's usage counters also reset to zero, so the managed
  user gets a fresh allowance on the day of the switch.

## Known limitations (experimental)
- **No feedback for a wrong / unknown account.** If the email doesn't match any
  cloud tenant the agent just stays *pending* forever with no error. This is
  deliberate for now — it means the enrolment endpoint can't be used to probe
  which emails have accounts (no account-enumeration oracle). The cost is a
  silent failure on a typo'd address. A later, non-experimental design would
  replace the bare email with an authenticated enrolment token issued to a
  logged-in account, which removes both the oracle and the silent-typo problem.
- Enrolment still trusts agent-supplied `hostname` / `machine_id`. Hardening this
  (operator must type the console pairing code to accept) is tracked in
  [issue #9](https://github.com/adambie/screenguard/issues/9).

## How to test
1. `agent --cloud-account you@example.com --server-url ws://<host>:8080` against a
   server where `you@example.com` signed up. Agent should appear *pending* in that
   tenant only; accept → pairing completes → reconnect → online.
2. Same agent with **no** `--cloud-account` on a LAN with a community server — it
   must discover and pair exactly as before.
3. Pair locally, then restart with `--cloud-account a@x` — agent logs the
   re-bind, wipes, and re-pairs against the cloud. Restart again with **no** flag
   — it must **not** wipe; it reconnects to `a@x` (from the saved binding).
   `screenguard-agent --reset` is the only way back to local.
4. Pair with `--cloud-account a@x`, then restart with `--cloud-account b@y` —
   one wipe, re-pairs into `b@y`.

## Consequences
- One additive field is the entire wire delta; old and new agents/servers interop.
- The cloud vs. local decision lives entirely in the agent and is opt-in, so the
  self-hosted experience is untouched.
- Server-side pairing still requires the tenant admin to **accept**, so a bad
  `cloud_account` just lands an agent in that tenant's pending queue to be rejected.
