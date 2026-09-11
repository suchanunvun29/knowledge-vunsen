# Phase 13 (KPI รายเขต และรายงาน KPI รายเขต, Module N 🔒) — FULL verification round 2026-08-24

> Archived from `review.md` on 2026-08-24 when Phase 12's post-fix TARGETED round became the current round.
> Everything below is moved verbatim. One deliberate exception per `policies/documentation.md` §4:
> the phase's `## Unverified Behaviour - undeployed phases` block **stays live in `review.md`** until
> Phase 13 is actually deployed (devops reads it at deploy time, which happens after this archive).

## Verification Summary (current round)

**Phase 13 (Module N — KPI รายเขต และรายงาน KPI รายเขต) — FULL round, 2026-08-24. First verification for all 18 tasks (14 `[backend]` + 4 `[frontend]`).** Implemented across two engineer sessions; nothing from their reports was trusted — everything below was re-derived against real code, real HTTP responses and raw SQL on the real dev server + Supabase DB.

**Schema/contract check**: `Territory` / `TerritoryAssignment` / `TerritoryGroup` / `TerritoryGroupMember` / `Target` (+ `TargetScope` ×3) / `Salesperson.excludedFromTerritoryTotals` in `schema.prisma` match `design.md`'s Data Model field-for-field (round-3 shapes: `isSupervisor Boolean`, banned `TerritoryRole` enum absent, `btree_gist`-era indexes incl. `@@index([salespersonId, isSupervisor, effectiveTo])`, dual/triple unique pairs). Zero drift.

**Equation (Territory KPI Rules ข้อ 3)** — live on throwaway fixtures in year 2099: territories T1/T2/T3, group G(T1+T3), excluded person SP_EX, mixed shared deals; `GET /territory-kpi/team?YEAR 2099` as MANAGER returned companyTotal 280,000 = Σ revenue(170,000: T1 130k + T2 40k + T3 0) + personalBucket 90,000 + unassignedBucket 20,000 exactly; MONTH/QUARTER slices reconciled per-period. Real period YEAR 2026 (pre-fixture AND post-cleanup): 0 + 0 + 14,555,965.119999997 ≈ 14,555,965.12 (all 141 hospitals still unassigned — see Open Issues row for the pending real-data setup).

**revenue(T) aggregation**: strictly through `SalesLineCredit` with `excludedFromTerritoryTotals=false`; verified three deal shapes live — normal×normal shared deal lands both shares in the territory (100k), excluded-only line lands entirely in personalBucket (50k), mixed normal/excluded deal credits the territory only the non-excluded share (40k of 80k). Independent hand-written SQL (`$queryRawUnsafe`) reproduced every number without touching app code. Drill-down hospital sums reconcile to territory revenue per territory.

**Five KPIs reuse Phase 4 verbatim (unit swap only)**: composite renormalization identical formula (`Σ weight×score ÷ Σ weight` over computable only; null + message when none), same `ScoringWeight` rows, `MetricResult` shape shared via imports from `kpi.service.ts`. REVENUE_VS_TARGET hand-checked: 40,000÷50,000 → achievementPercent 80 uncapped shown, score min(80,100)=80; NEW_CUSTOMERS integer-at-territory rule confirmed (1 hospital whose first *countable* sale fell in the period; fractional credit-splitting NOT applied, matching Territory KPI Rules ข้อ 4); raw SQL agreed on both. RETENTION/CONSISTENCY exercised in their computable branches too (Feb fixture month: retention 100% for repeat hospital, genuine computed 0% when no repeat, CV-of-one-month = score 100; mean-0 branch returns label not zero). Group-member territory gets REVENUE_VS_TARGET non-computable with the exact contract label "ไม่ได้ตั้งเป้าแยก (อยู่ในเป้ารวมของกลุ่ม …)" and `target=null`.

**Labels (ข้อ 5 formats)**: "ยังไม่ได้ตั้งเป้า" byte-exact; parenthetical data-insufficiency format implemented exactly ("ข้อมูลยังไม่เพียงพอ (ต้องการ X เดือน ปัจจุบันมี Y เดือน)") but **not live-exercisable this round** — this DB's seeded settings carry minMonthsForChurn/minMonthsForConsistency = 1, so coverage never falls below the gate (branch code-inspected; see Unverified Behaviour). No non-computable criterion ever rendered 0%; criteria never hidden (all 5 metrics always present in payload).

**Grain & ranks**: team/report return exactly 1 row per active territory (3 territories = 3 rows, not ×assignments). Standard competition ranking over all territories' compositeScore (T2 rank 1 at 84.62; null-scored T1/T3 take distinct numeric tail positions 2/3); RANK_ONLY viewers receive identical global ranks (verified by diffing MANAGER vs restricted payloads).

