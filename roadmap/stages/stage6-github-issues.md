# Stage 6 — cellpy 2.3 scope plan (issue set)

**Date:** 2026-09-26 · **Status:** 🟠 **draft — proposed for maintainer confirmation** (no
issues cut, no tracking issue yet). Decisions to confirm in §9; once confirmed, cut the
issue set (label `cellpy2-stage6`, milestone `v.2.3`, tracking issue) and add the Stage 6
row to the [architecture dashboard](../cellpy2-architecture-plan.md).

Coordinating doc: [cellpy2-architecture-plan.md](../cellpy2-architecture-plan.md) (§3 "complete
cellpy 2", §5 loader contract, §6 timeline). Day-to-day: [`../../CURRENT.md`](../../CURRENT.md).
Predecessor: [stage5-github-issues.md](stage5-github-issues.md) (2.2, tracking #783).

## 0. Why a Stage 6

Stage 5 was cut lean on purpose: additive features on the **current** schema, with the
contract-touching work pushed out. Three kinds of debt are therefore waiting, and none of
them has a stage:

1. **The deferred structural headline** — SPEED-30 versioned headers (`v.2.3` milestone,
   stage5 §6), plus the two analysis routines parked with it (#73, #889).
2. **Designs captured but unscheduled** — data curation & provenance (#206), written to be
   sequenced *after Epic L*, which shipped 2026-09-26 (#1016, #1100–#1104); and the push half
   of external metadata sources (#784) — its read path is pulled forward into 2.2 (§6).
3. **cellpy 1 left-overs with no owner.** 2.1 removed the *API* shims (stage4 track E). The
   *engine/IO* left-overs were never assigned: `OldCellpyCellCore` and the
   `native_schema=False` bridge, `cellpycore.legacy.mapping` / `to_legacy`, HDF5 v4–v8
   `legacy_read`, and — the biggest — the `BaseLoader` class hierarchy that every shipped
   instrument loader still subclasses, although the 2.0 contract is the structural
   `InstrumentLoader` Protocol and only the `maccor_txt_native` pilot conforms to it.
   Architecture plan §3 says "deleted when v1.x support ends"; that gate is **decision
   #438-6**: 12 months after stable v2.0.0 (2026-07-26) → **2027-07-26**.

Stage 6 is the release that takes (1) and (2) and *prepares* (3) so that the sunset date is
a deletion, not a project.

**Reactive stream stays separate**, as in Stage 5: v2 bug reports ship as `v2.2.x` patches
off `master`; Stage 6 scope does not absorb them. Open Stage 5 issues (S/I/R epics) are not
rolled into Stage 6 by default — they finish in 2.2 or are moved *explicitly* (§7).

## 1. Scope (proposed)

| In 2.3 | Prepared in 2.3, deleted at the v1.x sunset (≥ 2027-07-26) | Parked (spot kept) |
|---|---|---|
| **F** SPEED-30 value+unit+dtype **versioned headers** behind the Schema indirection; cellpy-file **v10**; `units_label()` re-pointed — **core-first**, the structural headline | **X** `OldCellpyCellCore` + `native_schema=False` bridge; `cellpycore.legacy.mapping` / `translate.to_legacy`; HDF5 v4–v8 `legacy_read` → `legacy-files` extra / standalone converter | UUID / BattINFO-EMMO ontology mapping (opens with M, decided there) |
| **P** Loader shell retirement: built-in loaders become plain `InstrumentLoader` classes (`can_load` / `load` → `LoaderResult` tuple); `BaseLoader` / `AutoLoader` / `TxtLoader` / `AtomicLoad` and `processors/` deleted — **cellpy-only** | | narwhals evaluation (polars plan decision 2) |
| **A** Analysis: GITT/PITT #73 · Fredrik ICA #889 — on `cellpycore.curves` / `cellpy.ica` | | per-test `raw_units`; fsspec beyond ssh; volumetric mode |
| **C** Data curation & provenance #206, phase 1 (recipe object, `c.clean.*`, `revert_to_original`, in-memory) — persistence rides on F's v10 | | |
| **M** External metadata sources #784 — **read path stays in 2.2** (maintainer priority 2026-09-26: try BatBase as soon as possible); Stage 6 takes **M3** push/journal integration and **M4** file pointers (#1107, skip `filefinder`) | | |
| **H** Housekeeping: #770 migration-test cleanup · `DEPRECATIONS.md` removals due 2.2/2.3 · `_old_docs/` · `.cellpy_prms_*.conf` handling | | |

**Cross-repo (F9):** **F** is core-first (header object + Schema indirection live in
cellpycore; cellpy re-pins). **A** is core-first only for curve math that belongs in
`cellpycore.curves`; the ICA/GITT front ends are cellpy. **P, C, H, X are cellpy-only**
except the final `cellpycore.legacy` deletion in X. **M** spans cellpy (Protocol, resolver hook)
and `cellpy-connectors` (client + adapter).

---

## 2. Epic F — SPEED-30 versioned headers (structural headline)

Origin: cellpy-core `column-headers-review.md` §F, `SuperDuperCols` prototype; parked from
2.1 ([stage4 §1](stage4-github-issues.md)), deferred from 2.2 with reasoning in
[stage5 §6](stage5-github-issues.md). Gap analysis [F1](../cellpy2-plans-gap-analysis.md):
`df.attrs["units"]` and schema stamping "must converge on one mechanism".

Same weight class as the native-headers flip: a persisted format-contract change with its
own read-compat, migration and oracle surface. It gets the stage's focus.

| # | Issue | Repo | Spec / goal | Depends | yolo |
|---|---|---|---|---|---|
| F1 | Header object: value + unit + dtype, versioned | core | One `Schema`-indirected header type per column (`RawCols`/`StepCols`/`SummaryCols`); `dtype_map()` and `raw_units` derive from it; schema version bumped and readable at runtime. Replaces `df.attrs["units"]`. | — | no |
| F2 | cellpy consumes F1 | cellpy | `units_label()` / `with_cellpy_unit()` / `c.schema.*` re-pointed to the header object; `CellSchema` becomes a view over core's. Re-pin core. | F1 | no |
| F3 | cellpy-file **v10** | cellpy | Write v10 (headers carry unit+dtype+version); read v9 with upgrade-on-load; `CELLPY_FILE_VERSION` bump; golden fixtures regenerated by script (F8). | F2 | no |
| F4 | Oracle + delta register | cellpy | Comparator through the header mapping; exceptions documented; Δ-row in architecture plan §7. | F3 | no |
| F5 | Docs + migration note | cellpy | Fundamentals "headers & units" page; migration guide entry; `llms.txt`. | F3 | yes |

Goal: per-column units resolved "for good"; no ad-hoc unit dicts anywhere in cellpy or core.

## 3. Epic P — Loader shell retirement (cellpy-only)

Current state (2026-09-26): every shipped loader is `class DataLoader(BaseLoader | AutoLoader
| TxtLoader)`. Stage 3 converted their *internals* to the two-stage `parse()` +
`declarations()` → `harmonize()` design but kept the 1.x shell: `AtomicLoad` temp-file
copying, `fid` generation, `is_db`, `get_raw_units()` statics, `headers_normal`,
`post_processors`, `loader() -> list[Data]`. `set_instrument` imports `DataLoader` modules
directly; `registry.find_loader` is only used for listing. Only
`maccor_txt_native.MaccorTxtLoader` is a pure Protocol loader. The optional
`SupportsIncrementalLoad` (Epic L) is already Protocol-style and is unaffected.

| # | Issue | Spec / goal | Depends | yolo |
|---|---|---|---|---|
| P1 | Framework boundary for the outer contract | One boundary in `cellreader` / `readers/instruments`: resolve path (OtherPath → local `Path`), sniff via `registry.find_loader` / `can_load`, call `load()`, stamp provenance + `FileID` + `test_id` on each `LoaderResult`, build `Data`. `set_instrument(instrument=…)` routes through it; `model=` maps to `instrument_config`. Legacy `DataLoader.loader()` kept behind an adapter for the tiers not yet ported. | — | no |
| P2 | Port tier 1 loaders | `arbin_res`, `arbin_sql` (+ `_7`, `_csv`, `_h5`, `_xlsx`), `neware_txt`, `maccor_txt` (fold the `maccor_txt_native` pilot in): plain classes with `can_load` / `load`; `parse` + `declarations` unchanged; `load_since` kept. Conformance kit (`instruments/testing`) runs on each in CI. | P1 | no |
| P3 | Port tier 2/3 loaders | `custom`, `local_instrument`, `batmo_bdf`, `biologics_mpr`, `pec_csv`, `neware_nda` / `ext_nda_reader`, `neware_xlsx`. `pec_csv` multi-cell (#827, Stage 5 I5) lands as a multi-element `LoaderResult` tuple — coordinate. | P2 | partly (per loader, pattern-following after P2) |
| P4 | Delete the shell | Remove `AtomicLoad`, `BaseLoader`, `AutoLoader`, `TxtLoader`, `processors/pre_processors.py` + `post_processors.py` (hooks stay), `get_headers_*`, `headers_normal` use inside `instruments/`. `LoaderError` semantics unchanged. | P3 | no |
| P5 | Third-party loader guide | Docs: "write a loader in 30 lines" against the Protocol, entry-point registration, conformance kit usage; agents chapter pointer. | P2 | yes |

Goal: `cellpy/readers/instruments/` contains vendor `parse()` code, declarations, hooks, the
contract, the registry and the conformance kit — nothing inherited from cellpy 1.

## 4. Epic A — Analysis routines

| # | Issue | Repo | Spec / goal | Depends | yolo |
|---|---|---|---|---|---|
| A1 | GITT/PITT #73 | cellpy (+ core curves if needed) | Diffusion-coefficient extraction from titration steps; seeded by `docs/examples/05_GITT.md` OCV workflow; `cellpy.gitt` module + tutorial. | — | no |
| A2 | Fredrik ICA #889 | cellpy (+ core curves) | Empirical delithiation analysis of galvanostatic curves (paper method); as an `ica` transform / recipe on `IcaOptions`; supervised summer-student code integrated with tests. | — | no |

Independent of F; both start any time. `collect.IcaOptions` removal (H2) lands in the same
release so A2 does not extend a deprecated surface.

## 5. Epic C — Data curation & provenance (#206), phase 1

Design: [cellpy2-data-curation-provenance.md](../../active/cellpy2-data-curation-provenance.md)
("sequence after / with the live-incremental design" — L is done). Confirm the 2026-09-08
note: the #989 rebase-on-load is *ingestion*, so "pristine raw" = post-rebase raw.

| # | Issue | Spec / goal | Depends | yolo |
|---|---|---|---|---|
| C1 | Recipe object + `c.clean.*` accessor | Ordered, serializable steps (`remove_spikes`, `interpolate`, `downsample`, …); each returns a new cell; `c.is_original()`, `c.history()`, `c.revert_to_original()`. Interaction with `c.update()` defined: new rows pass through the recipe, or recipe is re-applied — decide in the issue. | — | no |
| C2 | Persist recipe in cellpy-file | Recipe + derived cache in v10; `save(only_processed=True)`. | F3, C1 | no |
| C3 | Batch surface | `b.clean(...)` applying one recipe across cells; report column "curated". | C1 | yes |

## 6. Epic M — External metadata sources (#784): fast path in 2.2, push in 2.3

Design: [cellpy2-metadata-source-integration.md](../../active/cellpy2-metadata-source-integration.md).
**Priority change (2026-09-26):** the maintainer wants to interact with BatBase as soon as
possible, so the **read path stays in `v.2.2`** (Stage 5) and is *not* moved here. What
makes that cheap: the prerequisites already exist —

- BatBase: OAuth2 client-credentials (`/o/token/`, scopes `read`/`write`/`groups`),
  self-service API clients (`/o/applications/`, ife-bat/batbase#390 closed), project-scoped
  row access (#391 closed), DRF endpoints incl. `/api/test-cellpy-tag/`, `/api/test-batch/`,
  `/api/project/`, `/api/cell-*`; reference `scripts/get_bearer_token.py`,
  `scripts/check_api_connection.py`.
- cellpy-connectors: shared base shipped (cellpy/cellpy-connectors#3, #4): `ApiClientBase`,
  keyring/env credential resolution, `cellpy connectors configure <name>`, CLI mount on
  cellpy (jepegit/cellpy#1058/#1059).

| # | Issue | Repo | Where | Spec / goal | Depends | yolo |
|---|---|---|---|---|---|---|
| M0 | `BatBaseClient` | cellpy-connectors#1 | **2.2 — first** | Client-credentials token fetch + in-memory expiry cache, one re-auth on 401, keyring/env credentials, `configure batbase`, `scope="read"` default, `BatBaseAuthError`. Plus a `get(path, **params)` passthrough and `cellpy connectors batbase get <endpoint>` so the API can be explored the same day. | — | **yes** (well specified, on the shared base) |
| M1 | `MetadataSource` protocol + `MetaResolver` hook | cellpy (#784) | 2.2 | `fetch(key) -> MetaRecord | None`; entry-point registry `cellpy.metadata_sources`; JOURNAL/DB layer precedence; provenance names the source; null-object when unreachable; `CellMeta.uuid`. Read-only. | — | no |
| M2 | BatBase `MetadataSource` adapter | cellpy-connectors#2 | 2.2 | Map `/api/test-cellpy-tag/` (+ batch/cell rows) → `MetaRecord`; query key decided here (cellpy tag / cell name); offline ⇒ empty layer. | M0, M1 | no |
| M3 | Push (POST/PUT) + batch journal integration | cellpy + connectors | **2.3** | Explicit opt-in write-back with `write` scope; `batch.load(metadata_source=...)`; journal columns from the source. First payload: the files cellpy actually loaded (→ M4's pointers). | M2 | no |
| M4 | File pointers from the source (skip `filefinder`) | cellpy jepegit/cellpy#1107 + connectors | **2.3** | `MetaRecord.files: tuple[FileRef, ...]`; `cellpy.get(source=, key=)` / `batch.from_source(...)` open the pointed-to `.cellpy` / raw files (`OtherPath`) before falling back to `filefinder`; `size`/`mtime` can short-circuit `update()`. Optional: no `files` ⇒ today's behaviour. Server side: ife-bat/batbase#474 (`TestDataFile` model + `files` on the journal API). | M2, batbase#474 | no |

Status 2026-09-26: M0 shipped (cellpy-connectors#8), M1 merged (jepegit/cellpy#1106),
M2 in review (cellpy-connectors#9). BatBase API gaps filed: ife-bat/batbase#473 (journal
annotations `mass`/`area`/`loading`/`nom_cap`/`cell_type` + name filters), #474 (file
pointers). `CellMeta.uuid` is core-first: cellpy/cellpy-core#151.

Fastest "try it" order: **M0 → explore endpoints from the CLI → M1 ∥ M2**. M0 needs no
cellpy release; M1 is the only cellpy-side change and is additive.

Ontology mapping (BattINFO/EMMO, F10) stays parked; M1 only reserves the vocabulary hook.

## 7. Epic H — Housekeeping

| # | Issue | Spec / goal | yolo |
|---|---|---|---|
| H1 | #770 clean up migration tests | Remove v1→v2 migration-only tests from `master`; keep those that guard v9 read compat. | yes |
| H2 | DEPRECATIONS removals due ≤ 2.3 | `MultiCycleOcvFit.data` / `set_data` (due 2.2), `cellpy.collect.IcaOptions` (due 2.3); regenerate `DEPRECATIONS.md`. | yes |
| H3 | Repo debris | Delete `_old_docs/`; remove `.cellpy_prms_*.conf` legacy-config read path or keep behind a one-line warning; drop `batch_tools/` remnants. | yes |
| H4 | Stage 5 carry-over decision | Any S/I/R issue still open at 2.2 release is either closed as won't-do, or moved to `v.2.3` **explicitly** with a note here. No silent roll. | — |

## 8. Epic X — v1.x sunset (prepared in 2.3, executed at the gate)

Gate: decision #438-6, 12-month `v1.x` bugfix window from stable v2.0.0 → **2027-07-26**.
Stage 6 makes the deletion mechanical; the deletion itself happens in the first release after
the gate (2.4 or 3.0, §9 decision 4).

| # | Issue | Spec / goal | Depends | yolo |
|---|---|---|---|---|
| X1 | Inventory + isolation | Enumerate every `native_schema=False` / `OldCellpyCellCore` / `to_legacy` / `legacy_read` call site (34 `native_schema=False` hits today, incl. `merge`'s "until native merge lands (5b)" note — verify native merge is complete first). Move them behind one module (`cellpy.legacy`) with a single import point; mark with `warn_once`. | — | no |
| X2 | `legacy-files` extra | HDF5 v4–v8 read moves to an optional extra (pandas/pytables dependency isolated) or a standalone `cellpy convert` command producing v9/v10. Default install reads v9+ only. | F3 | no |
| X3 | Sunset deletion (post-gate) | Delete `cellpy.legacy`, `OldCellpyCellCore` and `cellpycore.legacy.mapping` (core PR), the `native_schema` constructor flag; `pandas` becomes an optional dependency if nothing else needs it. Release note + Δ-row. | X1, X2, gate | no |

---

## 9. Sequencing (the DAG) and decisions to confirm

```
 cellpycore-first ──►  F1 header object ─► F2 cellpy re-pin ─► F3 cellpy-file v10 ─► F4 oracle/Δ ─► F5 docs
                                                                    │
 cellpy-side:                                                        ├─► C2 persist recipe
              P1 boundary ─► P2 tier-1 loaders ─► P3 tier-2/3 ─► P4 delete shell   │
                         └─► P5 third-party guide                                  │
              C1 recipe (in-memory) ─────────────────────────────────────────────────┘ ─► C3 batch
              (M0 → M1 ∥ M2 in 2.2)  M3 push/journal ─► M4 file pointers (#1107)
              A1 GITT/PITT · A2 Fredrik ICA          (independent)
              H1 · H2 · H3 (anytime; H2 before A2 lands)
              X1 inventory ─► X2 legacy-files extra (needs F3) ─► [gate 2027-07-26] ─► X3 delete
```

Constraints:
1. **F is the only core-first critical path.** Start F1 first; everything else is parallel.
2. **P must not change loader output** while porting: the #778-style equality oracle
   (harmonized raw of `load()` == today's `loader()`) guards every P2/P3 port.
3. **C2 and X2 wait for F3** (v10). C1 and X1 do not.
4. **P4 before X3** — the shell is what keeps pandas-shaped `Data` alive on the loader side.
5. Startable immediately: **F1, P1, C1, A1, A2, H1–H3, X1** (M0–M2 run in Stage 5 now).

**Decisions to confirm (maintainer):**
1. **Stage 6 = cellpy 2.3** with SPEED-30 as the structural headline (carries stage5 §6
   decision 1 forward) — yes / re-scope.
2. **Loader shell retirement (P) is in 2.3**, not deferred to the sunset: it is cellpy-only,
   unblocks third-party loaders, and is a prerequisite for X3.
3. **#784 read path stays in `v.2.2`** (M0–M2 now, via cellpy-connectors#1/#2 + cellpy #784);
   only M3 (push) is Stage 6. Reflects the 2026-09-26 priority; confirm.
4. **Sunset release**: X3 lands in **2.4** (minor, after the gate) or is the trigger for
   **3.0** (drop pandas, drop `native_schema`). Recommendation: 2.4 if pandas can stay optional
   without an API break; otherwise 3.0.
5. **#206 phase 1 (C1) in 2.3, persistence (C2) with v10** — or C entirely to 2.4.
6. **Yolo policy**: only F5, P5, C3, H1–H3 and per-loader P3 ports are yolo-fit. Everything
   else is `yolo: no`; drive/auto runs on this epic need the non-yolo lane
   ([jepegit/issue-flow#386](https://github.com/jepegit/issue-flow/issues/386)) or run by hand.

## 10. Candidates considered (dispositions)

| Issue / item | Fits | Disposition |
|---|---|---|
| SPEED-30 | F | **In** — headline. |
| #73 GITT/PITT, #889 Fredrik ICA, #770 | A, H | **In** — already `v.2.3`. |
| #206 data curation | C | **In, phase 1**; persistence tied to v10. |
| #784 metadata sources | M | **Read path stays 2.2** (fast path §6); push M3 in 2.3. |
| Loader `BaseLoader` shells | P | **In** — new epic; unscheduled until now. |
| `OldCellpyCellCore`, `legacy_read`, `to_legacy` | X | **Prepare in 2.3, delete post-gate.** |
| narwhals / polars-native public frames | parked | Revisit with X3 (pandas optional) — not 2.3. |
| Volumetric mode, per-test `raw_units`, fsspec beyond ssh | parked | Spot kept; no trigger yet. |
| Open Stage 5 S/I/R issues | H4 | Explicit move or close at 2.2 release; never a silent roll. |
| #340 plotly perf | perf | Separate triage (unchanged from stage5 §7). |

## 11. Next steps

1. Maintainer confirms §9 decisions (edit this file: `Status: confirmed`).
2. Create tracking issue "cellpy 2.3 (Stage 6) — tracking issue", label `cellpy2-stage6`,
   milestone `v.2.3`; cut F1–F5, P1–P5, A1–A2 (existing #73/#889), C1–C3, M3, M4 (#1107)
   (anchor #784), H1–H4 (existing #770), X1–X2 (X3 created at the gate). Record numbers in
   the "Created issue map" header of this file, mirroring stage5.
3. Add the Stage 6 row to the architecture dashboard; move the "Deferred → 2.3" block in
   `CURRENT.md` to point here.
4. Draft the epic plan in cellpy (`/iflow-epic <tracking>`), stages = F | P+A+H | C+M | X.
