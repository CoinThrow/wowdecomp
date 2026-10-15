# Ascension-native M0 — Foundations & Coverage Tooling — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the Ascension build flags into wowdecomp and build the coverage tooling that joins the Extensions.dll reconstruction's complete behavior inventory against wowdecomp's source, ending with every inventory entry triaged in a committed ledger.

**Architecture:** Three small Python tools (offset index → recon inventory → join/coverage) produce `Tools/ascension/coverage.tsv`; CMake options + presets give the flag matrix; a trivial `Ascension` static library proves the tree compiles into the client. **No behavior is ported in M0.**

**Tech Stack:** CMake 4.x · Visual Studio 2022 (MSVC v143, x64) · Python 3.9 (stdlib only: `unittest`, `argparse`, `json`, `re`) · git

**Spec:** `docs/superpowers/specs/2026-10-15-ascension-native-integration-design.md` — read §2–§5 before starting; the plan argues from it.

## Global Constraints

- All `WOWDECOMP_ASCENSION*` options default **OFF**; sub-flags imply the master and the CMakeCache must show it.
- Compile-time gating only. A stock (all-OFF) build must remain upstream-equivalent: no Ascension source compiled, no Ascension symbols, no behavior change.
- Tooling: Python 3.9 stdlib only (no third-party packages); emit POSIX-style relative paths inside artifacts; tools create their own output directories.
- Every recon inventory entry ends M0 with a `status`, and every `unmapped` entry additionally with a `ruling` — zero untriaged.
- Heavy regenerable artifacts live under `Tools/ascension/out/` (gitignored). Committed: tools, tests, `coverage.tsv`, `coverage-summary.md`.
- Marker syntax is exact and machine-read — never paraphrase: `// ASCENSION: <recon-path>:<line> | ...`
- Commit style: short imperative ("Add ...", "Fix ...").

## Review Focus

1. **Stock pollution** — Ascension code or defines leaking into a flag-OFF build. Owner: Task 2 (build-log + lib-absence assertions).
2. **Flag imply logic** — a sub-flag ON while the master silently stays OFF (guards never compile). Owner: Task 1 (cache assertions in all combos).
3. **Tag parsing** — hex case/padding variants, CRLF, a tag whose symbol isn't on the next line. Owner: Task 3 (fixture tests).
4. **Inventory completeness** — a parser silently under-matching (dropped table rows). Owner: Task 4 (counts must equal the reference totals exactly, verified against the real tree).
5. **Mid-function address resolution** — a patch site resolving to the wrong containing function, or none near image start. Owner: Task 5 (fixture tests incl. boundary cases).

---

### Task 1: Flags, presets, repo hygiene

**Files:**
- Modify: `CMakeLists.txt` (root; after the `WOWDECOMP_DISABLE_SIGNATURE_CHECKS` block)
- Create: `CMakePresets.json`
- Modify: `.gitignore`
- Create: `AGENTS.md`, `GOALS.md`

**Interfaces:**
- Produces: cache variables `WOWDECOMP_ASCENSION`, `WOWDECOMP_ASCENSION_AUTH`, `WOWDECOMP_ASCENSION_ANTICHEAT` (BOOL, default OFF); compile definitions `WOWDECOMP_ASCENSION[_AUTH|_ANTICHEAT]=1` defined only when effective-ON (guards use `#if WOWDECOMP_ASCENSION` — undefined = 0 in `#if`); presets `stock`, `ascension`, `ascension-srp6`, `ascension-noac` with binaryDirs `${sourceDir}/build/<name>`.

- [ ] **Step 1: add options + imply block to root `CMakeLists.txt`:**

```cmake
# Ascension (Extensions.dll reconstruction) integration. See docs/superpowers/specs/.
option(WOWDECOMP_ASCENSION "Build the Ascension (Extensions.dll) integration" OFF)
option(WOWDECOMP_ASCENSION_AUTH "Use the Ascension logon protocol instead of SRP6" OFF)
option(WOWDECOMP_ASCENSION_ANTICHEAT "Build Ascension anti-cheat behaviors" OFF)
# Sub-flags imply the master; FORCE so the cache tells the truth.
if (WOWDECOMP_ASCENSION_AUTH OR WOWDECOMP_ASCENSION_ANTICHEAT)
    set(WOWDECOMP_ASCENSION ON CACHE BOOL "Build the Ascension (Extensions.dll) integration" FORCE)
endif()
if (WOWDECOMP_ASCENSION)
    add_compile_definitions(WOWDECOMP_ASCENSION=1)
    if (WOWDECOMP_ASCENSION_AUTH)
        add_compile_definitions(WOWDECOMP_ASCENSION_AUTH=1)
    endif()
    if (WOWDECOMP_ASCENSION_ANTICHEAT)
        add_compile_definitions(WOWDECOMP_ASCENSION_ANTICHEAT=1)
    endif()
endif()
```