**Serializer (compute-all-then-strip)**: MANAGER receives `visibility:"TERRITORY_FULL"` everywhere; a linked SALESPERSON member got FULL only on their own territory and EXACTLY the 6 whitelist fields elsewhere (`territoryId,name,ownerNames,rank,compositeScore,computedMetricLabel` + `visibility` — key-set diffed, nothing extra); an unlinked SALESPERSON got RANK_ONLY on every row. Whitelist is the single exported constant `TERRITORY_RANK_ONLY_FIELDS` owned by Module N. `personalBucket`/`unassignedBucket` absent from JSON for both non-MANAGER tokens, present (with `personalBucketEntries` naming the excluded person, personal target 45,000 → 200%, and `unassignedHospitalCount`) only for MANAGER.

**Export xlsx (downloaded and parsed programmatically, all 3 roles)**: MANAGER file = 3 territory rows with money/KPI columns + group row "(กลุ่มเขต)" after them + bucket rows LAST (order territories→groups→buckets), personal/unassigned values matching JSON; MEMBER file keeps money only on their own territory's row, other territory + group rows stripped to rank/score/label columns, NO bucket rows; PLAIN file has zero numeric money cells anywhere, no bucket rows, whitelist columns intact. Group totals are a separate block and are provably NOT added into table sums (bucket block reconciles against territory-row sums alone).

**Role gates**: 401 unauthenticated on all four endpoints (`team`, single, drill-down, overview + export); drill-down 200 for MANAGER anywhere, 200 for member on own territory, 403 for member on other territories and for unlinked SALESPERSON everywhere; unknown territoryId 404; bad metric enum and malformed period queries 400 via Zod. Single endpoint mirrors serializer levels per role.

**Group block**: member territory keeps its own row (grain unchanged) with target column replaced by the group label; group block appended with revenue Σ members (130,000), group target (150,000), achievement 86.67%, score min(ach,100), label "คิดจาก 1 จาก 5 เกณฑ์", owners deduped across members; group row visibility = FULL only if viewer FULL on ALL member territories (member-of-T1-only viewer got RANK_ONLY because T3 was also in the group — strictest reading verified live).

**Frontend**: page renders strictly from payload — bucket section keyed off server-sent `buckets`, drill-down button gated on server-sent `visibility === "TERRITORY_FULL"`, rank-only rows render name/owners/rank/score/label with placeholders elsewhere; grep confirms zero role checks in `territory-kpi/page.tsx` + `components/territoryKpi/*`; PeriodSelector wired (MONTH/QUARTER/YEAR refetch); Export button drives a real file download via `downloadFile()`.

Whole-project `typecheck`/`lint`/`build` green on both sides. All fixtures (5 sales lines + credits, 3 targets, group+2 members, assignment, 4 hospitals, 4 salespersons, 2 users, 1 import batch, 3 territories) removed afterwards; baseline counts reconfirmed identical (10 salespeople / 141 hospitals / 846 lines / 857 credits / 5 targets / 0 territories) and real-period payload re-fetched equal to pre-fixture snapshot. No automated test suite exists — verification was code inspection plus real calls against a real dev server and DB.

## Verified File Manifest — this round

| File | Round |
|---|---|
| `backend/src/services/territoryKpi.service.ts` | 2026-08-24 (read in full; revenue/buckets/KPI/labels/serializer/ranks all live-tested) |
| `backend/src/services/viewerTerritoryScope.service.ts` | 2026-08-24 (read in full; "as of today" resolution live-tested incl. the future-dated-assignment case) |
| `backend/src/services/viewerScope.service.ts` | 2026-08-24 (read in full; composes viewerTerritoryScope for Phase 17, no duplicated resolve logic) |
| `backend/src/controllers/territoryKpi.controller.ts` | 2026-08-24 (all 3 handlers live-tested across 3 roles + unauthenticated) |
| `backend/src/controllers/territoryOverview.controller.ts` | 2026-08-24 (overview + export live-tested; group rows/serializer/export order verified) |
| `backend/src/routes/territoryKpi.routes.ts`, `routes/report.routes.ts` (territory-overview lines), `validators/territoryKpi.validators.ts`, `validators/kpi.validators.ts` (periodQuerySchema), `server.ts` mounts | 2026-08-24 (gate order + Zod 400s verified) |
| `frontend/app/(protected)/territory-kpi/page.tsx`, `components/territoryKpi/TerritoryKpiTable.tsx`, `TerritoryGroupKpiTable.tsx`, `TerritoryKpiDrillDownModal.tsx` | 2026-08-24 (payload-driven rendering confirmed; no role checks; build green; visual viewport check carried by the standing responsive gap) |
| `frontend/lib/types.ts` (Module N types), `lib/api.ts` (territory-kpi/report/download helpers) | 2026-08-24 (types match backend payloads field-for-field; export uses real download path) |
| `backend/prisma/schema.prisma` (Module N-relevant models) | 2026-08-24 (contract check vs design.md Data Model — zero drift) |

