# Ascension-native integration design — Extensions.dll reconstruction → wowdecomp

**Date:** 2026-10-15
**Status:** Design approved (in-chat) → awaiting spec review → then implementation plan (superpowers:writing-plans)
**Work branch:** `ascension-native` in `CoinThrow/wowdecomp` (fork; `upstream` = `SatyPardus/wowdecomp`, pull-only)

---

## 0. Terms, references, pins

- **wowdecomp** — C++ decompilation of the WoW 3.3.5a (build 12340) client; x64 is the main build target. Every
  reconstructed function carries a `// OFFSET: 0x...` tag and every global a `// STATIC: 0x...` tag
  (21,424 OFFSET occurrences across 1,317 files, measured 2026-10-15).
- **The reconstruction ("recon")** — `G:\REPO\ascension-extensions-reconstruction` — a function-by-function C++
  reconstruction of Project Ascension's `Extensions.dll` (32-bit; loaded by `Ascension.exe` via a `LoadLibrary`
  cave at `0x4E5CB0`). Coverage (ground truth = the machine-generated reference totals, verified 2026-10-15):
  **1,346 Lua bindings** (167 into the glue state), **533 server packet handlers + 151 client packet senders**
  (opcodes > `0x55E`, routed via detours on `0x631FA0`/`0x631FE0`), **266 events**, **93 CVars** (19 glue /
  74 world), **347 client hooks**, **639 byte-patch diffs** (407 of them `E9` jumps), across **136 modules**.
  (The README's headline 1,277 / 530 / 400 counts a narrower registrar/install-site set; totals are reconciled
  during M0 triage.) `docs/DLL_REFERENCE.md` (709 KB, generated from source) indexes everything to file:line.
  **Read-only behavioral source of truth for this campaign.**
- **Pinned references:** recon `main` @ `b20b3c7` (behavior baseline; includes the *inert* kill switch per the
  2026-09-28 user decision) and recon `full-anticheat` @ `6a764ef` (active kill switch with the captured
  server allow-list — reference data only, **not** ported active). Port base: wowdecomp `main` @ `0e4e9fe`.
- **The address bridge** — wowdecomp's offset tags use the **same addresses** as `Ascension.exe` for the client
  code the recon hooks. Verified on every critical site: `0x8CCE00` → `Grunt.cpp` (`PackLogon` hello
  serializer), `0x8CC7A0` → `Grunt.cpp` (proof/reply handler), `0x9A83E0`/`0x9A88C0` → `SRP6_Client.cpp`,
  `0x466BF0` → `WowConnection.cpp` (header-cipher init), `0x81B5F0` → `FrameScript.cpp` (event dispatch),
  `0x403340` → `Client.cpp` (per-frame tick, kill-switch target). Hook-site → source mapping is therefore
  mechanical via an address index.

## 1. Intent

Make the reconstructed `Extensions.dll` behavior **native** inside the wowdecomp client: the logic lives in the
client's real functions at the same semantic points the DLL detoured/patched, gated by compile-time flags for
the two user-selectable subsystems. **No DLL, no runtime byte-patching, no trampolines, ever.**

**User decisions (2026-10-15):**

1. **Full parity is the end goal** — every recon inventory entry ported; campaign starts with auth + anti-cheat +
   the login flow, then proceeds in feature waves.
2. **Anti-cheat ON = exact genuine behavior minus the kill switch** — kill-switch machinery is present but
   **inert** (matches the recon-main decision of 2026-09-28); DivxTac load attempt is reproduced (no-op on x64 —
   it is a 32-bit module; recorded as such); fingerprint token + tamper reports exact.
3. **Auth and anti-cheat are compile-time gated**; users choose the client flavor at build time. Defaults OFF.
4. **Approach:** in-tree module port with call-site integration + coverage tooling ("Approach 1").
5. **Fork workflow:** `origin` = the user's fork (pushes), `upstream` = the real decomp repo (pull-only).

**Success criteria:** the campaign completion state *is* the coverage ledger — every entry `ported` +
`verified`, or an explicit documented non-port ruling. No behavior claims without ledger evidence.

## 2. Architecture & integration model

### 2.1 Code layout (wowdecomp)

