# Baseline code review — HA Finder BLISS (`finderblissha`)

**Date:** 2026-09-14  
**Reviewer:** Airbloomer first-ever org review (findings only; no code changes)  
**Repository:** https://github.com/Airbloomer/finderblissha (PUBLIC)  
**Branch reviewed:** `main` @ `5ac96ab60a6e96beff48e21a10e352092fc26bf1`  
**Declared version:** `0.5.0` (`custom_components/finderblissha/manifest.json`)  
**Scope:** Full tree + complete git history (all commits, including deleted files). Focus: public-repo secret exposure, authentication, correctness, tests.  
**Out of scope:** Functional patching, merge of this PR, live cloud login against Finder.

This document is a findings report only. It does not propose a merge of itself into product code beyond recording the review.

---

## Executive summary

This is a Home Assistant custom integration that logs into Finder’s cloud with the user’s Bliss app email/password (OAuth2 resource-owner password grant), keeps a SignalR WebSocket open, and exposes climate / sensor / select entities for BLISS1 and BLISS2 thermostats.

**No Critical findings.** In particular, this public repository and its entire git history do **not** contain live secrets, access tokens, refresh tokens, private keys, cloud credentials, or real user passwords.

The integration is nonetheless **not ready to treat as an org-owned, supportable public product** on this `main`:

1. **This fork is stale.** `Airbloomer/finderblissha` is a public fork of `condatek/finderblissha`. Upstream `main` is **9 commits ahead** (released as `0.5.1`) and already ships fixes for several High bugs that are still on this tree (schedule `TypeError`, BLISS2 HVAC mode case, WebSocket command races, stale-device removal, connection resilience).
2. **Authentication is fragile and noisy.** `is_logged_in` is checked but never implemented, so every coordinator poll and every command performs a full password grant. Failed setups can generate thousands of logins per day. Token refresh is unused. Auth vs. connectivity errors are conflated. After retries, 401 detail is discarded so reauth may never fire.
3. **Correctness bugs affect heating control and entity state.** BLISS2 `OFF` can display as Heat; the schedule sensor crashes when `setPoint` is a dict (open upstream issues #7, #8, #14); BLISS1 “manual” Heat/Cool is a **1-hour timed override** that silently reverts — not documented in the Features list; some accounts never receive a device payload and cannot set up at all (upstream issue #13).
4. **There are no tests and no CI.** Zero unit/integration tests, no `.github/workflows`, no `quality_scale`, no diagnostics platform. A cloud-controlled heating integration with no automated regression net is the largest process gap for a first org review.

**Recommendation:** Do not merge this review PR. Treat `main` as a snapshot of upstream `0.5.0`, fast-forward (or rebase) onto upstream `0.5.1`, then schedule a follow-up hardening PR for auth, protocol sync, tests, and public-repo hygiene. Issues are disabled on this fork; the working tracker is still https://github.com/condatek/finderblissha/issues.

| Severity | Count |
|---|---|
| Critical | 0 |
| High | 7 |
| Medium | 10 |
| Low | 8 |

---

## Methodology

- Read the full integration tree (`custom_components/finderblissha/**`, `README.md`, `hacs.json`, `LICENSE`, `.gitignore`).
- Walked every commit on all local/remote branches (`git log --all`, file add/delete history, `git log -p` secret-pattern scan, `git grep` across all blobs).
- Compared this `main` to upstream `condatek/finderblissha` `main` (`d95fd57`, manifest `0.5.1`).
- Cross-checked open/closed issues and merged PRs on the upstream tracker (#4, #7, #8, #11, #12, #13, #14).
- No live Finder login was performed (public review; no org credentials in this environment).

---

## Secrets, tokens, and credential exposure (public repo)

### Result: clean

| Check | Result |
|---|---|
| Current tree (source, JSON, README, license) | No live secrets |
| Full git history (all commits, including deleted `device.py` and replaced LICENSE files) | No live secrets |
| Private keys / PEM / cloud keys (`AKIA…`, `sk_live_`, `ghp_`, `github_pat_`, `-----BEGIN`) | None |
| Hardcoded user emails/passwords other than placeholders | None |
| `.env`, `secrets.yaml`, GitHub Actions secrets, `.github/` | Not present |
| Access / refresh tokens committed | Not present |

What *is* in the repo, and why it is not treated as a leak:

- **`CLIENT_ID = "com.findernet.Bliss2"`** in `pyfinderbliss/const.py` is a **public mobile OAuth client id** used by the unofficial client to impersonate the official Android app. There is **no `client_secret`**. This is expected for this architecture, not a leaked server credential. Combined with `GRANT_TYPE = "password"` it does mean anyone can stand up a client that accepts Bliss usernames/passwords — that is Finder’s public-app design, not a repo secret.
- **`PyFinderBlissAPI("123", "123")`** in `pyfinderbliss_wrapper.py` (`async_main`) is a dummy `__main__` harness, present since the initial release commit. Not a real account.
- Runtime credentials are the user’s Bliss email/password, stored in the Home Assistant config entry (`entry.data["username"]` / `entry.data["password"]`). That is the normal HA pattern; they are not in git.
- Bearer tokens are held only in memory (`BlissClientAsync._token`) and are not logged. Login **failures** interpolate the raw HTTP body into the exception (`Login failed ({status}): {text}`), which can surface in HA logs if Finder ever returns token material or PII on error.

### Residual public-repo hygiene (not a leak, still fix)

- `.gitignore` only ignores `venv/`, `.venv/`, `__pycache__/`. A public heating-control repo should also ignore `.env`, `secrets.yaml`, `*.pem`, IDE junk, and HA `.storage` dumps so a future contributor cannot accidentally commit credentials.
- Author emails (`olli90.ita@gmail.com`, Apple Hide My Email) are in commit metadata. Normal for git; not a code secret.
- There is no `SECURITY.md` and no secret-scanning CI.

**No rotation is required** based on this history review.

---

## Findings

### High

#### H1 — Org `main` is a stale public fork (0.5.0 vs upstream 0.5.1)

`Airbloomer/finderblissha` is a GitHub fork of `condatek/finderblissha`. Reviewed `main` stops at `5ac96ab` (merge of PR #6, v0.5.0). Upstream `main` has since merged:

| Upstream | What it fixed |
|---|---|
| PR #11 / `ca3e9b1` | Schedule sensor `TypeError` when `setPoint` is a dict |
| PR #12 / `d95fd57` (v0.5.1) | WS reconnect resilience, OFF-temp vs season, `async_remove_config_entry_device`, local auto-mode set-point resolution, serialize concurrent `ws.receive()`, confirm commands before reporting applied, stop partial syncs from dropping devices |

This fork still has `origin/fix/sensor-set-point-dict` (the PR #11 commit) **unmerged**. Manifest / README / `issue_tracker` / HACS install URL all still point at `condatek`. Issues are **disabled** on the Airbloomer repo.

Anyone who adds *this* repository as a HACS custom source gets a known-buggy 0.5.0. Anyone who follows the README adds upstream instead — the org copy is then unused and misleading.

**Ask:** Decide whether this fork is the product surface or a personal working copy. If it is the org product, fast-forward `main` to upstream 0.5.1 (or vendor a snapshot) and turn issues on. If it is not, do not advertise it as the install source.

#### H2 — Schedule sensor crashes when `setPoint` is a dict

**Files:** `sensor.py` (`_format_schedule`), `device_parser.py` (`parse_set_point`)  
**Upstream:** issues #7, #14; fix merged in PR #11, **not on this `main`**

```python
temp = sp.get("setPoint", 0) / 10
```

Finder sometimes returns `{"unit": "C", "value": 210}` instead of `210`. This raises `TypeError` on every coordinator tick, puts `sensor.*_schedule` in an error state, and spam-logs. `parse_set_point()` already handles both shapes. `_normalize_schedule_days` (used for climate preset matching) also compares `setPoint` without normalizing, so dict vs int can make two identical schedules look different (related to issue #8).

#### H3 — BLISS2 mode strings are uppercase; climate compares lowercase

**Files:** `device_parser.py` `determine_bliss2_mode`, `climate.py` `hvac_mode` / `preset_mode`  
**Upstream:** fixed in 0.5.1 (parser now returns `off`/`auto`/`manual`; climate uses `.lower()`)

BLISS1 modes are `"off"` / `"auto"` / `"manual"`. BLISS2 modes are `"OFF"` / `"AUTO"` / `"MANUAL"` / `"UNKNOWN"`.

```python
mode = getattr(dev, "mode", "off")
if mode == "off":
    return HVACMode.OFF
# …else HEAT or COOL from season
```

and

```python
if mode != "auto":
    return None
```

After a poll, a BLISS2 device that is off is therefore shown as **Heat** (season defaults to `WINTER`). Presets never match. After a local `set_mode` the wrapper writes lowercase, so the entity looks correct until the next refresh, then flips. README marks BLISS2 as “Partial”; this is a concrete control-surface bug, not just missing features.

#### H4 — Password grant on every poll (`is_logged_in` is dead)

**Files:** `pyfinderbliss_wrapper.py` `_async_ensure_authenticated`, `client.py` `_login`  
**Upstream:** issue #13 (also still unfixed there)

```python
if hasattr(self._client, "is_logged_in") and self._client.is_logged_in:
    return
await self._client._login()
```

`BlissClientAsync` never defines `is_logged_in`, so the guard is always false. Every `async_get_devices` / setter, including the 60s coordinator interval and the 3× retry loop, performs a full ROPC login against `https://accounts.iot.findernet.com/connect/token`.

A healthy instance: ~1 password grant/minute. A failing config entry (issue #13): on the order of **thousands of password logins per day** from one IP. That is an auth-availability and account-lock risk against a third-party IdP that already rejects clients on User-Agent (issue #4). `SCOPE` requests `offline_access` but `refresh_token` / `expires_in` are ignored.

#### H5 — Device sync can fail permanently (`serverPayload: null`)

**Files:** `client.py` `get_devices`  
**Upstream:** issue #13 (open, help wanted)

`get_devices` sends `clientSyncVersion: 0`, `serverSyncVersion: 0`, `clientOperationKey: "ALL"`, `userId: all-zero`, then waits 15s for a frame whose `serverPayload` is not `None`. On some real accounts the server answers in ~300ms with a live `serverSyncVersion` and `serverPayload: null`, then only keepalives. The client treats that as “not yet” and times out. Wrapper retries 3×, then raises a generic `"Failed to fetch devices after retries"`. Setup never completes; the official iOS app works on the same account.

`connect_ws`’s InitRequest wait calls `self._ws.receive()` **without** `asyncio.wait_for`. The `TimeoutError` handler is dead. A silent server hangs setup indefinitely (`ConfigEntryNotReady` never fires).

#### H6 — BLISS1 “manual” HVAC is a 1-hour timed override (undocumented)

**Files:** `pyfinderbliss_wrapper.py` `set_mode` / `set_setpoint` / `_set_manual_timer`; `climate.py` `async_set_hvac_mode`

Setting Heat or Cool on BLISS1 does **not** latch manual mode. It leaves `settings.mode = "AUTO"`, sets `manualSchedule.isOn = true`, and writes `start`/`stop` timestamps with **`duration_hours=1`**. After one hour the thermostat returns to the weekly schedule. Changelog mentions “timed override”; the Features / climate section does not. Home Assistant users (and automations / HomeKit) expect `HVACMode.HEAT` to persist.

This is a comfort/safety surprise: overnight heat or a “hold 21°C” can silently expire.

#### H7 — No tests, no CI, no quality gate

There is no `tests/` tree, no pytest config, no GitHub Actions, no `quality_scale.yaml`, no diagnostics platform, no `translations/` directory (only `strings.json`).

`hacs.json` has no `homeassistant` minimum version. `manifest.json` `requirements` is empty (acceptable if only HA-bundled deps are used) and `codeowners` is `@condatek`.

For an org-owned public integration that writes thermostat setpoints over an unofficial cloud API, the absence of even parser/mode-mapping unit tests is a High process finding. The known bugs above (dict `setPoint`, BLISS2 case, auth guard) are exactly the class of defect a 20-line test would have caught.

---

### Medium

#### M1 — Auth failure detection is string-matching and is discarded after retries

`__init__.py` raises `ConfigEntryAuthFailed` only if the exception text contains `"invalid"`, `"unauthorized"`, or `"401"`. `async_get_devices` catches the real error, `print()`s it, then raises `"Failed to fetch devices after retries"` — which matches **none** of those tokens. Coordinator updates therefore do not start reauth after a persistent 401. Only a login failure that escapes *before* the retry loop will.

#### M2 — Credential validation reports network failures as bad passwords

`async_validate_credentials` swallows **all** exceptions and returns `False`. `validate_input` then raises `InvalidAuth`. Users see “Invalid username or password” when Finder is down, DNS fails, or the User-Agent is rejected again.

#### M3 — Config-flow password field is a plain string; unique_id is not updated on reconfigure

`STEP_USER_DATA_SCHEMA` uses `vol.Required(CONF_PASSWORD): str` rather than HA’s password `TextSelector`. Reconfigure / reauth can change the username without `async_set_unique_id` / `async_update_reload_and_abort(..., unique_id=...)`. The entry unique_id stays `original_username.lower()`, so account identity and title can drift.

#### M4 — Entities are a snapshot of the first refresh

Climate / sensor / select platforms iterate `coordinator.data` once in `async_setup_entry`. Devices added in the Bliss app later never appear until a full reload. `is_deleted` is parsed and sent back on writes but **not** used to hide entities. Upstream 0.5.1 added `async_remove_config_entry_device`; this tree cannot remove a stale thermostat from the UI.

If a device is missing from a later payload, `_find_device()` returns `None` and climate `hvac_mode` becomes `OFF` — which can look like the heat was turned off.

#### M5 — Login / WS sessions have no explicit timeouts; errors go to `print`

`aiohttp.ClientSession` is created without a `ClientTimeout`. Retry failures use `print(f"[FinderBliss] Device fetch failed ...")` instead of `_LOGGER` (changelog for 0.5.0 claimed print was removed). On HAOS those lines miss the integration’s filtered log / diagnostics.

#### M6 — Weak public-repo `.gitignore`

See secrets section. Medium because the next accidental `secrets.yaml` or captured `access_token` dump would be public immediately.

#### M7 — Set-point sensors missing `device_class`; last-sync TZ errors uncaught

`FinderBlissSetPointSensor` and `FinderBlissManualSetPointSensor` have a °C unit and `state_class` but no `SensorDeviceClass.TEMPERATURE`. `FinderBlissLastUpdateSensor` catches `ValueError` / `KeyError` but not `zoneinfo.ZoneInfoNotFoundError`. `_set_manual_timer` can raise the same if `device.tz` is unexpected.

#### M8 — HACS / docs / tracker still describe the upstream repo; this fork has issues disabled

Install steps tell users to add `https://github.com/condatek/finderblissha`. That is honest about where issues live, but it means the Airbloomer public repo is not an operable support surface. Combined with H1, org ownership is unclear.

#### M9 — Optimistic climate state without server ACK

`climate.py` writes HA state immediately after `send_operation`. On this tree, a missing ACK is only a warning (`No ACK received from server after command`). The UI can show Heat / 21°C while the thermostat never applied the change. Upstream 0.5.1 started confirming commands; this `main` did not.

#### M10 — `updateStep` profile values vs “minutes”

`select.py` maps Energy saving/Normal/Fast/Super fast → `1/2/3/4`. `set_update_step` is documented as taking minutes and stores `str(minutes)`. README documents 10 / 5 / 2 / 1 minute intervals. If the cloud expects minutes, the select writes the wrong values; if it expects enum levels, the API wrapper is misnamed. No test or captured payload on this tree settles it.

---

### Low

#### L1 — Public official-app client id and spoofed Android User-Agent / device fingerprint

Necessary for the unofficial API (issue #4). Not a secret. Worth documenting as a ToS / breakage risk if Finder tightens client attestation.

#### L2 — OAuth password grant

ROPC is deprecated (OAuth BCP). Acceptable only because Finder’s public app still uses it. Long-term this should move to whatever official redirect flow exists, if any.

#### L3 — `hass.data[DOMAIN]` instead of `ConfigEntry.runtime_data`

Legacy HA pattern. Fine for 0.5.x; migrate when touching setup.

#### L4 — `iot_class: cloud_polling` with a persistent WebSocket

Accurate today (the WS is used as a request/response pipe, not a push listener). A future `cloud_push` design would cut polling and H4 login volume.

#### L5 — `strings.json` has no `last_sync` entity name; no `translations/`

English still works. Last-sync sensor name is hardcoded on the entity.

#### L6 — GPL-3.0 license

Valid. Many HA custom components use Apache-2.0/MIT; GPL-3 is a distribution constraint for some vendors. Not a defect.

#### L7 — Dummy `__main__` harness with `"123"/"123"`

Harmless. Easy to mistake for leftover credentials in a secret scan; prefer a comment that they are placeholders, or drop `__main__` from the custom component package.

#### L8 — No `SECURITY.md`, no code scanning, `codeowners: [@condatek]`

Expected for a first org snapshot; should be updated if this fork becomes the product.

---

## Authentication review (summary)

| Topic | Assessment |
|---|---|
| Credential collection | Config flow + reauth + reconfigure exist and are the right HA shape |
| Storage | HA config entry (standard). Not in git |
| Transport | HTTPS token endpoint + WSS SignalR with `Authorization: Bearer` |
| Client secret | None (public client). OK |
| Token lifetime | Not tracked; refresh token unused |
| Session reuse | Broken (`is_logged_in` missing) → password grant every call |
| Reauth trigger | Works on setup login failure; unreliable on later poll failures (M1) |
| Invalid vs unreachable | Conflated (M2) |
| Password UI | Not masked via selector (M3) |
| History leak | None found |

---

## Tests review (summary)

| Asset | Present on `main`? |
|---|---|
| Unit tests (parser, mode map, schedule format, unique_id) | No |
| Config-flow tests (`pytest-homeassistant-custom-component` or HA helpers) | No |
| Climate command tests (lock, season+mode, 1h timer) | No |
| Fixture payloads (BLISS1 / BLISS2 `serverPayload` samples) | No |
| CI (lint, pytest, HACS / hassfest validate) | No |
| `quality_scale.yaml` / diagnostics | No |

Highest-value first tests, when someone does implement them (not in this PR):

1. `determine_bliss1_mode` / `determine_bliss2_mode` contract (`off`/`auto`/`manual` only).
2. `parse_set_point` + `_format_schedule` for int and dict shapes.
3. `_normalize_schedule_days` equality across those shapes.
4. Config flow: 401 → `invalid_auth`, connection error → `cannot_connect` (today both become invalid_auth).
5. `_async_ensure_authenticated` does not call `_login` when a non-expired token exists.

---

## Suggested order of work (not part of this PR)

1. **Decide fork role** (H1). If this is the org product: enable issues, fast-forward to upstream 0.5.1, retarget HACS/docs/codeowners or keep documenting condatek as upstream.
2. **Stop password-grant hammering** (H4): implement real session state (`access_token`, `expires_in`, optional refresh).
3. **Unblock setup for null `serverPayload`** (H5 / issue #13): protocol follow-up using the server’s `serverSyncVersion`; timeout InitRequest.
4. **Land remaining correctness** if not pulled from upstream: H2, H3, H6 documentation or a true HA hold, M1–M3.
5. **Add tests + hassfest/HACS CI + diagnostics + `.gitignore` / `SECURITY.md`**.

---

## Appendix A — Tree reviewed

```
.gitignore
LICENSE
README.md
hacs.json
custom_components/finderblissha/__init__.py
custom_components/finderblissha/climate.py
custom_components/finderblissha/config_flow.py
custom_components/finderblissha/const.py
custom_components/finderblissha/manifest.json
custom_components/finderblissha/select.py
custom_components/finderblissha/sensor.py
custom_components/finderblissha/strings.json
custom_components/finderblissha/pyfinderbliss/__init__.py
custom_components/finderblissha/pyfinderbliss/client.py
custom_components/finderblissha/pyfinderbliss/const.py
custom_components/finderblissha/pyfinderbliss/device_parser.py
custom_components/finderblissha/pyfinderbliss/pyfinderbliss_wrapper.py
```

No other product files exist on `main`.

## Appendix B — History reviewed

`a4dba11` (initial README/LICENSE) … `5ac96ab` (PR #6 / v0.5.0). Also inspected unmerged `origin/fix/sensor-set-point-dict` (`ca3e9b1`) and deleted `pyfinderbliss/device.py`. No credential-bearing blobs in any revision.