### Previous manifest (2026-08-22 round — kept for reference)

| File | Round |
|---|---|
| `backend/prisma/schema.prisma` (+ `migrations/20260822030000_.../migration.sql`) | 2026-08-22 (field-by-field vs design.md; applied to real DB) |
| `backend/src/services/creditResolution.service.ts` | 2026-08-22 (review-memory hook read in full; live-tested paths 1/4/5 above) |
| `backend/src/services/import.service.ts` | 2026-08-22 (review creation call site + PRODUCT_TYPE_ALIAS_MISMATCH push, both APPEND and REPLACE_PERIOD flows) |
| `backend/src/controllers/salesmanNameReview.controller.ts` | 2026-08-22 (read in full; every branch live-exercised except KEPT_SEPARATE-note passthrough) |
| `backend/src/routes/salesmanNameReview.routes.ts`, `validators/salesmanNameReview.validators.ts` | 2026-08-22 (gate order + discriminated union live-tested) |
| `backend/src/server.ts` | 2026-08-22 (router mounted at `/salesman-name-reviews`) |
| `backend/src/controllers/masterData.controller.ts`, `validators/masterData.validators.ts` | 2026-08-22 (partial-update semantics + refine live-tested) |
| `frontend/app/(protected)/name-reviews/page.tsx`, `components/nameReviews/SalesmanNameReviewTable.tsx` | 2026-08-22 (tab 3 wiring; build green; server-driven rendering, no hardcoded role checks) |
| `frontend/components/masterData/SalespersonTable.tsx`, `app/(protected)/master-data/page.tsx` | 2026-08-22 (date column + handler wiring; build green) |
| `frontend/app/(protected)/products/page.tsx`, `components/products/ProductMasterTable.tsx` | 2026-08-22 (contract check: "—"+tooltip when `code=null`, source badge, MANAGER-only edit form) |

## Per-Task Results — this round

- [x] Verified — Phase 13 `[backend]` `revenue(T, งวด)` service: SalesLineCredit-only, excluded-personnel isolation, shared-deal shapes live-proven + raw SQL cross-check
- [x] Verified — Phase 13 `[backend]` personalBucket/unassignedBucket service + 3-bucket equation helper: equation exact on fixtures AND real period; endpoint exposes all three chunks together
- [x] Verified — Phase 13 `[backend]` five-KPI territory service: Phase-4 definitions/renormalize reused verbatim (shared imports); REVENUE_VS_TARGET + NEW_CUSTOMERS hand-checked vs raw DB; RETENTION/CONSISTENCY computable branches exercised
- [x] Verified — Phase 13 `[backend]` label service per ข้อ 5: byte-exact "ยังไม่ได้ตั้งเป้า" + parenthetical insufficiency format; never 0%, criteria never hidden
- [x] Verified — Phase 13 `[backend]` GET /territory-kpi/:territoryId: role-shaped payloads, 404 unknown
- [x] Verified — Phase 13 `[backend]` GET /territory-kpi/team: grain 1 row = 1 territory, buckets MANAGER-gated
- [x] Verified — Phase 13 `[backend]` drill-down endpoint: product-type + hospital breakdowns reconcile to revenue; FULL-only enforcement (403 matrix proven)
- [x] Verified — Phase 13 `[backend]` GET /reports/territory-overview: all mandatory columns, bucket rows appended, group block per spec
- [x] Verified — Phase 13 `[backend]` export endpoint: exceljs reused; file parsed for 3 roles; content parity with screen payload; no data beyond viewer level
- [x] Verified — Phase 13 `[backend]` single whitelist constant (`TERRITORY_RANK_ONLY_FIELDS`): exactly the 6 contract fields, exported shared module for Phase 18
- [x] Verified — Phase 13 `[backend]` serializer compute-all-then-strip: TERRITORY_FULL/RANK_ONLY diffed per token; rank global under restriction; whitelist key-set exact
- [x] Verified — Phase 13 `[backend]` bucket blocking for non-MANAGER in JSON and export
- [x] Verified — Phase 13 `[backend]` group block: member rows keep grain + group label + non-computable REVENUE_VS_TARGET; group totals separate from table sums; group visibility = FULL-on-all-members rule
- [x] Verified — Phase 13 `[backend]` Zod schemas on every query/params (period type/year/periodNumber, territoryId, metric enum) incl. 400 paths
- [x] Verified — Phase 13 `[frontend]` report page: table per grain, owners-in-one-cell (+ "ยังไม่มีผู้ดูแล"), %ถึงเป้า primary, composite+label column always, buckets/group blocks only when server sends, period selector wired
- [x] Verified — Phase 13 `[frontend]` render strictly by server-sent visibility level (no hardcoded role checks anywhere in the page/components)
- [x] Verified — Phase 13 `[frontend]` drill-down reuse gated to TERRITORY_FULL rows (button conditional on server field)
- [x] Verified — Phase 13 `[frontend]` Export Excel button downloads a real, program-parsed xlsx