- `WoW/Source/Ascension/` — **flat 1:1 mirror** of recon `src/Ascension/` (~230 files, same filenames,
  e.g. `AscSpellRank.cpp` ↔ `AscSpellRank.cpp`). Each file header links its recon counterpart and the DLL
  addresses it covers. File-level traceability is the contract; the ledger maps paths if either side moves.
- `WoW/Source/Ascension/CMakeLists.txt` — added to the build **only** when `WOWDECOMP_ASCENSION` is ON.
- `WoW/Source/Ascension/INTEGRATION.md` — curated list of every touch point in stock files (skeleton generated
  by tooling, annotations curated).
- `Tools/ascension/` — campaign tooling (Python) + the committed `coverage.tsv` ledger; regenerable heavy
  artifacts under `Tools/ascension/out/` (gitignored).
- `docs/superpowers/specs/` — this document + the implementation plan.

### 2.2 Integration contract ("patch the real functions")

| Recon mechanism | Native form in wowdecomp |
|---|---|
| Detour / replacement on client function (404 install sites) | Code at the same semantic point **inside the real function**; calls the ported module logic. No trampolines. |
| `ReplaceFunction` sites | Function body swapped under the flag, marked; stock body preserved in `#else` where the flag flips behavior. |
| Byte patches (639; recon `.generated.inc` tables) | Source-level edits to the containing wowdecomp function (semantic equivalent); conditional patches (e.g. class-dependent stance tweaks in `AscWorldEntryTweaks`) become calls to ported logic. |
| Lua natives (recon detour on `0x855060`) | Registered from wowdecomp's real Lua-state setup at the same lifecycle point; glue/world state split preserved (167 / rest). |
| Packet handlers (detours `0x631FA0`/`0x631FE0`) | Added to the real message dispatcher; opcodes > `0x55E`; handlers as new module code. |
| CVars (two lifecycle lists: 19 glue / 74 world; flag `0x21` per-character) | Real CVar registration at the same points. |
| Events (recon detour on `0x81B5F0`) | Custom names appended to the real event table at registration. |
| Bridge DBCs (MMgr64.exe via MemoryBridge) + MPQ DBC readers | Same data sources; access reimplemented natively. |
| Load/attach order (`DllMain` → scaffold plumbing → `AscRuntime::InstallHooks` → module `Init` chain → `AscRegistry::OnAttach`) | A single `Ascension::Init()` invoked from the equivalent startup phase; order preserved exactly as documented in recon `INIT-TAKEOVER-WALK.md`. |

**Marker convention** — every touch point carries a machine-readable comment so the tooling can audit claims:

```cpp
// ASCENSION: src/Ascension/AscWorldEntryTweaks.cpp:126 | site 0x403340 | killswitch | inert allow-list check
```

**Guarding convention:**

- feature call sites: `#if WOWDECOMP_ASCENSION` — compiled out entirely when the master flag is OFF.
- auth sites: `#if WOWDECOMP_ASCENSION && WOWDECOMP_ASCENSION_AUTH`
- anti-cheat sites: `#if WOWDECOMP_ASCENSION && WOWDECOMP_ASCENSION_ANTICHEAT`

### 2.3 Campaign tooling (built in M0)

1. **Offset index** — scans wowdecomp for `// OFFSET:` / `// STATIC:` tags →
   `Tools/ascension/offset_index.json` (`{address → file, line, symbol, kind}`). *Nearest-smaller tag* resolves
   mid-function patch sites to their containing function automatically.
2. **Recon inventory** — normalizes every entry from the recon source + tables (attach sites, hook targets,
   natives, opcodes, CVars, events) → `Tools/ascension/inventory.tsv`.
3. **Join + coverage report** — maps each inventory entry to a wowdecomp target with status
   (`unmapped → mapped → ported → verified`) → `Tools/ascension/coverage.tsv` + burn-down counts. A `report`
   mode re-scans the tree for `// ASCENSION:` markers so the ledger cannot drift from the source.

### 2.4 Explicit non-ports (recorded, not translated)

- VM / dense obfuscated layer (~5.6 MB of the DLL) — behavior it *computed* (e.g. the proof-check HMAC) is
  reimplemented natively; the machinery itself has no place in a self-built client.