- [ ] **Step 2: verify imply logic in every combo.** Fresh dirs, then grep the cache:
  - `cmake -S . -B build/tmp-flags -G "Visual Studio 17 2022" -A x64` → `Select-String build/tmp-flags/CMakeCache.txt 'WOWDECOMP_ASCENSION:BOOL'` → `=OFF`
  - `cmake -S . -B build/tmp-flags2 -G "Visual Studio 17 2022" -A x64 -DWOWDECOMP_ASCENSION_AUTH=ON` → master `=ON`
  - `... build/tmp-flags3 ... -DWOWDECOMP_ASCENSION_ANTICHEAT=ON` → master `=ON`
  - `... build/tmp-flags4 ... -DWOWDECOMP_ASCENSION=ON` → `AUTH:BOOL=OFF`, `ANTICHEAT:BOOL=OFF`
  - clean up `build/tmp-flags*` afterwards.

- [ ] **Step 3: create `CMakePresets.json`** — version 3; all presets inherit a base with generator `Visual Studio 17 2022` and `"architecture": { "value": "x64", "strategy": "set" }`; cacheVariables per Interfaces (`ascension` = all three ON; `ascension-srp6` = master+AC; `ascension-noac` = master+AUTH). Verify: `cmake --list-presets` lists all four.

- [ ] **Step 4: `.gitignore`** — append under a new comment: `build/` and `Tools/ascension/out/`. Verify: `git status -sb` no longer lists `build/`.