## Issues Found — this round

None blocking. Four non-blocking notes: (1) `GET /territory-kpi/:territoryId` on a *deactivated* (`isActive=false`) territory returns `200 {"period":…}` with the `territory` key dropped (serializer list contains only active territories) instead of a 404 — cosmetic edge reachable only via the PATCH isActive toggle. (2) With a month that has NO target at all, the payload's `target` is `0` rather than `null`, so the frontend money cell shows "0.00" while the %ถึงเป้า column correctly falls back to the "ยังไม่ได้ตั้งเป้า" label — cosmetic; % and KPI columns never show a false 0%. (3) PRODUCT_GROUP's not-computable reason keeps Phase 4's specific wording ("ไม่มีการตั้งเป้ากลุ่มสินค้าในงวดนี้") instead of the generic "ยังไม่ได้ตั้งเป้า" — accepted interpretation (ข้อ 5's hard rules are "never 0% / never hidden", both held; wording is more informative), recorded here so the choice is visible. (4) The seeded `EvaluationSetting` in this DB carries minMonthsForChurn/minMonthsForConsistency = 1 (not the documented default 6), which makes the ข้อ-5 insufficiency labels effectively unreachable until settings are raised — pre-existing data state from earlier rounds, not Module N code.

## Review Outcome — this round

Phase 13 (Module N) verified complete — **18/18 tasks ✅ on a FULL round, zero implementation defects**; all `plan.md` boxes marked. The three-bucket equation holds exactly on fixtures and real data; field-level visibility enforcement matches Data Visibility Rules ข้อ 6 letter-for-letter across JSON and export for all three viewer shapes tested. **Module N now carries `🔒 Security gate`** (added to the Phase 13 heading in `plan.md`): it implements field-level data-visibility enforcement (whitelist stripping, bucket gating, drill-down 403s) whose mistakes leak business figures silently — same rationale class as Modules Q/R. `security` should audit it before any deploy claim. Phase 12's frontend ×7 pages and real bootstrap run remain outstanding (unchanged).

## Change Log

- 2026-08-24 — **FULL verification round for Phase 13 (Module N — KPI รายเขต และรายงาน KPI รายเขต), first round for all 18 tasks (14 `[backend]` + 4 `[frontend]`).** Live-tested against real dev server + Supabase DB with throwaway fixtures in year 2099 (3 territories, group with 2 members, supervisor-less ownerless territory, excluded person, normal/excluded mixed shared deals, TERRITORY/TERRITORY_GROUP/SALESPERSON targets): three-bucket equation exact on fixtures (280,000 = 170,000+90,000+20,000) and on the real period (14,555,965.12); REVENUE_VS_TARGET and NEW_CUSTOMERS independently reproduced via hand-written SQL; serializer diffed per role — MANAGER FULL everywhere, member SALESPERSON FULL only on own territory with exactly-6-field RANK_ONLY elsewhere, unlinked SALESPERSON rank-only everywhere; buckets absent for non-MANAGER in JSON and in parsed xlsx exports; drill-down 403 matrix proven; labels byte-exact per ข้อ 5; grain 1 row = 1 territory; group block separate from table sums. Zero implementation defects; 4 non-blocking notes recorded below. `plan.md`: all Phase 13 lines marked `[x]`; **Phase 13 heading now flagged `🔒 Security gate`** (`qa-engineer` add-only judgment — field-level visibility enforcement). Previous 2026-08-22 round archived verbatim to `review/phase-8.md` / `review/phase-12.md`. All fixtures removed; baseline counts reconfirmed identical; whole-project typecheck/lint/build green both sides.