- Anti-debug / anti-tamper armor (21-entry catalogue, watchdog `0x10A3FCF0`, thread hider `0x100B7F21`).
- The DLL loader + its hook engine + trampoline machinery; packer mechanics; the `ClientExtensionsDummy` export.
- Crypto note: `Ascension/Crypto/` implements X25519, HKDF, ChaCha20, SHA-256/HMAC-SHA256 natively, validated
  against the vectors in `re/ascens_crypto.py` (the verified reference implementation).

### 2.5 Inherited deliberate differences (ported as-is)

The recon documents three deliberate differences from the genuine DLL; we inherit them:

1. **Logon protocol** — the only one that changes form: becomes our `WOWDECOMP_ASCENSION_AUTH` flag
   (flag ON = native Ascension logon; OFF = stock SRP6, recon's default posture).
2. **Character advancement init correction** — the `0x726` known-entries handler sync (recon
   `AscCharacterAdvancement`).
3. **Tooltip insert fix** — `AscTooltipInsertFix` (inserted lines past the tooltip end).

## 3. Flags & build matrix

CMake options, `WOWDECOMP_*` style consistent with `WOWDECOMP_DISABLE_SIGNATURE_CHECKS` — all default **OFF**:

| Option | Effect when ON |
|---|---|
| `WOWDECOMP_ASCENSION` | Master: compiles the Ascension tree; all (non-gated) features active. |
| `WOWDECOMP_ASCENSION_AUTH` | Native Ascension logon replaces the SRP6 path (hello trio at `PackLogon`/`CalculateProof`/`VerifyServerProof`, `kd`/`ks40` session semantics, world-header observables). Implies master. |
| `WOWDECOMP_ASCENSION_ANTICHEAT` | AC per decision 2: fingerprint token + tamper reports exact; DivxTac load attempt (no-op on x64, recorded); kill-switch machinery present but inert. Implies master. |

**Presets** (`CMakePresets.json`): `ascension` (master + auth + AC — primary build), `ascension-srp6`
(master + AC, stock SRP6 — the "recon main" equivalent), `ascension-noac` (master + auth). Default/stock
configure remains upstream-equivalent.

**Build matrix (all must compile):** stock `(F)`; `(T,F,F)`; `(T,T,F)`; `(T,F,T)`; `(T,T,T)`.

**Compile-time only** (user requirement). Runtime configurability can layer on later without redesign — flags
gate *inclusion*; future config would gate behavior details.

**Observable deltas (enumerated; spec-level contract):**

- Auth ON: Ascension logon handshake on the wire (hello `629+pwlen`, challenge, 75 B response, proof
  `[01 00] ‖ HMAC-SHA256(kd,"OK") ‖ 10 B tail`); **plaintext world headers** after `CMSG_AUTH_SESSION` (genuine
  client behavior; server side `CoA.PlaintextWorldHeaders = 1`).
- Auth OFF: stock SRP6; header cipher keyed as stock (server side `CoA.PlaintextWorldHeaders = 0`).

## 4. Porting workflow & verification

**Per-module pipeline** (identical for every module):

1. **Select** — next module from the coverage ledger, in milestone-priority order.
2. **Resolve targets** — the join tool emits the module's mapping table (hook sites → functions via nearest
   `OFFSET`; patch sites → containing function; global reads/writes → `STATIC` symbols; by-address calls →
   real functions).
3. **Translate** — rewrite against wowdecomp types. Fixed conventions:
   - raw reads (`player+0x14`, `DAT_...`) → typed field / symbol access;
   - detour semantics (before / after / override) → inline at the semantic point;
   - behavior **verbatim including bugs**; deviations recorded, never silently fixed;
   - generated recon tables stay generated (port the generators, not the output).
4. **Wire call sites** — `// ASCENSION:`-marked edits in stock files, guarded per §2.2.
5. **Build** — module compiles; full flag matrix stays green.
6. **Verify** — tiered (below), proportional to risk.
7. **Ledger update** — status per entry (`ported` → `verified`) + evidence (build log / test output / wire log).

**Verification tiers:**