- [ ] **Step 5: create `AGENTS.md`** (how to work here: read the spec first; the two build commands incl. preset usage; tooling commands; flag + marker conventions; ledger discipline: claims need evidence rows) **and `GOALS.md`** (standing goal = reach full parity per spec §5; M0 checklist with the spec's M0 acceptance as checkable boxes). Each ≤ ~60 lines.

- [ ] **Step 6: commit:**
```bash
git add CMakeLists.txt CMakePresets.json .gitignore AGENTS.md GOALS.md
git commit -m "Add Ascension build flags, presets and repo process files"
```

---

### Task 2: Ascension tree skeleton wired into the client

**Files:**
- Create: `WoW/Source/Ascension/CMakeLists.txt`, `WoW/Source/Ascension/Ascension.h`, `WoW/Source/Ascension/Ascension.cpp`, `WoW/Source/Ascension/INTEGRATION.md`
- Modify: `CMakeLists.txt` (root; conditional `add_subdirectory`)
- Modify: `WoW/Source/Win64/WinMain.cpp` (call site)

**Interfaces:**
- Consumes: flags from Task 1.
- Produces: static lib target `Ascension` (`<build>/lib/Ascension.lib` when master ON); `Ascension::Init()` — `void`, no args, called from `WinMain`; `Ascension::kBuildTag` (`"ascension-native"`); `INTEGRATION.md` — touch-point table and marker/guard conventions.

- [ ] **Step 1: create the skeleton files.** `CMakeLists.txt` modeled on an existing module (see `WoW/Source/WowServices/CMakeLists.txt`). `Ascension.h`:
```cpp
#pragma once
namespace Ascension {
    void Init();
    extern const char kBuildTag[];
}
```
`Ascension.cpp`: defines `kBuildTag = "ascension-native"`; `Init()` empty body, header comment citing spec §2.4 (M1 replaces it with the real init chain).

- [ ] **Step 2: root `CMakeLists.txt`** — after the `WoW/Source/...` `add_subdirectory` block:
```cmake
if (WOWDECOMP_ASCENSION)
    add_subdirectory(WoW/Source/Ascension)
endif()
```

- [ ] **Step 3: call site in `WoW/Source/Win64/WinMain.cpp`** (top of `WinMain`; exact marker, no paraphrase):
```cpp
#if WOWDECOMP_ASCENSION
    // ASCENSION: src/Ascension/AscMain.cpp:22 | site 0x4E5CB0 | init | M0 wiring; position refined in M1 (INIT-TAKEOVER-WALK.md)
    Ascension::Init();
#endif
```
Include `Ascension.h` under the same guard (or unconditionally if that follows module conventions better — check how sibling modules include cross-tree headers).

- [ ] **Step 4: stock check.** `cmake --preset stock` then `cmake --build build/stock --config Release --target WowClient --parallel`, tee log to `Tools/ascension/out/log_stock.txt`. Expected: success; `Test-Path build/stock/lib/Ascension.lib` → `False`; log contains no `Ascension.cpp`.

- [ ] **Step 5: ascension check.** `cmake --preset ascension` then build, tee to `out/log_ascension.txt`. Expected: success; `build/ascension/lib/Ascension.lib` exists; log contains `Ascension.cpp`.

- [ ] **Step 6: commit:**
```bash
git add WoW/Source/Ascension CMakeLists.txt WoW/Source/Win64/WinMain.cpp
git commit -m "Add Ascension tree skeleton wired into WowClient"
```

---

### Task 3: Offset index tool

**Files:**
- Create: `Tools/ascension/offset_index.py`
- Create: `Tools/ascension/tests/test_offset_index.py`
- Create: `Tools/ascension/tests/fixtures/sample_tags/a.cpp`, `.../b.h`

**Interfaces:**
- Produces: `parse_tags(text: str, file: str) -> list[dict]`; CLI `python Tools/ascension/offset_index.py --root . --out Tools/ascension/out/offset_index.json`; JSON `{"root": str, "tag_count": int, "tags": [{"kind": "function"|"static", "address": int, "file": str, "line": int, "symbol": str}]}`, sorted by `(address, file, line)`. Tools create output parent dirs (`os.makedirs(..., exist_ok=True)`).
- Parse rules (pinned): tag line matches `^// (OFFSET|STATIC): 0x([0-9A-Fa-f]+)`; for `function`, scan forward to the first line containing `(`, symbol = substring before `(`, last whitespace-separated token (`int32_t SRP6_Client::VerifyServerProof(` → `SRP6_Client::VerifyServerProof`); if no `(` within 5 lines → `symbol = ""`. For `static`, first following non-empty line: symbol = identifier immediately before the first of `[`, ` =`, `(`; fallback `""`.

- [ ] **Step 1: fixtures.** `a.cpp`: an OFFSET tag `0x009A88C0` followed by the real signature shape; a STATIC tag followed by `static const uint8_t s_x[4] = {...};`; `b.h`: tag with no padding `0x8CCE00` plus a CRLF-formatted function. Include one 2-line `template <>`-style static to pin the fallback.

- [ ] **Step 2: failing test** (unittest; every test file starts with `sys.path.insert(0, str(Path(__file__).resolve().parents[1]))`):
```python
def test_function_tag_variants(self):
    tags = parse_tags(fixture("a.cpp"), "a.cpp")
    self.assertEqual(tags[0]["address"], 0x009A88C0)
    self.assertEqual(tags[0]["symbol"], "SRP6_Client::VerifyServerProof")
    self.assertEqual(tags[0]["kind"], "function")

def test_static_and_crlf(self):
    b = parse_tags(fixture("b.h"), "b.h")
    self.assertEqual(b[0]["address"], 0x8CCE00)
    self.assertEqual(parse_tags(fixture("a.cpp"), "a.cpp")[1]["symbol"], "s_x")
```
Run: `python -m unittest discover -s Tools/ascension/tests -v` → FAIL (no module).

- [ ] **Step 3: implement** `offset_index.py` (stdlib only; walk `*.cpp`/`*.h` under `--root`, skip `build/`, print per-kind counts to stdout).

- [ ] **Step 4: unit pass + real run.** Run tests → PASS. Then `python Tools/ascension/offset_index.py --root . --out Tools/ascension/out/offset_index.json` → expect ≥ 21,000 tags (record the exact count in the ledger notes, not asserted).

- [ ] **Step 5: commit:**
```bash
git add Tools/ascension/offset_index.py Tools/ascension/tests
git commit -m "Add wowdecomp offset index tool"
```

---

### Task 4: Recon inventory tool

**Files:**
- Create: `Tools/ascension/recon_inventory.py`
- Create: `Tools/ascension/tests/test_recon_inventory.py`
- Create: `Tools/ascension/tests/fixtures/mini_reference.md`, `.../mini_attach/AscAttachPatches.generated.inc`

**Interfaces:**
- Produces: CLI `python Tools/ascension/recon_inventory.py --recon-root <path> --out Tools/ascension/out/inventory.tsv [--verify-totals]`; TSV columns `kind \t id \t address \t recon_file \t recon_line \t detail`; kinds `native|packet_server|packet_client|event|cvar|hook|patch`; `address` = `0x...` or empty; `--verify-totals` exits non-zero if any kind's count differs from the counts parsed from the reference doc's `**Totals:**` line.
- Sources (pinned): `docs/DLL_REFERENCE.md` tables — `Lua API index` (natives; `@`/`#` names included; `Original` column = DLL address or `—`), `Server packet index`, `Client packet index`, `Event index`, per-module `Client hooks` tables; patches — `src/Ascension/AscAttachPatches.generated.inc`, `AscAttachLate.generated.inc`, `AscAttachInline.generated.inc` (`{0xADDR, N, {bytes}}` rows and `// skipped ...` rows; skipped rows are inventoried with `detail = "skipped: holds DLL address"`).
- `detail` conventions: natives `module=<Name>;state=<glue|world|both>`; packets `opcode=<hex>;name=<NAME>`; patches `len=<N>;bytes=<hex>`.

- [ ] **Step 1: build miniature fixtures** mirroring the real formats (a `**Totals:**` line, one table per kind, one module's Client hooks table, a mini `.inc` with 3 patch rows + 1 skipped row). Unit tests assert exact counts/rows from the mini fixtures.

- [ ] **Step 2: failing test** — e.g.:
```python
def test_mini_counts(self):
    entries = inventory(SELF, "--recon-root", FIXTURES)
    kinds = Counter(e["kind"] for e in entries)
    self.assertEqual(kinds["native"], 3)
    self.assertEqual(kinds["patch"], 4)  # incl. the skipped row
```
Run tests → FAIL.

- [ ] **Step 3: implement** the parsers (stdlib `re` for markdown table rows `^\| ... \|$`, section tracking by `##`/`###` headers; `.inc` line regex `^\{0x([0-9A-Fa-f]+), (\d+), \{(.*)\}\},$` and `^// skipped 0x([0-9A-Fa-f]+) \((\d+) bytes\): (.*)$`).

- [ ] **Step 4: unit pass + real-tree verification.** Run tests → PASS. Before parsing, confirm the clone is at the pinned reference: `git -C G:\REPO\ascension-extensions-reconstruction rev-parse HEAD` → `6a764ef…` (`full-anticheat`; the reference doc is the one regenerated at `b20b3c7` and is untouched by that branch's extra commit). If the doc's `**Totals:**` line differs from 1346/533/151/266/93/347, stop and re-pin — do not silently adapt. Then:
`python Tools/ascension/recon_inventory.py --recon-root G:\REPO\ascension-extensions-reconstruction --out Tools/ascension/out/inventory.tsv --verify-totals`
→ expected: exit 0; counts native=1346, packet_server=533, packet_client=151, event=266, cvar=93, hook=347; patch count = number of `{`-rows across the three `.inc` files. Fix adapters until it passes — this count gate is the completeness proof. Spot-check 3 rows by eye (one native name+address, one hook address, one patch with bytes).

- [ ] **Step 5: commit:**
```bash
git add Tools/ascension/recon_inventory.py Tools/ascension/tests
git commit -m "Add recon inventory tool"
```

---

### Task 5: Join + coverage + marker report tool

**Files:**
- Create: `Tools/ascension/join.py`
- Create: `Tools/ascension/tests/test_join.py`
- Create: `Tools/ascension/tests/fixtures/mini_index.json`, `.../mini_inventory.tsv`, `.../mini_tree/client/grunt.cpp` (with a marker)

**Interfaces:**
- Produces: `resolve(address: int, function_tags: list[dict]) -> dict | None` — exact match first, else nearest-smaller `kind=function` tag, `None` below the lowest; CLI `python Tools/ascension/join.py --index <json> --inventory <tsv> --out Tools/ascension/coverage.tsv [--report-root .]`; coverage columns `kind id recon_ref site_addr resolved_addr wowdecomp_file wowdecomp_line wowdecomp_symbol status ruling notes`; `recon_ref` = `<recon_file>:<recon_line>`; join sets `mapped` for address-resolvable rows (hooks/patches) and `unmapped` for the rest (natives/packets/events/cvars, and unresolvable hooks/patches), `ruling` empty; `--report-root` scans `*.cpp`/`*.h` under the root for markers matching `// ASCENSION: (\S+:\d+) \|`, upgrades matching rows to `ported`, and prints `marker-orphan` warnings for unknown refs.
- Status values: `unmapped | mapped | ported | verified` (`verified` is set by hand later).

- [ ] **Step 1: fixtures + tests**: resolution boundary cases — exact match; site between two functions (picks lower); site below the lowest function → `None`; duplicate address → deterministic first-by-sort; marker upgrade changes `unmapped→ported`; orphan marker warns without touching rows.

- [ ] **Step 2: run tests → FAIL.**

- [ ] **Step 3: implement.** TSV written with `\t`, LF, stable row order `(kind, id)`.

- [ ] **Step 4: run tests → PASS.**

- [ ] **Step 5: commit:**
```bash
git add Tools/ascension/join.py Tools/ascension/tests
git commit -m "Add coverage join and marker report tool"
```

---

### Task 6: First full coverage run + triage

**Files:**
- Create (generated, committed): `Tools/ascension/coverage.tsv`
- Create: `Tools/ascension/coverage-summary.md`
- Modify: `WoW/Source/Ascension/INTEGRATION.md` (link + running numbers)

**Interfaces:**
- Consumes: outputs of Tasks 3–5.
- Produces: committed ledger; every row has `status`; every `unmapped` row has `ruling ∈ {module-port, needs-reconstruction, not-applicable}` + short note. Ruling semantics: `module-port` = ported via source-level module translation in feature waves (all natives/packets/events/cvars, and hooks/patches without a client-address-space target); `needs-reconstruction` = client function not yet in wowdecomp, reconstructed during its module's port; `not-applicable` = documented non-port.

- [ ] **Step 1: run the pipeline on the real trees** (index → inventory → join); capture the per-kind totals and the mapped/unmapped split.
- [ ] **Step 2: triage every `unmapped` row** — assign ruling + note. Verify each `needs-reconstruction` by confirming no wowdecomp function is at/below the address (grep the index); use the reference doc's module column for `module-port` assignments.
- [ ] **Step 3: write `coverage-summary.md`** — totals per kind, mapping rates (especially: % of hooks and patches that resolve to a wowdecomp function), unmapped breakdown, notable findings. The hooks/patches mapping rate gates M1 wave ordering.
- [ ] **Step 4: verify** — `Select-String Tools/ascension/coverage.tsv -Pattern '\tunmapped\t\t'` → no matches (no unmapped row with empty ruling); counts in summary match `coverage.tsv`.
- [ ] **Step 5: commit:**
```bash
git add Tools/ascension/coverage.tsv Tools/ascension/coverage-summary.md WoW/Source/Ascension/INTEGRATION.md
git commit -m "Add first coverage ledger and triage summary"
```

---

### Task 7: M0 acceptance close-out

**Files:**
- Modify: `GOALS.md`, `WoW/Source/Ascension/INTEGRATION.md`

- [ ] **Step 1: full build matrix** — configure+build `WowClient` in all five configs (fresh dirs, tee logs to `Tools/ascension/out/matrix_<name>.log`): stock (all OFF) · `-DWOWDECOMP_ASCENSION=ON` only · ascension-noac · ascension-srp6 · ascension (presets). Expected: all five succeed.
- [ ] **Step 2: stock regression sanity** — stock exe builds; no `Ascension.lib`; log has no `Ascension.cpp` (re-confirm Task 2 assertions).
- [ ] **Step 3: update `GOALS.md`** (tick M0 acceptance boxes) **and `INTEGRATION.md`** (final numbers); commit:
```bash
git add GOALS.md WoW/Source/Ascension/INTEGRATION.md
git commit -m "Close out M0 foundations"
```

---

## M1 outline (next plan — not tasks here)

- `Ascension/Crypto/` — X25519, HKDF, ChaCha20, SHA-256/HMAC-SHA256; validated against `re/ascens_crypto.py` vectors.
- Native logon trio behind `WOWDECOMP_ASCENSION_AUTH` at `Grunt::ClientLink::PackLogon` (`0x8CCE00`), `SRP6_Client::CalculateProof` (`0x9A83E0`), `VerifyServerProof` (`0x9A88C0`); session-key + plaintext world-header observables.
- Net-side slice driven by Task 6's mapped hook list: opcodes > `0x55E` on the login/realm/charselect/world-entry path; keepalive.
- E2E both auth modes against the `azerothcore-wotlk-coa` fork; wire parity vs the recon client.
