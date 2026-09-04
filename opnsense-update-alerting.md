---
title: OPNSense Update Notifications
description: Using Uptime Kuma to notify you of available updates for your OPNsense router
published: true
date: 2026-09-04T22:17:43.580Z
tags: 
editor: markdown
dateCreated: 2026-09-04T11:19:24.892Z
---

# OPNsense Firmware Alerting

How OPNsense got a working "update available" notification in Uptime Kuma — the API user it runs as, the two monitors that do the work, and three real bugs that showed up once it hit production.

## Introduction
A long-standing source of frustration for me with OPNsense is that it has no built-in way to notify you of an available update.  You can set up a cron job to check for available updates, but you need to log into the GUI to see if any are available.  I've found a variety of scripts to run on the OPNsense box itself that are supposed to provide these notifications, but haven't had any luck.

This guide, prepared with the assistance of [Claude](https://claude.ai), describes using [Uptime Kuma](https://uptimekuma.co/) to notify you of available updates.  It assumes you already have Uptime Kuma installed, and one or more notification methods configured.  Uptime Kuma will then trigger the update check every six hours, and notify you via your chosen method when an update is available.

It's likely this method can be adapted to other monitoring tools.  If they're able to make authenticated HTTP requests on a schedule, and parse a JSON response, they should be able to follow this method.

## Why two monitors, not one

OPNsense doesn't check for firmware updates on a schedule by itself — its cached result only refreshes when something calls `POST /api/core/firmware/check`. Left alone, the web UI's "update available" banner can go stale indefinitely. So the setup splits into two jobs:

| | Monitor | Job |
|---|---|---|
| 1 | **OPNsense - Firmware Check Trigger** | Every 6h, ask OPNsense to refresh its cache |
| 2 | **OPNsense - Firmware Update Available** | Poll that cache and alert when it says something's pending |

They're independent Uptime Kuma monitors with no dependency link between them — Kuma has no "run B after A" feature. Monitor 2 just polls often enough (every 30 minutes, see [Bug 1](#bug-1-the-alert-monitor-was-always-checking-mid-refresh)) that it doesn't matter exactly when Monitor 1 last fired.

## OPNsense: API user and permissions

Firmware-status access needs its own user, scoped to nothing else:

1. **System → Access → Users** — create (or reuse) a user dedicated to this integration, not a personal login.
2. Open the user, scroll to **Effective privileges**, click **+**.
3. Search `Firmware` and select **"System: Firmware"**. This is the *only* privilege OPNsense exposes for this — there's no finer-grained option scoped to just `/status` or just `/check`. Its source definition (`OPNsense/Core/ACL/ACL.xml`, page id `page-system-firmware-manualupdate`) covers:

   ```xml
   <pattern>ui/core/firmware/*</pattern>
   <pattern>api/core/firmware/*</pattern>
   <pattern>ui/diagnostics/log/core/pkg/*</pattern>
   <pattern>api/diagnostics/log/core/pkg/*</pattern>
   ```

   So alongside the two API endpoints used here, it also grants the Firmware GUI page and the package-log diagnostics endpoint — still well short of admin, but worth knowing it isn't scoped to *only* what this integration calls.
4. Save the user, then scroll to **API keys** and click **+** to generate a key/secret pair. OPNsense shows the secret exactly once at creation — copy it immediately, it can't be retrieved again later.
5. Auth is HTTP Basic on every request: the **API key** is the username, the **API secret** is the password.

## Uptime Kuma: the two monitors

Both monitors authenticate the same way as above (HTTP Basic, key/secret) and notify through the existing **Matrix Alert** channel.

### Monitor 1 — Firmware Check Trigger

| Field | Value |
|---|---|
| Type | HTTP(s) |
| Method / URL | `POST https://<ROUTER_FQDN>/api/core/firmware/check` |
| Accepted status codes | `200-299` |
| Interval / retry | 21600s (6h) |
| Max retries | 0 |

A plain success/failure check — 200 means the refresh request was accepted, nothing more.

### Monitor 2 — Firmware Update Available

| Field | Value |
|---|---|
| Type | JSON Query |
| Method / URL | `GET https://<ROUTER_FQDN>/api/core/firmware/status` |
| JSON path (JSONata) | `` status = "none" ? "OK" : status_msg `` |
| Operator / expected value | `==` / `OK` |
| Interval / retry | 1800s (30 min) |
| Max retries | 0 |

The `jsonPath` field takes any JSONata expression, not just a bare key — which is what makes this one work. `status` is OPNsense's reliable three-state enum (`none` / `update` / `upgrade` / `error`); `status_msg` is its human-readable sentence, e.g. *"There is 1 update available, total download size is 6.0MiB."* The ternary uses the enum to decide UP/DOWN and only surfaces the sentence when there's something to say — so a DOWN alert in Matrix reads as an actual description, not `comparing update == none`.

<details>
<summary>Implementation note: building these via the Socket.IO API</summary>

Uptime Kuma has no REST CRUD for monitors — management goes through its Socket.IO API, here via the `uptime-kuma-api` Python client. The installed client version predates Kuma's newer "Monitor Conditions" system, so its `add_monitor()`/`edit_monitor()` wrappers choke on the server's NOT NULL `conditions` column. Workaround: call the private `_build_monitor_data()` builder directly, manually set `data["conditions"] = []` and `data["jsonPathOperator"]`, and pass the result to the low-level `api._call('add' | 'editMonitor', data)`. Also note `notificationIDList` must be a **dict** (`{"1": True}`) on writes, even though reads return it as a **list** (`[1]`).

</details>

## Three bugs found running this in production

### Bug 1: the alert monitor was always checking mid-refresh

**Symptom:** Monitor 2 stayed UP for over a day despite the OPNsense UI showing a pending update since the previous afternoon.

**Cause:** Monitor 2 was originally paired to Monitor 1's exact 6h cycle, firing under a second after it. But `POST /check` doesn't update the cache synchronously — for roughly 10 seconds after it's called, `GET /status` returns a placeholder with no `status` key at all, before settling on the real result. Monitor 2's schedule landed inside that window on *every single cycle*, so it never once saw a completed check.

**Fix:** decoupled Monitor 2 onto its own 30-minute interval instead of chasing Monitor 1's cadence. `GET /status` is a cheap read that doesn't trigger anything, so polling it independently and more often means it reliably lands outside the refresh window — and self-corrects within minutes on the rare cycle it doesn't.

### Bug 2: the alert text was useless

**Symptom:** Matrix notifications read `JSON query does not pass (comparing update == none)` — accurate, meaningless.

**Cause:** Uptime Kuma's JSON Query alert message is hardcoded to `comparing <actual> <op> <expected>`, built from whatever the `jsonPath` extracts. Pointed at the bare `status` field, "actual" was just an enum token.

**Fix (first pass):** pointed `jsonPath` straight at `status_msg` instead — OPNsense's own descriptive sentence. This immediately surfaced Bug 3.

### Bug 3: the descriptive field lied about "safe" states

**Symptom:** After installing the pending update (which reboots the router), Monitor 2 should have gone quiet — instead it looked set to alert again.

**Cause:** OPNsense caches its last check result in `/tmp/pkg_upgrade.json`, which the post-install reboot wipes. Until something re-triggers a check, `GET /status` returns:

```jsonc
{ "status": "none", "status_msg": "Firmware status requires to check for update first to provide more information." }
```

`status` is still the safe `"none"` — but that `status_msg` string doesn't match "no updates" text, and Bug 2's fix compared `status_msg` directly, so this uninitialized-cache state would have read as a false DOWN.

**Fix:** the JSONata ternary in the final config — `status = "none" ? "OK" : status_msg`. The reliable enum decides pass/fail (so "not yet checked" and "confirmed up to date" both correctly read as OK); the descriptive sentence only appears in the alert when there's actually something to report.

## Verified against a real update

This whole chain got exercised end-to-end by an actual firmware release, `26.7.3_8 → 26.7.3_11`:

1. OPNsense's own scheduled check found it and showed it in the web UI.
2. Monitor 2 (post-fix) read `status: "update"` and alerted with *"There is 1 update available, total download size is 6.0MiB."*
3. Update installed, router rebooted, `/tmp/pkg_upgrade.json` cleared as expected.
4. `GET /status` briefly returned the uninitialized-cache state from Bug 3 — Monitor 2 correctly stayed UP through it, no false alarm.
5. A fresh `POST /check` confirmed `product_version: 26.7.3_11` and `status_msg: "There are no updates available on the selected mirror."` — Monitor 2 UP on the real result.

## Known caveats

- `status: "error"` (e.g. the router can't reach its mirror) trips the same DOWN alert as a real update, with the connectivity error as the message text. Deliberate — a broken check is itself worth knowing about — but worth remembering if a Matrix alert ever mentions a mirror problem instead of a version number.
- The two monitors are timed to make collisions unlikely, not to guarantee ordering. If Uptime Kuma's scheduler ever bunches them back to back again, Bug 1 applies for that one cycle — self-correcting within 30 minutes, not a silent multi-day miss like before.