| Tier | What | Applies |
|---|---|---|
| V0 | Build matrix: 5 configs compile; stock config remains upstream-equivalent (launch sanity check) | Every change |
| V1 | Tooling audit: `// ASCENSION:` markers ↔ ledger cross-check (no unmarked deltas, no missing entries) | Every module |
| V2 | Offline: crypto vs RFC + `ascens_crypto.py` vectors; fingerprint token vs `token_repro.py` reproduction; packet serialization byte-compared against captures; recon's portable tests (e.g. `native_ca_packets.py`) adapted to run against native modules | Deterministic-I/O modules |
| V3 | Live E2E: native client against the user's `azerothcore-wotlk-coa` fork + loopback fakes; wire logs diffed side-by-side with the recon-DLL client on the same steps | Connectivity-path modules |
| V4 | Milestone acceptance: the milestone's explicit criteria, demoed live | Milestones |

**Module definition of done:** all ledger entries `ported`; V1 clean; builds in all relevant configs; V2/V3
where applicable; evidence recorded in the ledger.

**Unmapped-entry handling:** wowdecomp is ~90% ported. For each inventory entry the join cannot map, M0 records
a ruling: `needs-reconstruction` (the function is reconstructed from the IDA databases as part of that module's
port) or `not-applicable` (behavior achieved through existing code paths / non-port). No entry stays untriaged.

## 5. Phasing & milestones

| Milestone | Contents | Acceptance |
|---|---|---|
| **M0 — Foundations & tooling** | `ascension-native` branch; flags + presets + empty Ascension tree wired into CMake; offset index + inventory + join tools; first full coverage report; **triage of all unmapped entries**; `INTEGRATION.md` skeleton; housekeeping (`.gitignore += build/`, `AGENTS.md` + `GOALS.md` at repo root per the standing-goal pattern) | Coverage report with totals per kind, zero untriaged entries; stock build untouched; all 5 configs compile |
| **M1 — Connectivity** (critical path) | Ascension auth behind the flag: crypto module; native logon trio; session-key + world-header observables. Login/realm/charselect/world-entry protocol essentials from the net modules (opcodes > `0x55E` used in these stages, realm data, keepalive). Any AC pieces the server requires during login (scoped from recon evidence — likely the fingerprint token) | Native client logs into the CoA server fork: auth OK in **both modes**; realm list; char select; world entry; stable keepalive; wire-level side-by-side parity vs recon client for the login flow |
| **M2 — Anti-cheat** | Per decision 2: fingerprint token + tamper reports exact; DivxTac load attempt (no-op x64, recorded); kill-switch machinery present but inert | Flag OFF: no AC code active. Flag ON: genuine-equivalent server-visible behavior, verified against recon observations |
| **M3…Mn — Feature waves** | Everything else: net handlers (530) by subsystem → Lua natives (1,277) by module group → CVars / hooks / UI / data patches → long tail. Wave order from M0 data: connectivity dependency, server correlation, risk, visibility | Each wave demoed working live; ledger burn-down strictly decreasing |
| **Complete** | Every inventory entry `ported`+`verified` or explicit non-port ruling; flag matrix all green | The ledger says full parity |

**Sizing:** M0 small (tooling + wiring). M1 first substantial build-out (critical path; de-risks tooling on real
code). M2 small. Feature waves = the bulk — voluminous but mechanical, which is what the M0 tooling is for.

## 6. Risks & open items

1. **Unmapped inventory entries** — count unknown until the M0 join; the biggest scheduling unknown. Mitigation:
   M0 quantifies before wave order is set.
2. **Mid-function patch semantics** (639 diffs) — each needs a semantic restructure in the containing function;
   rules per case, fallback documented in the ledger.
3. **Missing wowdecomp functions** (~10% unported) — per-entry `needs-reconstruction` ruling; IDA databases
   exist for both binaries.
4. **Crypto correctness** — dual validation (offline vectors + live accept); a wrong primitive fails visibly at
   login.
5. **x64 environment deltas** (alignment, 64-bit time_t, container layouts) — watch during waves; translation
   conventions require typed access, which surfaces mismatches at compile time.
6. **Recon drift** — pinned commits mitigate; the rejoin tooling handles future recon movement.
7. **DivxTac on a 64-bit client** — genuinely cannot load; official-server acceptance of a 64-bit client is
   inherently limited beyond what we reproduce natively. Documented, not silently skipped.
8. **Upstream merge friction** — contained: mass in new directories; stock-file diffs small/guarded/marked.

## 7. Next steps

1. User reviews this spec.
2. `superpowers:writing-plans` → implementation plan for M0 (+ M1 outline).
3. Execute M0; coverage report reviewed; wave order fixed.
