# MILDFOODS — SYSTEM & KNOWLEDGE BASE (AI-Optimized Edition)

> **Purpose**: Reference document for an AI system providing first-line support on Mildfoods' inventory/production/cost/sales/accounting system. Every fact below was verified by either (a) live SQL query against the production database, or (b) direct inspection of source code (SSIS package XML, VB.NET source, stored procedure definitions). Facts are NOT inferred from conversation alone.
>
> **Status tags used throughout**:
> - `[VERIFIED]` — confirmed by running a query or reading source code directly, with real sample data shown.
> - `[VERIFIED W/ OWNER]` — confirmed by the system's human owner after the AI's own investigation was inconclusive or ambiguous.
> - `[PARTIAL]` — structurally confirmed to exist, but full behavior/edge cases not traced end-to-end.
> - `[OPEN QUESTION]` — known gap; do not assert an answer, ask the user or flag uncertainty.
>
> **Databases**: `MildFoods` (sales/purchasing docs), `inventory` (lowercase; production/stock/cost), `STAGING` (P&L pipeline only). Cross-database joins require `COLLATE THAI_CI_AS` on both sides when comparing nvarchar codes. Server referenced in SSIS as `mfit.dyndns.biz`.

---

## CHANGE LOG

This is now the single source of truth for this knowledge base (the `.docx` twin was discontinued 2026-07-04 per user instruction — markdown-only from this point forward).

| Date | Version | What changed |
|---|---|---|
| 2026-07-21 | v5.13 | **Second sweep pass (same day as v5.12): re-ran `GetIncompletedProduction` and the negative-balance sweep — confirmed sweeps are continuous, not one-time. New §23.** 3 new `GetIncompletedProduction` mismatches found and fixed, including 2 new sub-patterns: Sub-pattern D (`PR081025-0001`) — a 9-month-closed production's STOCKTRX line was cancelled very recently with no redo and no downstream reuse, traced to an accidental recent cancel-click (cancel-sync bug §17.10 caught live); fixed by **un-cancelling the original row** (owner-directed — restore, don't fabricate ADJUST STOCK) which then exposed a pre-existing double-draw negative balance requiring standard §17.4 remediation on top. Second new pattern: `st_cnt > pi_cnt` (reversed from all prior cases) on `PR150726-0053`, traced to a **prior Claude session's own mistake** (inserted a compensating OUT against the wrong lot, identifiable via `InvoiceNo='CLAUDE AI'`/`ID1='Manual'`) — fixed via standard §17.15 cancel. 2 new negative-balance sweep items: a MOVE-category duplicate traced to a genuine warehouse double-scan (`ID1='Scanner'`, 8 minutes apart, same rootlotno moved twice in one batch — confirmed via `StockDate` real-insert-time, not `StockTrxDate`) — root-caused to `Create_MOVEINOUT` having zero applock/idempotency/post-insert protection (unrelated to the 07-20 `GetStockAllQR` fix, which patched a different bug entirely); permanent 3-layer fix (`sp_getapplock` + idempotency pre-check + post-insert negative-balance guard, mirroring `UpdateApproveProduction`'s proven pattern) delivered as `Create_MOVEINOUT_guard_v2.sql`, not yet deployed, with matching rollback script. Second new item: a genuinely new **SELL late-invoice** pattern (invoice dated 5 days before its actual AUTOSELL fulfillment, `StockTrxDate` stamped to the invoice date rather than the cut date, producing an apparent-but-not-real chronological inversion) — distinct from both §17.16 (wrong-lot AUTOSELL) and §21 (MOVE subquery); fixed via standard ADJUST STOCK compensation, follow-up flagged (AUTOSELL should stamp cut date, not invoice date). Also ran a full cost-field consistency audit (`Cost/smcost/cycletime/CostID/CostDetailID/InventoryCost/DLThbKg/LGCost/LossKG` must match within a rootlotno, per owner directive): found and corrected a recurring anti-pattern from early Claude sessions (counters #00005, #00047) that fabricated placeholder `CostVatDetail` rows (`RMCost=0`) and linked `CostID`/`CostDetailID` to them instead of leaving both `NULL` per the current §20 standard — 2 rows fixed, orphaned placeholder `CostVatDetail` rows left in place (harmless). Confirmed several apparent inconsistencies are NOT errors: `InventoryCost/CostID/CostDetailID=NULL` on MOVE rows and `LGCost=NULL` on SELL rows (both by design, §17.15), and `Cost=0` on productions approved within the last 1–3 days (pending nightly `UpdateRecalCost`/`CalculateNewCost`, self-heals). |
| 2026-07-21 | v5.12 | **Sweep fully complete: all 44 flagged `StockLotBalanceUnit<0` rows resolved (items #29–44 fixed this session, closing out the sweep started in v5.8). New §22: new standing Remark convention for ADJUST STOCK IN rows, retroactively applied to all 44 prior Claude-authored rows, and 5 new sub-patterns discovered during the final 16 items.** New Remark convention (owner-directed): `ADJ{YYMM}CL-{XXXXX}` — `YYMM` from the row's own `StockDate`, `CL` literal, `XXXXX` a 5-digit running count scoped only to Claude-authored rows (`InvoiceNo='CLAUDE AI'`), assigned in StockRunning/insert order; applied to all 44 pre-existing rows in one bulk UPDATE plus every new row from this point forward (now at `#00058`). Five new sub-patterns found and documented (§22.3–22.8): (1) **corruption baked into `produceitem` itself** — a production's ProductID/rootlotno field can be corrupted (stray characters, wrong prefix) at the moment of creation and this propagates identically into both `produceitem` (master) and `stocktrx`; when `produceitem` independently confirms the exact same corrupted value, the compensating ADJUST STOCK IN must match it (not "clean" it), since `RootLotProduct`'s trigger-enforced 1-productid-per-rootlotno lock would reject a mismatched clean value anyway (items #29, #32, #35); (2) **stocktrx-only corruption, NOT matching produceitem** — rarer case where `produceitem`'s own rootlotno field is correct but `stocktrx`'s copy of it is wrong; here the correction goes the other way (fix stocktrx to match produceitem), and — after explicit owner review of the evidence (no such product exists in the `product` master table, appears exactly once ever in the whole database) — the owner directed correcting the same demonstrably-nonexistent ProductID string in `produceitem` too, as a one-off, explicitly justified as "fixing a typo in a field, not altering what physically happened" (LOTNO/KG/Place/Type all untouched) — this remains an exception, not a reversal of the standing "never alter produceitem" rule (item #36); (3) **duplicate-OUT within the same production event** — two STOCKTRX OUT rows sharing the identical timestamp, `CostID`, and `CostDetailID` (proving they're a single duplicated write, not two genuine physical draws) draw down the same one-unit MOVEIN twice; fixed via the standard §17.15 CANCEL mechanism, not ADJUST STOCK IN (item #34); (4) **multi-unit shortage** — a partition can be shorted by more than 1 unit when 3+ genuine transactions (verified individually via `produceitem`) draw against a rootlotno that only ever received 1 unit; fixed with a single ADJUST STOCK IN using `StockPerUnit=+2` (or the exact shortfall count) rather than multiple +1 rows, backdated to before the earliest transaction that first went negative (item #37); (5) **`Remark='CANCEL'` stuck on an IN row from creation, never cleared** — an IN row can be born with the app's provisional `Remark='CANCEL'` template value and never get "confirmed" (cleared) by a subsequent step, which silently excludes it from `UpdateFixRootlotno`'s balance formula (`WHERE ... AND remark<>'cancel'`) forever, even though a later genuine OUT event (a real physical disposal, confirmed by its own descriptive remark) is recorded against it as if it were live stock (item #43); (6) **anachronistic StockRef** — found in 2017-era legacy data: a row can reference a `StockRef` that exists in the table but is dated *chronologically later* than the referencing row itself (impossible causally), and that referenced row is separately cancelled/excluded anyway — functionally identical to a broken/non-existent reference for balance-calculation purposes, just with a real (if backwards-in-time) row behind it rather than a true dangling pointer (item #44). Also newly confirmed in practice (not just structurally, per §20.2): the `BlockRootlotnoMultiProduct`/`trg_stocktrx_rootlot_lock` trigger pair actively REJECTS any INSERT/UPDATE that would introduce a second distinct `productid` for an already-used `rootlotno` (`RAISERROR` + `ROLLBACK TRANSACTION`, confirmed hit live this session) — any manual compensating row on a rootlotno that already carries a corrupted/unusual `ProductID` must reuse that exact string, or the write is silently rolled back with only the trigger's raised error as a symptom. |
| 2026-07-20 | v5.8 | **Session tooling correction + new §17.16: `UpdatePickUpCutlot` root-cause found, fixed, deployed; first-ever system-wide `StockLotBalanceUnit<0` sweep run (44 rows found, 42 remain).** MCP tooling capability corrected: `MSSQL_ENABLE_WRITES=true` was enabled on the connector this session, and `execute_write_query` successfully ran `CREATE OR ALTER PROCEDURE` directly (no SSMS needed) — the long-standing "EXEC/DDL blocked via MCP" note (§14, §17.1, §17.2) was true only while writes were disabled and is now corrected; DDL still can't run through the read-only `execute_read_query` tool (rollback-wrapped), but works fine through `execute_write_query` once the env var is set. First-ever system-wide sweep for `StockLotBalanceUnit<0 AND Active=1 AND StockINOUT<>'cancel' AND Remark<>'cancel'` run across all of `stocktrx` (previously only found incidentally per-production, per §17.5's flagged gap) — 44 rows found, spanning PRODUCTION (16), MOVE (16), SELL (10), ADJUST STOCK (1), SM(OVERHEAD) (1); remediation in progress one row at a time, newest-first triage, 42 remain. Root cause of the SELL-category rows found: `UpdatePickUpCutlot` (the SP behind all `ID1='AUTOSELL'` STOCKTRX rows) computes `StockLotBalanceUnit = StockLotBalanceUnit - cutqty` by reading the **stale snapshot balance stored on whatever row `pickuplotCut.StockRunning` happens to reference** — not the lot's live current balance — and has **zero guard** against cutting from an already-exhausted lot (the SELL-side equivalent of the pre-2026-07-08 `UpdateApproveProduction` gap, §17.1, except never fixed on this side). Fixed and deployed same day in 2 iterations: v1 (all-or-nothing — reject the whole cut if it would overdraw) superseded by v2 per owner directive ("ถ้า lot ไม่พอ ต้องตัดไม่เกิน" — if the lot doesn't have enough, cut no more than what's actually there) — v2 computes live per-partition balance (`rootlotno`,`StockPerKG`,`StockPlace`) via `SUM(StockPerUnit) WHERE StockINOUT<>'cancel' AND Remark<>'cancel'`, then caps each cut via a running-cumulative-demand CASE expression (`CumExclusive >= Available → 0`, `CumInclusive <= Available → full CutQty`, else `Available − CumExclusive` for the row that crosses the threshold) so a batch of multiple cuts against the same partition partial-fills correctly in sequence, not just per-row independently. Full cuts, partial cuts, and zero-cuts are all handled; a `#PickupCutLotShortfall` result set is returned at the end listing any row that didn't get its full requested `CutQty` — **known gap**: `pickuplotCut` has no status/error column, so unless the calling VB.NET app actually reads this trailing result set, a partial/rejected cut currently fails silently from the staff's point of view; app-side code not yet reviewed to confirm. Two real sweep rows fixed and verified as worked examples of 2 different root-cause sub-patterns (full detail in new §17.16): (1) **wrong-lot AUTOSELL pick** (`StockID 050126-0912`/rootlotno `050126-1117`) — fixed by redirecting the sale's attribution to the lot that actually supplied it, not just zeroing the wrong lot's balance; (2) **duplicate MOVEOUT, no physical counterpart** (`StockID 060226-7185`) — fixed via the standard §17.15 CANCEL mechanism, not ADJUST STOCK IN. |
| 2026-07-20 | v5.9 | **New §20: `CostVatHD`/`CostVatDetail` relationship to `ADJUST STOCK` remediation rows — investigated and confirmed clean.** Owner raised a valid concern ("every ADJUST STOCK IN affects CostVatHD/CostVatDetail, right?") after 6 `ADJUST STOCK IN` compensating rows were inserted this session (§17.16 pattern, StockRunning `01146360/373/375/383/439/448`). Investigated and found **no actual impact**: (1) the 2 triggers on `stocktrx` (`trg_stocktrx_rootlot_lock`, `BlockRootlotnoMultiProduct`) only write to `RootLotProduct` — neither touches `CostVatHD`/`CostVatDetail`, confirming there is no trigger-level link from generic STOCKTRX inserts into the cost tables; (2) `CostVatDetail.lot` format is `{productid}-{DDMMYY}` where the date is the **PO/claim receiving date**, not any arbitrary STOCKTRX transaction date — checked all 6 touched `rootlotno`/`productid` combinations directly against `CostVatDetail` and none match any existing row; (3) `CostVatDetail` is populated exclusively by the PO-receiving/claim-adjustment write path (`PODocno` referencing `MF_PO.DocNO` or `ADJ-CLAI-*` claim docs) — a completely separate path from ad-hoc `INSERT INTO stocktrx`. Confirmed the existing cost-field-copy convention (§14/§17.15: backfill `Cost`/`InventoryCost`/etc. on manual inserts from the nearest real row on the same `rootlotno`) is safe — it inherits an already-computed downstream value, it does not re-derive from or write back into `CostVatDetail`. **Open item flagged, not yet investigated**: whether any FG cost recompute process re-averages across all STOCKTRX rows on a rootlotno (including manually-inserted ones) in a way that could shift computed cost indirectly, without ever touching `CostVatHD`/`CostVatDetail` — owner did not confirm this is the specific concern; follow up if raised again. |
| 2026-07-20 | v5.10 | **`UpdatePickUpCutlot` v2 guard shipped broken — live production outage, found via owner's own error output, fixed same session as v3.** The v2 guard (§17.16, deployed 21:54:55) built a `#LiveBalance` temp table but never actually joined it into the `#PickupCutLotAlloc` subquery that references `lb.LiveBalance` — every real call after deploy failed with `Msg 4104: The multi-part identifier "lb.LiveBalance" could not be bound`, meaning **all AUTOSELL cuts were completely blocked** (not silently wrong — a hard error) from 21:54:55 until the owner ran it live and reported the exact error text. Root cause: classic temp-table-built-but-never-joined mistake — writing v2's guard logic in isolation and not re-verifying the full subquery's FROM clause after adding the new CTE-equivalent. **Fix (v3)**: added the missing `LEFT JOIN #LiveBalance lb ON p.rootlotno = lb.rootlotno AND p.StockPerKG = lb.StockPerKG AND p.StockPlace = lb.StockPlace` — no other logic changed. Deployed 23:18:29, confirmed via `OBJECT_DEFINITION` that the join is present in the live version. **Process lesson**: after any `CREATE OR ALTER PROCEDURE` deploy via `execute_write_query`, a real end-to-end test call (not just re-reading the definition back) is needed before considering the deploy verified — `sys.objects.modify_date` and `OBJECT_DEFINITION` only confirm the code was SAVED, not that it RUNS. This session's v2 deploy was reported as "confirmed deployed" on that basis alone, which missed the bug entirely; owner's live test call is what actually caught it. |
| 2026-07-21 | v5.11 | **New §21: root cause of the MOVE-category `StockLotBalanceUnit<0` sweep pattern found and fixed at the VB.NET app layer — `StockCls.GetStockAllQR`.** Traced the recurring sweep pattern (a lot fully consumed by production, then MOVEd again weeks/months later, going negative — hit repeatedly at §17.16 items #15/#16/#19/#20/#21/#22/#23) past the DB layer into the actual app source (`E:\...\STOCK\Project\MF_INVENTORY\MF_INVENTORY`, folder connected this session). Confirmed `dbo.Create_MOVEINOUT` (the SQL Server SP with the 2026-07-08 unapproved-production guard) is **dead code in practice** — its only caller (`StockCls.StockMove`) has the `EXEC CreateMOVEINOUT` line commented out; real MOVE creation is 100% client-side via `StockCls.StockMoveOUT`/`StockMoveIN` (called from `STOCKMOVETMP.vb`'s QR-scan handler), which builds raw `INSERT INTO stocktrx` directly — no SP involved. Found the actual bug in `StockCls.GetStockAllQR` (the function that validates a scanned lot has `stocklotbalanceunit >= 1` before allowing the move): 4 subqueries of the form `select top 1 @rootlotno = rootlotno from stocktrx where stockid = @X` and `select top 1 stockperkg from stocktrx where stockid = @X` have **no `ORDER BY`** — when a StockID's lot has been repacked/re-produced multiple times (common, per the repacking-cascade pattern documented throughout §17.16's worked examples), SQL Server can return an arbitrary historical row's `rootlotno`/`stockperkg` instead of the current one, causing the live-balance check to validate against the wrong weight-partition and pass incorrectly. One block additionally had a stray `·` character corrupting `stockplace` (would throw a SQL syntax error if that branch executed — `MoveDestinationPlace=""` case). **Fix delivered as inline code (not a file) to the owner** for all 4 call sites: collapsed the two independent `top 1` subqueries into one (`select top 1 @rootlotno=rootlotno, @stockperkg=stockperkg ... order by stockrunning desc`) so both values always come from the same current-state row, replaced `stockperkg in (select top 1 ...)` with a direct `stockperkg = @stockperkg` comparison, changed the second query block's trailing `order by stockRunning asc` to `desc` (was picking the oldest matching row instead of the newest), and removed the stray `·`. **Owner applied the fix 2026-07-21** (confirmed via chat, not independently verified in source — no follow-up read-back done this session). This is the client-side counterpart to §17.16's `UpdatePickUpCutlot` fix: `UpdatePickUpCutlot` guards the SELL/AUTOSELL path at the DB layer, this fix guards the MOVE path at the app layer — together they close both known negative-balance-creation vectors found by the sweep. PRODUCTION-category sweep items (#18's Sub-pattern B, `#24` onward) may have a separate root cause not yet investigated. |
| 2026-07-20 | v5.7 | **New §19 (Lot numbering system) — suffix-overflow incident found + fixed + deployed same day**: documented that the `DDMMYY-XXXX` lot suffix is a SHARED numeric pool per date-prefix between `rootlotno` and `stockid` (every MOVE mints a new stockid from the original lot's date prefix, months after production). Prefix `200426` exhausted the 4-digit space (`-9999`) on 2026-07-20 via MOVEs; the old generator would have wrapped to `-0000` and silently minted duplicate StockIDs from the 2nd move onward. Exhaustive scan of all 3 DBs confirmed only 3 modules consume the suffix numerically (`FINDNEWLOT`, `Create_MOVEINOUT`, `UpdateFixMOVE`) — everything else parses only the front-6 DDMMYY or exact-matches. Fix (natural-grow suffix: `...9999, 10000, 10001...`, format-agnostic numeric parsing, NO zero-padding beyond 4 digits) deployed to all 3 SPs 2026-07-20 17:15 + `chk_ChainTest_A.rootlotno` widened varchar(11)→varchar(20); also fixed 2 pre-existing collision bugs in `UpdateFixMOVE`. Verified live same day: 213 contiguous 5-digit stockids (`200426-10000`..`-10210`), zero duplicates; QR-reading SPs confirmed unaffected. Rollback script archived. |
| 2026-07-20 | v5.6 | **Merge**: reconciled a version fork — a parallel session had independently tagged its work "v5.5" (§18, DNS/LAN routing finding above) while this session separately tagged unrelated work "v5.5" too (new §16.5 below, SSMS client SSL/TLS connection troubleshooting). Both are genuine, non-overlapping findings from the same day; merged into one document as v5.6 with no content lost. **New §16.5**: documented an SSMS client-side connection failure — "Cannot connect to mfit.dyndns.biz ... The target principal name is incorrect" (SSL Provider error 0) — hit when connecting via SQL Server Authentication with `Encryption: Mandatory` and "Trust server certificate" unchecked. Root cause: the server's SSL/TLS certificate CN/SAN doesn't match the DynDNS hostname (`mfit.dyndns.biz`) used to dial in, so SSMS's strict TLS validation rejects the handshake after a successful network connection (client-side cert/hostname mismatch, not a server outage or bad credentials). Documented 4 fixes (quick → permanent) and confirmed the MCP `mssql-mcp-node` connector (§16.4) is unaffected, connecting fine to the same endpoint. Worth noting: this is a distinct issue from §18's LAN DNS-redirect finding — §16.5 is a TLS certificate/hostname mismatch (any client, any network path), §18 is LAN clients being routed to the wrong physical host entirely — they can compound (a LAN client hitting the wrong host via §18 would *also* fail cert validation) but neither is the root cause of the other. |
| 2026-07-03/04 | v1.0 – v2.9 | Initial build through P&L pipeline, Account Subsystem, AutoEmail automation, and various corrections (fan-out join bug, MAINCAT taxonomy fix, SA Exp vs SBExp clarification). Full history of this range is in the discontinued `.docx` version. |
| 2026-07-04 | v3.0 | Major research pass resolving 6 long-standing open questions in one session: (1) BOM MAIN/ALTERNATIVE ↔ NeedToBuy consumption order — fully resolved via `CheckBuyQtyR2`/SQL `CreateBOMTrx` source; (2) STOCKTRX cancel flow — fully resolved with a real reversal-row example; (3) `ProjectID='ADJUST STOCK'` — real example found (year-end stock count); (4) `ClaimID` — format and meaning confirmed, revealed 2 new ProjectID values (RETURN, PREMIUM); (5) `StockPlace` — confirmed NO clean master list exists (100+ free-text values); (6) `CalculateNewLoss` — verified with real data. Also added partial coverage of `CostVatHD` VAT/WHT columns, `ProfitlossS1` structure, `Email.vb` function catalog, and resolved the `PI` document-prefix question. |
| 2026-07-04 | v3.1 | Added AP/AR subsystem tables (verified real names: `APMain`/`APDetail`, `ARMain`/`ARDetail`, `AR_MM_Main`/`AR_MM_Detail` — corrected from the `_R`-suffixed names initially given), `MF_Invoice.PaymentStatus` (AR trigger) and `MF_PO.PaymentStatus` (AP trigger), `MF_PODetail.QTYReceived`/`ReceivingStatus` (populated via `UpdatePODetailQtyReceive`/`_Batch` SPs from STOCKTRX), and confirmed MF_Invoice DocNO convention (INV/CN standard prefixes; EX suffix ↔ INVTypeID export, D suffix ↔ INVTypeID domestic — verified by correlation; older 2017-2020 documents have inconsistent legacy prefixes). |
| 2026-07-04 | v3.2 | Corrected v3.1: `APMain_R`/`APDetail_R`/`ARMain_R`/`ARDetail_R`/`ARMMMain_R`/`ARMMDetail_R` DO exist exactly as originally named — they are **table-valued functions** (`SQL_TABLE_VALUED_FUNCTION`), not base tables, which is why they didn't appear in an `INFORMATION_SCHEMA.TABLES` search. Parameterized by `@YearMonth` (e.g. `APMain_R('202606')`) — confirmed this is the same function the `AP_Report.xlsx` VBA tool (see prior memory) queries. |
| 2026-07-04 | v3.3 | Corrected v3.2: opened `APDetail_R`'s actual body and found the exact parse logic (`substring(@YearMonth,1,4)` for year, `substring(@YearMonth,6,2)` for month) — the correct input format is **`YYYY/MM`** (separator required at position 5). The earlier test with `'202606'` (no separator) coincidentally returned the same row count only because SQL Server's date parser tolerates a single-digit month string (`'2026-6-01'`); it is NOT a reliable alternate format and should not be used. |
| 2026-07-04 | v3.4 | Deep dive into AP/AR and NeedToBuy internals: opened `ARDetail_R` (cross-validates the TypeID pricing rule from §7.4 independently), documented `APTrans`/`ArTrans` schema (the raw ledger transaction logs both `_R` functions read from) with their `Trans` type vocabulary (PAY/STOCKIN/NON-STOCKIN/DELIVERY/RECEIVED/CN/ISSUE(D)/PAYMENT.DONE), confirmed real `NON-STOCKIN` PO examples (service/testing suppliers with no physical goods receipt), confirmed `CostVatHD.Approve` has 3 states, and opened `SOIdetail2`/`CreateNowPO` (siblings of `CreateBOMTrx` inside `CheckBuyQtyR2`) — `CreateNowPO` is confirmed as the source table behind `GetRemainingPORMStock()` in the VB.NET NeedToBuy algorithm (§8.4), fully closing that chain. |
| 2026-07-04 | v3.5 | Confirmed `ARTrans`/`APTrans`/`AR_MMTrans` are built by `Create_ARTrans`/`Create_APTrans`/`Create_AR_MMTrans` (full-refresh: DROP+rebuild each run, all depend on a `lastdate` master table and call `Create_DocnoCompany` first; `Create_APTrans` also calls `Create_APDATA` and filters to `>= 2023-01-01`). Found an important parameter-format inconsistency: these `Create_*` SPs parse `@YearMonth` via `LEFT(...,4)`/`RIGHT(...,2)` (separator-agnostic — any single separator character works, documented convention is `'YYYY-MM'` with a dash), which is DIFFERENT from the `_R` reporting functions (`APDetail_R` etc., §11.5) that require a rigid position-5 separator and the `'YYYY/MM'` slash convention. Also documented `ARTrans`'s running-balance-per-docno logic and its Thai-language `Result` column (`ลูกค้าจ่ายเกิน`/`ลูกค้าค้างจ่าย`/`Completed`). |
| 2026-07-04 | v3.6 | Diagram consolidation pass (no new facts, clarity fix): added §2.6 STOCKTRX Quick Reference covering all `ProjectID` values (BUY/production/SELL/MOVE/ADJUST STOCK/RETURN/PREMIUM/cancel) in one place with explicit join keys; merged §8.4's NeedToBuy diagram into one continuous STEP 0→4 flow (previously the `CheckBuyQtyR2` prep stage, BOM explosion, consumption loop, PO-reservation, and final result were described in separate prose blocks); fixed a stale `[OPEN QUESTION]` tag on `ADJUST STOCK` in the §2 master diagram that should have been resolved back in v3.0. |
| 2026-07-04 | v3.7 | **Major discovery**: found an entire previously-undocumented Balance Sheet module — table `BalanceSheet` (monthly ASSET/LIABILITY/EQUITY snapshots by account category), `BalanceSheetIncome` (period-over-period movement), views/TVFs `APIncomeStatement`/`ARBalanceSheet`/`ARMMBalanceSheet`, and orchestration SPs `RefreshBalanceSheet`/`RefreshIncomeStatement`/`GetBalanceSheet`/`GetBalanceSheetIncome`. Confirmed the daily scheduled-task list (via `tracking` table types cleared by the Refresh* SPs) is larger than previously documented: `LastDate`, `SetExchangeRate`, `ImportAcct`, `UpdatePriceFromPODocno`, `CreateInventory`, `CreateAcct`, `CreateBalanceSheet`, `BackupStatement` — several of these (`SetExchangeRate`, `UpdatePriceFromPODocno`, `CreateInventory`, `CreateBalanceSheet`) were not in the AutoEmail task list documented in §12.1. Added new §13. |
| 2026-07-04 | v3.8 | Owner confirmed: Income Statement coverage = `CreateBackUpStatement` (§11.3), Balance Sheet coverage = `GetBalanceSheet`. Opened `GetBalanceSheet`'s full body (~8,000+ chars) — confirmed **EQUITY** is a real third `MAINCAT` (computed as `Assets − Liabilities` per month, resolving the earlier "presumably also exists" uncertainty), documented the full list of source ledgers unioned into `BalanceSheet` (Cash via `ACCT`/`ACCT_MM`, Inventory via `inventorybylastdate`, Trade Receivable/Deferred Revenue via `ARBalanceSheet()`/`ARMMBalanceSheet()`/`Trade_ReceivableMM`, Account Payable via a monthly cursor loop calling `Create_APTrans`+`APMain_R` per month), found a hardcoded Category→sort-order (`no` 1-27+) and Category→MAINCAT mapping table embedded directly in the SP, and noted a typo'd dependency `Create_Trade_Receivabale` (missing 'b'). |
| 2026-07-04 | v3.9 | **Architecture clarification from owner**: the "3 systems" question is resolved — `Profitloss_R2`/`IncomeStatement`/`BalanceSheet` are 3 DELIBERATELY SEPARATE, parallel views of overall company health, each at a different granularity, NOT meant to be unioned into one combined statement: (1) **Profitloss** (§10) = transaction-level P&L, per invoice/product; (2) **IncomeStatement** (§11.3) = period-level P&L, annual/yearly view; (3) **BalanceSheet** (§13) = financial-position view (assets/liabilities/equity), not a profit/loss view at all. This resolves the open question about whether `APIncomeStatement`'s `MAINCAT='09/AP'` numbering implies a union with `Profitloss_R2`'s 01-08 — it does not; the numbering is coincidental/internal to each system, not a shared cross-system sequence. |
| 2026-07-04 | v4.0 | Ran a real company-health check: `BalanceSheet` shows Liabilities exceeding Assets in both 06/2026 and 07/2026 (computed equity roughly -1.0M and -8.4M THB respectively), traced the entire month-over-month liability increase to a single category (`ACCOUNT PAYABLE`, +4.93M, matching the total liability delta almost exactly — Long-Term/Short-Term Borrowing were flat). In the process found `BalanceSheet`'s `ACCOUNT PAYABLE` figure (2.03M for June) disagrees substantially (~3×) with `APMain_R('2026/06')`'s outstanding AP figure (6.99M) for the same month. **Owner directive**: `APMain_R`/`APDetail_R` (§11.5) is the authoritative AP source — `BalanceSheet`'s AP line (populated via its internal monthly cursor loop, §13.1.1) should be treated as potentially stale/secondary when the two disagree. |
| 2026-07-07 | v4.6 | **New topic**: designed KB distribution architecture so external AI clients (not just this Claude Project) can read this document via MCP — evaluated filesystem MCP server, custom Python MCP server with `search_kb` tool, and `OPENROWSET`-in-SQL Server approaches; decided `OPENROWSET` is preferred so the client only needs SQL Server access, no network-share mount required client-side. **Critical security finding**: discovered a THIRD hardcoded plaintext credential in the wild — SQL Server `sa` account with full admin privilege was sitting in a `claude_desktop_config.json` MCP connector config (`mssql-mcp-node`), separate from the `xp_cmdshell`/`ImportACCT` credential (§11.1) and the `ConStr.vb` credential (§12.3). Remediated by creating a least-privilege `mcp_kb_reader` SQL login (SELECT-only + `ADMINISTER BULK OPERATIONS`, DENY on INSERT/UPDATE/DELETE/EXECUTE) across all 3 databases, specifically for MCP/AI client connections — `sa` should no longer be used for this purpose and its password should be rotated. Added new §16. |
| 2026-07-04 | v4.1 | **Root cause of the AP discrepancy found and the v4.0 guidance refined**: it is NOT a data-quality bug, it's a call-sequence artifact. `Create_APTrans`'s docno-inclusion subquery uses `GETDATE()` (today), not `@YearMonth`, to decide which documents to include — only per-row date filtering respects the parameter. `GetBalanceSheet`'s cursor calls `Create_APTrans(@YearMonth)` immediately before `APMain_R(@YearMonth)` for EVERY historical month in sequence, so each month's figure is captured at the moment `APTrans` was freshly rebuilt for it — but `APTrans` is a single shared full-refresh table, so calling `APMain_R('2026/06')` standalone (without first re-running `Create_APTrans('2026/06')`) reads whatever `APTrans` was last rebuilt as (typically the most recent month), giving an inflated/wrong figure for the requested historical month. **Corrected rule**: to get an accurate AP figure for any past month, always call `EXEC Create_APTrans '2026/06'` immediately before querying `APMain_R('2026/06')`/`APDetail_R('2026/06')` — never call the `_R` functions alone for a historical month. Also did a scheduler health check via `tracking`: discovered the `date` column is stored as **text** (`DD-MM-YYYY`), so naive `ORDER BY date DESC` sorts alphabetically, not chronologically (a real trap — `'31-12-2025'` sorts before `'04-07-2026'`). After correcting with `MAX(CONVERT(date,...))`, confirmed most daily tasks (`CreateBalanceSheet`, `ImportAcct`, `CreateAcct`, `LastDate`, `CreateInventory`, `BackupStatement`) ran today (2026-07-04) as expected, but 2 tasks are genuinely stale: `SetExchangeRate` (last ran 2026-01-08, ~6 months) and `UpdatePriceFromPODocno` (last ran 2023-02-13, 3+ years) — worth escalating. |
| 2026-07-04 | v4.2 | **`SetExchangeRate` staleness explained — not a broken job, an intentional replacement.** Owner clarified the task was superseded by a per-invoice actual-rate system: `ExchByINV` (view) computes each invoice's REAL exchange rate from actual bank receipts (ACCT_A-G, `dep IN ('REVENUE','SALE')`, matched to the invoice via `reference_doc`), as `SUM(THB received) / SUM(USD received)` — falling back to the invoice's nominal `ExchangeRate` or the annual average when no completed receipt exists yet. `AvgExchByINV` (view) computes the same idea but aggregated per YEAR, for expenses that aren't tied to any specific invoice. Opened both views and confirmed the SQL matches this description exactly. Added new §11.7. |
| 2026-07-04 | v4.3 | **Corrected §11.6**: `TF%`/`TSX%` DocNO prefixes are NOT legacy one-off artifacts as previously lumped in with `QUO`/`ABC`/`PI19-001` etc. — verified both are a legitimate, currently-active invoice numbering pattern, consistently `TypeID=5` ("SP to MM to CUS"), `INVTypeID=1` (EXPORT-CTN), `CurrencyID=2` (USD). `TSX` documents run right up to the present (June 2026), so this is an ongoing trade-route-specific prefix, not dead legacy data. |
| 2026-07-04 | v4.4 | Opened `googleCls.vb` — found the actual function is `UploadPOAuto()` (not "UpdateLoadPOAuto"), a Google Drive attachment-upload subsystem covering 8 document types (SOIPI, PRPO, SOIPO, SOIPIPO, INVOICE, STX, CostVat, and default/PO), driven by a `needUploadPO` queue table. Documented the full upload/path-translation/DB-writeback flow, the per-type destination tables (`img_soi`, `img_ProductPO`, `img_soipo`, `img_soipipo`, `img_inv`, `stxapprove`, `img_PO`), and 2 new gotchas: hardcoded Google OAuth Client ID/Secret in source (3rd hardcoded-credential finding in this codebase, alongside `sa` in ConStr.vb and network creds in ImportACCT), and pervasive SQL-string-concatenation (injection-risk pattern, though inputs are internal). Added new §12.5. |
| 2026-07-04 | v4.5 | Confirmed the business purpose of the `SOIPO`/`SOIPIPO` attachment types (§12.5): whenever a customer issues their own PO to order goods, that customer PO document is scanned/photographed and uploaded as evidence, attached directly on `MF_SaleOrder`. Found the 3 parallel attachment slots on `MF_SaleOrder`: `AttachFileName`/`AttachFilePath` (base, DocType `SOIPI`), `AttachFileNamePO`/`AttachFilePathPO` (customer's PO specifically, DocType `SOIPO`), `AttachFileNamePIPO`/`AttachFilePathPIPO` + `PIPOReceivingDate` (combined PI+PO with a receiving-date confirmation, DocType `SOIPIPO`) — verified with real uploaded files (e.g. `PO6905319 กุ้งขาวใหญ่016 มายฟู้ดส์.pdf`). |
| 2026-07-09 | v4.7 | **Major cost/cycletime/loss/DL audit — 6 SPs reviewed, 1 critical bug found+fixed+deployed, 2 process bugs found+fixed (fix files delivered, deployment not yet confirmed), 2 minor inconsistencies flagged (not fixed).** (1) 🔴 **`CalculateNewCost`/`Create_Recal_Cal` CycleTime sign bug** — the `cy` subquery re-declared `pi` against raw `produceitem` (unsigned kg) instead of the outer signed-kg CTE, so REMAINING kg was ADDED instead of SUBTRACTED; verified against `PR260626-0021` (CycleTime 1105.983 → correct 38.013, ~29× inflated). Proven algebraically that the fix only ever DECREASES CycleTime, never increases it (`New = Old − 2×Σ(cycletime×kg)_REMAINING`). Both SPs patched and deployed 2026-07-09 (confirmed via `sys.objects.modify_date`). (2) Discovered `Recal_Cal` is a **cached physical table**, not a live view — after redeploying `Create_Recal_Cal`, the cache still held pre-fix values until `EXEC Create_Recal_Cal` (or `UpDateRecalCost`) actually ran again, causing a "fix deployed but nothing changed" false alarm. (3) Documented that CycleTime, `DLThbKg`, and `LGCost` are 3 parallel accumulation systems that inherit via the production chain (BUY lot → production → production → ...) — CycleTime and DLThbKg both add a "new this round" term (`manhour`-based) on top of the inherited weighted-average; LGCost is pure pass-through with no new-this-round term. Confirmed the 3 are fully independent (separate stocktrx columns, separate SPs, no cross-reference) — the CycleTime fix has zero effect on DLThbKg or LGCost. (4) Audited `UpDateRecalDL`/`RecalDL` — formula correctly sign-flips REMAINING inline (no CycleTime-style bug); flagged 2 minor unfixed inconsistencies: `ph.Status=1` filter present in `RecalDL` but missing from `UpDateRecalDL`'s inline duplicate query, and `RecalDL`'s `val` subquery joins only on `ProductionID` (not `ProductionItemID`) risking fan-out for multi-output productions. (5) Audited `CalculateNewLoss` — `AccLoss%`/`NewLoss` formula correctly sign-flips REMAINING; found and fixed 2 process bugs (fix files delivered, not yet confirmed deployed): unguarded `DROP TABLE pdloss` fails on a cold session with no prior run; dead/unused `@inputKGRevert`/`@remainKGRevert` variables (computed, never consumed — left as-is, harmless). Also found the output column `'AvgInputloss'` actually selects `@InputLoss` (gross-kg-equivalent quantity) not `@AvgInputLoss` (the real average-loss percentage) — mislabeled but harmless since `UpDateRecalLoss` only consumes `[AccLoss%]`. (6) Found and fixed `UpDateRecalLoss` infinite-loop risk: the `else if (@rnd=2)` branch ("some productions won't converge") zeroed `@newCNT` but never set `@rnd`, so `while(@rnd<4)` could spin forever if the flagged-row count plateaus at that exact point — fixed by adding `set @rnd = 6` to match the sibling exit branch. (7) Confirmed the STOCKTRX/ProduceHD indexes proposed for `RecalLoss` performance (§14) are now deployed (`IX_stocktrx_production_join`, `PK_ProduceHD_ProductionID`). Proposed but **not yet deployed**: pushing `@productionid` filter into `CalculateNewCost`'s internal subqueries (currently filters only via a final `HAVING`, so each of the potentially hundreds of per-production cursor calls inside `UpDateRecalCost` risks aggregating across ALL productions before discarding all but one row), and eliminating `UpDateRecalCost`'s redundant double-call of `Create_Recal_Cal` per loop iteration (calls it twice — once for `@oldCNT`, once for `@newCNT` — when the second call's result equals the next iteration's first call). The pre-existing `AvgSMCost` fan-out bug (§15) remains unresolved. Expanded §4 into full subsections (4.1–4.9) to document all of this. |
| 2026-07-09 | v4.8 | **Deployment confirmation**: `fix_CalculateNewLoss.sql` (DROP TABLE guard) and `fix_UpDateRecalLoss.sql` (infinite-loop branch fix) from v4.7 are now confirmed deployed — verified both via `sys.objects.modify_date` (13:05:21 / 13:05:52) and by re-reading live `OBJECT_DEFINITION` to confirm the actual guard clause / `set @rnd = 6` are present in the deployed code, not just that *something* changed. Also ran an end-to-end re-check of `PR260626-0021` (the original CycleTime bug example, §4.3): `stocktrx.cycletime` now correctly reads 38.013 (was 1105.983), confirming the full `CalculateNewCost → Create_Recal_Cal → Recal (view) → UpDateRecalCost` correction pipeline works in practice, not just in isolated SELECT-level testing. `UpDateRecalCost` itself required no code change — its behavior was entirely inherited from `CalculateNewCost`/`Create_Recal_Cal`, both already fixed in v4.7. |
| 2026-07-09 | v4.9 | **New finding**: `UpdateRecalCost`, `UpdateRecalDL`, `UpdateRecalLoss`, and `UpdateCostFromRootlotno` (all audited in §4) are confirmed via the `tracking` table to be **daily scheduled automation tasks**, not manual/on-demand-only SPs as previously assumed — run counts in the 800-1,400+ range each, most recent run 2026-07-09. Not previously listed in §12.1's example task list or §13.3's Balance Sheet task list; added new §12.1.1. Practical implication: the §4.3/§4.7 bug fixes take effect automatically on the next scheduled daily run with no separate manual re-trigger needed going forward. Exact schedule time-of-day for these 4 tasks remains an open question. |
| 2026-07-09 | v5.0 | **New topic, new §17**: discovered and documented `UpdateApproveProduction` (the real production-approval gate, previously unknown to this KB) — carries 2 fixes dated 2026-07-08 (race-condition lock via `sp_getapplock`, and a post-insert negative-balance guard that blocks approval if ANY row on any touched rootlotno is negative, not just this production's own rows). Diagnosed and fixed 2 real approval failures (`PR090726-0002`, `PR090726-0006`) — both traced to the SAME root cause pattern: a pre-existing negative `StockLotBalanceUnit` left behind by a DIFFERENT, OLDER production that was approved before the 2026-07-08 guard existed (so it slipped through undetected — no contradiction, the guard simply didn't exist yet). Established and applied a 2-step remediation pattern: (1) INSERT compensating `ADJUST STOCK IN` sourcing cost fields from the same rootlotno, (2) UPDATE the original negative row's `StockLotBalanceUnit` to 0 as a "balance override" — **both steps required**, since the guard counts every historical row with a negative balance, not the net/current total; an offsetting insert alone does not stop the guard from tripping on the old row. Also corrected an in-session wrong hypothesis: `Active=0` on an IN/MOVEIN row is normal system behavior once consumed (not a "deactivated" marker) — confirmed by scanning a full 36-row rootlotno chain where this pattern holds almost universally; the real signal for this bug is a negative `StockLotBalanceUnit` on an `Active=1` row, not the `Active` value on the creating row. Flagged as an open item: no system-wide sweep for this exact anomaly pattern has been run yet — both cases found this session were discovered incidentally, one dating back to January 2026 (undetected for ~6 months), suggesting more likely exist. |
| 2026-07-10 | v5.1 | **New topic, new §17.6–17.9: `GetIncompletedProduction` and the produceitem/STOCKTRX row-count mismatch pattern.** Found and documented a previously-unknown diagnostic SP, `GetIncompletedProduction`, that flags any `ProductionID` where `COUNT(produceitem rows)` ≠ `COUNT(non-cancelled PRODUCTION-project STOCKTRX rows)`. Diagnosed and fixed 5 real flagged productions (`PR080626-0067`, `PR110626-0048`, `PR200526-0041`, `PR240626-0041`, `PR290626-0021`). **Critical data-hierarchy principle established by owner and now standing policy**: `produceitem` is the MASTER record of what actually happened on the production floor; `STOCKTRX` is the derived ledger and must always be corrected to match `produceitem` — NEVER the reverse (i.e. never delete/alter a produceitem row just to make the counts agree). Found the reliable join key for this diagnosis is `stocktrx.produceitemid` → `produceitem.ProductionItemID` (exact 1:1 link — more reliable than matching `produceitem.LOTNO` to `stocktrx.StockID` by value, since `ProductionItemID` alone is NOT a unique key — the same ID text is reused across many unrelated ProductionIDs, so any join/lookup on it MUST also filter by `ProductionID`). Documented 2 distinct sub-patterns requiring different fixes (§17.7). |
| 2026-07-10 | v5.2 | **New topic, new §17.10–17.14: root cause of the produceitem/STOCKTRX cancel-sync bug (application layer), and cascade-cancel SP design.** Traced the §17.6–17.9 bug to its true source by reviewing owner-supplied VB.NET source: the per-line cancel button (`BtnCANCEL_Click`) and the whole-production unapprove routine (`ProductionCls.UnApprove`) both funnel into generic `StockCls.CancelStockIN`/`CancelStockOUT`/`CancelStockMove` methods that only ever touch `STOCKTRX` (+ `Product.BalanceQty`) — never `produceitem` — confirmed by full source inspection (no line references the `produceitem` table in any of the three). Found direct evidence of intent-vs-implementation gap: `UnApprove` declares `Dim ObjProduceItem As New ProduceItemCls` and `Dim ObjProduceItemList As New List(Of ProduceItemCls)` at the top of the routine and **never uses either variable** — dead code strongly suggesting a produceitem-sync step was planned and never finished. Confirmed via `sys.sql_modules` search that `GetPreCanceltrx` (the existing downstream-chain-finder table-valued function) is called by **zero** stored procedures in the database — it is invoked directly from the application layer, consistent with the cancel logic living client-side rather than in SQL. Designed (not yet deployed — requires SSMS due to `CREATE OR ALTER`/`THROW`/`TRY-CATCH` being blocked by the mssql tool) a 3-part replacement: `sp_CancelStockTrxLine` (single-line cancel with produceitem sync built in for PRODUCTION-project rows), `sp_PreviewCancelChain` (dry-run, shows the full cascade before any change), `sp_ExecuteCancelChain` (owner-confirmed cascade execution, single all-or-nothing transaction, requires explicit `@Confirmed=1`). Verified the existing `GetPreCanceltrx` cascade logic live against a real 53-row chain (`StockRunning='01109344'`) spanning 6 productions and 18 days, surfacing a real downstream SELL invoice (`INV2026-06-D0636`) that would need reversing — confirming the preview-before-execute design is necessary in practice, not just in theory. **Key RM/FG vs SM cascade-scope rule established (owner-verified)**: RM/FG stock carries lot genealogy forward (an output rootlotno can become a later production's input rootlotno via MOVE), so cancelling a production requires unapproving every downstream production that consumed its output, transitively, until the chain terminates; SM (semi-finished/overhead) items are simple unit-count consumables with no rootlotno genealogy — cancelling a production that used an SM item only needs that item's specific lot balance restored (via the same generic reversal-insert mechanism, since each SM lot still has its own StockID/balance), never a cascade to whatever other productions separately drew from the same shared SM lot. Documented in `GetPreCanceltrx`'s own logic: the chain only widens to "every other line of the same ProductionID" when it hits a `PRODUCTION`+`OUT` row that must be unapproved as a whole; it does not widen across unrelated productions sharing a bulk/SM lot. |
| 2026-07-14 | v5.4 | **New topic, new §17.15: duplicate-MOVEOUT-within-a-single-move-batch pattern, and the real STOCKTRX CANCEL row structure (reverse-engineered from live examples).** Diagnosed a live approval failure on `PR140726-0059` — `chkproduction` (§17.2 Step 1) showed all 5 of its own lines OK, but Step 2 found a pre-existing negative-balance row (`StockID 070726-0156`, `rootlotno 070726-0144`, `StockLotBalanceUnit=-1`, `Active=1`) blocking the post-insert guard. Traced it to the same move document (`MV140726-0002`) as the production's own consumption: that rootlotno had 2 MOVEOUT rows but only 1 MOVEIN (every other rootlotno in the same 67-row move batch has exactly 1 MOVEOUT + 1 MOVEIN), meaning one MOVEOUT was a genuine duplicate/erroneous transaction — not a real physical shortfall like the §17.3 pattern. **New distinction established**: when a negative balance traces to a duplicate transaction with no real physical counterpart, the fix is to properly CANCEL the duplicate row using the exact mechanism the live application itself uses — NOT the §17.4 ADJUST STOCK IN + balance-override pattern (reserved for cases where a real physical unit was genuinely double-drawn and a compensating unit must be fabricated to match reality). Reverse-engineered the real STOCKTRX CANCEL-row shape from 5 live historical examples (`ProjectID='MOVE'`, `StockINOUT='CANCEL'`): same `StockID` as the row being cancelled, `StockPerUnit` negated, `StockRef` = the cancelled row's own `StockRunning`, `Remark` = the cancelled row's own `StockRunning` (not the literal text `'CANCEL'`), new row's own `StockLotBalanceUnit` recomputed net (here, 0), `Active=0` on the new CANCEL row. The original cancelled row is then UPDATEd: `Active=0`, `Remark='CANCEL'` (literal text, overwriting whatever the row's prior Remark was) — but its own `StockLotBalanceUnit` is left unchanged (confirmed from all 5 live examples), unlike §17.4's balance-override pattern which zeroes it. Applied live: cancelled `StockRunning 01141397` (the duplicate MOVEOUT) via new row `01141920`; confirmed the negative-balance guard check (§17.2 Step 2) returns 0 rows afterward. Also documented (owner-directed): `CostID`/`CostDetailID`/`InventoryCost` are only populated by the live application on the originating BUY/IN row of a rootlotno — every subsequent MOVEOUT/MOVEIN row the app itself generates leaves these NULL (confirmed on 2 genuine, untouched app-generated rows in the same chain, so this is normal system behavior, not a bug). Owner's standing preference when Claude manually inserts/corrects STOCKTRX rows on a rootlotno: backfill `CostID`/`CostDetailID`/`InventoryCost` on all touched rows to match that rootlotno's original BUY/IN row, even though the live app itself doesn't do this for ordinary MOVE rows — a deliberate divergence for manually-corrected rows, for cost-traceability. |
| 2026-07-13 | v5.3 | **New session: 3 more automation SPs audited via live diagnosis of real slow/hung runs (`Create_EmpAllocate`, `CreateBackUpProfitloss`, `UpDateRecalLoss`) — all 3 fixes delivered as SQL files, NONE confirmed deployed/rerun yet (pending owner execution in SSMS).** (1) `Create_EmpAllocate` (salary-to-invoice allocation): `sys.dm_exec_procedure_stats` showed wildly unstable runtime (60–90s typical, one historical run 28,232s / ~7.8hr) traced to `COLLATE THAI_CI_AS`-wrapped LEFT JOINs against `TSAmountKG` and `INVSellAmount`, both **zero-index heaps at the time** — reproducing just those two joins standalone timed out at 30s, confirming the bottleneck. Separately found a real logic bug: the line `--WHERE ec.employeeid = @EmpID` is commented out as a full line, so the next active line (`AND i.CustType <> 'Oversea'`) has no `WHERE` to attach to and silently becomes an extra `ON` condition of the last `LEFT JOIN` instead of a row filter — Part A never actually excludes Oversea rows, risking double-count against Part B. Mid-fix discovery: `INVSellAmount` is actually a **non-schema-bound VIEW**, not a table (`CREATE INDEX` on it fails with Msg 1939) — revised the fix to materialize it into an indexed temp table (`#Sell`) inside the procedure instead of altering the view. Delivered `Fix_Create_EmpAllocate.sql` (v2, supersedes v1) — adds `TSAmountKG` index, materializes `#Sell`, restores the real `WHERE`, adds `OPTION (RECOMPILE)`. (2) `CreateBackUpProfitloss`: root cause of the proc getting progressively slower found — `@NotOverDate` is `DECLARE`d but **never assigned** (stays NULL), so the intended "purge rows older than 30 days" step (`> @NotOverDate`) always evaluates against NULL and has **never deleted a single row**. Live-checked table sizes confirm it: `backupProfitloss_R2` = 30.48M rows (~13.9GB) spanning 37 days already (comment says intent was 30) and growing ~800K+ rows/day; `backupProfitloss` = 1.42M rows; `Backupprofitloss_Revenue` = 1.44M rows — both of the latter two are heaps with **zero indexes** on `trxdate`. Also found 5 dead `select *` debug statements with no consumer (one of them scans the full 30M-row R2 table every single run) and an unscoped `IF @cnt > 0` (missing `BEGIN/END` means only the first of 3 same-day-delete statements is actually conditional). **Retention window confirmed with owner: 30 days** (matches the original code comment). Delivered `Fix_CreateBackUpProfitloss.sql`: adds `trxdate` indexes on the two heap tables, a **batched** one-time catch-up purge (`DELETE TOP(50000)` loop, to avoid one giant transaction against a 13.9GB table), and a corrected proc (`@NotOverDate=30`, proper `BEGIN/END`, dead selects removed). (3) `UpDateRecalLoss`: confirmed the v4.7/v4.8 infinite-loop fix (`set @rnd=6` in the `@rnd=2` branch) is holding in production — `dm_exec_procedure_stats` shows 3 completed executions (17s / 1629s / etc.), none infinite anymore. New finding this session: the per-row `UPDATE stocktrx` inside the cursor filters `WHERE stockid IN (...) OR rootlotno IN (...)` — `rootlotno` has an index (`IX_stocktrx_rootlotno`) but **`stocktrx.StockID` has zero indexes**, forcing a near-full scan of `stocktrx` (1.14M rows) on the `stockid` branch for every flagged `recalLoss` row, repeated for up to 4 outer passes. Caught a real run live: only 24 source rows in `recalLoss`, yet 7.4+ minutes elapsed and 406,432 logical reads and still climbing (not blocked — `blocking_session_id=0` — just genuinely scanning); owner elected to `KILL` the session (confirmed autocommit-per-statement inside the proc, so no long rollback / no prior committed work lost) rather than let it run to historical worst-case (~27 min). Delivered `Fix_stocktrx_stockid_index.sql` (adds `IX_stocktrx_stockid`). Also manually inserted today's `tracking` row (`Type='UpdateRecalLoss'`, `Date='13-07-2026'`, `Time='7:36 AM'`) since the automated run didn't complete/log — confirmed existing `UpdateRecalLoss` tracking rows consistently use a `H:MM AM/PM` (12-hour, no zero-pad) format around 1:50–3:03 AM, distinct from the 24-hour-style timestamps seen on older/other `Type` values in the same free-text `Time` column (reconfirms v4.1's finding that `tracking.Date`/`Time` are unstandardized text, not real date/time types). **Common thread across all 3**: every one of these long-running automation SPs turned out to be slow/broken for a *table-hygiene* reason (missing index on a hot join/filter column, or a silently-broken purge letting a backup table grow unbounded) rather than a flaw in the actual business logic being computed — worth a proactive sweep of the other daily `tracking`-logged SPs (`UpdateRecalCost`, `UpdateRecalDL`, `UpdateCostFromRootlotno`, etc., §12.1.1) for the same 2 patterns. |
| 2026-07-20 | v5.5 | **New topic, new §18: `mfit.dyndns.biz:1433` LAN-vs-WAN routing investigation, resolved.** LAN clients connecting to the SQL Server hostname `mfit.dyndns.biz:1433` (server referenced throughout this KB per the header note) were being misrouted to the NAS (192.168.1.198) instead of the real SQL Server host "MF" (192.168.1.56), while external clients connected fine. Root cause: a Static DNS Configuration record on the router (Huawei HG8145X6, Network Application > DNS Configuration) intentionally answers this hostname with the NAS's IP for LAN queries (for the NAS's own unrelated uses) — DNS cannot differentiate by destination port, so port 1433 needs a TCP-level proxy on the NAS to reach MF instead. Found an existing but non-persistent ad-hoc `python3` proxy already doing this (started manually via SSH, not tied to any auto-start mechanism), which explains the prior intermittent "works sometimes" symptom. Verified working live (`TcpTestSucceeded: True`); a plan to make it permanent via a saved script + DSM Task Scheduler is documented but not yet executed. |


## TABLE OF CONTENTS

1. System Overview
2. Traceback Flow (Invoice → Stock → Production → Purchase → Cost)
3. Data Dictionary (core tables)
4. Cost / Cycletime / Loss Stored Procedures
5. Table Relationship Map (all tables, all domains)
6. Product Master & BOM (RM/SM/FG, Main/Alternative)
7. Sales Flow (SaleOrder → Invoice, Customer Type, Invoice Type)
8. NeedToBuy (production planning & purchasing) — incl. real VB.NET algorithm
9. Inventory Valuation formula
10. P&L Pipeline (SSIS ETL)
11. Account Subsystem (ACCT / TotalAcct / PR)
12. AutoEmail Scheduled Automation (VB.NET)
13. Balance Sheet Module (BalanceSheet, APIncomeStatement, ARBalanceSheet)
14. Master Lessons Learned / Gotchas (flat list, all critical warnings)
15. Open Questions / Known Gaps (do not answer these confidently)
16. KB Distribution Architecture & MCP Access (how AI clients read this document)
17. Production Approval & STOCKTRX Negative-Balance Remediation (`UpdateApproveProduction`, ADJUST STOCK fix pattern, `GetIncompletedProduction` row-count mismatches, cascade-cancel SP design)
18. Network / NAS DNS Redirect — `mfit.dyndns.biz` SQL Server LAN Routing (Static DNS root cause, TCP proxy workaround, permanent-fix plan)
19. Lot Numbering System (rootlotno / stockid, suffix-overflow fix)
20. CostVatHD / CostVatDetail Relationship to ADJUST STOCK Remediation Rows
21. MOVE-Category Negative-Balance Root Cause (`StockCls.GetStockAllQR` app-layer bug, fix delivered)
22. Sweep Completion — Items #29-44, New ADJUST STOCK Remark Convention, 5 New Sub-Patterns, `RootLotProduct` Trigger Enforcement Confirmed Live
23. Session 2026-07-21 (2nd pass) — GetIncompletedProduction Sub-Patterns D & Reversed, Cost-Field Audit, SELL Late-Invoice Pattern, `Create_MOVEINOUT` Permanent Guard Fix

---

## 1. SYSTEM OVERVIEW `[VERIFIED]`

| Database | Role | Key tables |
|---|---|---|
| `MildFoods` | Sales & Purchasing documents | MF_PO, MF_PODetail, MF_ProductPO, MF_Invoice, MF_InvoiceDetail, MF_Product, MF_Company, MF_Supplier, MF_SaleOrder, MF_SaleOrderDetail, MF_CustomerType, MF_Type, MF_INVType, MF_Currency |
| `inventory` | Inventory, production, real cost | STOCKTRX, produceHD, produceitem, CostVatHD, CostVatDetail, product, ProductionCostFix, BOMMain, BOMdetail, NeedToBuy, MoveHD, PR, PRProduct, ACCT_A–G, TotalAcct, TotalAcct_check |
| `STAGING` | P&L pipeline staging only | staging.pl.* (truncated/rebuilt every run) |

**Gotcha**: The same conceptual entity ("product") has two completely different ID systems: `MF_Product.ProductID` (int, sales-side, e.g. `2019`) vs `inventory.dbo.product.ProductID` (nvarchar RM/FG code, e.g. `177.004.001` or `DGS010`). They are bridged via `MF_Product.ProductCode_MildFoods = inventory.product.ProductID`, requiring `COLLATE THAI_CI_AS` on both sides for the join to work reliably.

---

## 2. TRACEBACK FLOW `[VERIFIED]`

Full path from a sale back to raw-material cost, validated end-to-end on real invoice `INV2026-03-D0248`:

```
MF_Invoice / MF_InvoiceDetail
   │  DocNO = 'INV2026-03-D0248'  (join via InvoiceID)
   │  ActiveFlag='A' only counts as real sale (ActiveFlag='R' = revised/excluded)
   ▼
STOCKTRX  (OUT, ProjectID='SELL')
   │  Linked to invoice via Remark = invoice DocNO   ⚠ NOT the InvoiceNo column (that stores something else)
   │  StockPerUnit negative = quantity removed from stock
   │  rootlotno / productionid point to the production lot that made this stock
   ▼
produceHD / produceitem  (Type='OUTPUT')
   │  ProductionID links to STOCKTRX.productionid
   │  produceitem also carries rootproductid/rootlotno for the INPUT side
   ▼
STOCKTRX (Type='INPUT', ProjectID='production')
   │  Raw material consumed by production (Cost/InventoryCost stamped at BUY-in time)
   ▼
rootlotno has 3 possible origins (ProjectID column tells which):
   1) 'BUY'          → came from a purchase order
   2) 'production'   → created as OUTPUT of a production run (new finished good)
   3) 'adjust stock' → manual inventory count correction (annual year-end count, see §3.1.2)

── For ProjectID='BUY' rows ──
STOCKTRX (IN)  →  StockDocNo = PO document number
   ▼
MF_PO (DocNO) → MF_PODetail (POID) → MF_ProductPO
   ⚠ MF_PODetail.UnitPrice/TotalPrice is usually 0 — NOT the real cost
   ⚠ STOCKTRX.PODetailID is ALWAYS 0 in real data — never join on it directly
   ▼
CostVatHD / CostVatDetail   ★ REAL COST SOURCE (RMCost / InventoryCost)
   join via PODocno + lot (pattern: {ProductCode}-{DDMMYY})
```

### 2.1 Verified example: cost consistency across the whole chain

| FG ProductID | Production | Root RM | RMCost (CostVatDetail) | STOCKTRX Cost (BUY) | OutputCost (CalculateNewCost) |
|---|---|---|---|---|---|
| 177.004.001 (White Sesame) | PR050326-0048 | DGS010 | 70 | 70 | 70 ✅ |
| 177.004.003 (Mung Bean) | PR050326-0047 | DGS001 | 36 | 36 | 36 ✅ |

All three numbers match exactly for both cases — the cost pipeline is internally consistent for this invoice.

### 2.2 Root product traceback mechanics

Find the root RM of an FG by joining `produceitem` INPUT rows to OUTPUT rows of the same `ProductionID`:

| Output (FG) | Production | Input lots | Root RM | Total KG | Ratio |
|---|---|---|---|---|---|
| 177.004.001 (White Sesame) | PR050326-0048 | 20 lots (050326-0095…0114) | DGS010 | 600 KG (20×30) | 1:1 |
| 177.004.003 (Mung Bean) | PR050326-0047 | 10 lots (050326-0085…0094) | DGS001 | 300 KG (10×30) | 1:1 |

Note: root product name matches output name here (repack, not transformation) and `cycletime=0` for both — this is a simple re-pack from bulk RM lot to carton lot, not a manufacturing transformation.

### 2.3 One PO, multiple root products (verified against `PO2026-03-00139`)

All rootlotnos above trace back to `ProjectID='BUY'`, `StockDocNo='PO2026-03-00139'` — a single PO that ordered 3 different RM items at once:

| ProductCode (RM) | Name | MF_PODetail.AmountCarton | CostVatDetail.lot | RMCost (THB/kg) | TotalAmount |
|---|---|---|---|---|---|
| DGS001 | Mung bean | 300 | DGS001-050326 | 36 | 10,800 |
| DGS009 | Black sesame | 300 | DGS009-120326 | 110 | 33,000 |
| DGS010 | White sesame | 600 | DGS010-050326 | 70 | 42,000 |

**Gotcha**: `MF_PODetail.UnitPrice/TotalPrice` = 0 for all 3 lines on this PO (never filled in at order time), while `CostVatDetail` has the real cost for every one — reconfirming CostVatDetail as the only trustworthy cost source. Also: DGS009 has `stocktrxdate` 12/03/2026, different from DGS001/DGS010's 05/03/2026, even on the same PO — goods receipt can happen in separate batches for one PO.

### 2.4 MOVE — inventory transfer between locations

`ProjectID='MOVE'` in STOCKTRX = moving stock between `StockPlace` locations. Linked to `MoveHD` via `MoveID`.

```
MoveHD (MoveID, Destination, Checker, Docno, Logistic, Trxdate)
   │  header only — destination/checker/logistics/date
   ▼
STOCKTRX × 2 rows per move (join via StockDocNo = MoveHD.MoveID)
   ├─ MOVEOUT at source StockPlace   (StockPerUnit negative, original StockID)
   └─ MOVEIN  at destination StockPlace (StockPerUnit positive, NEW StockID)
          rootlotno identical on both rows — only StockID/StockPlace differ
```

**Gotcha**: There is NO separate `MoveDetail` table — the line-item detail (what moved, how much) lives directly in `STOCKTRX` itself as the MOVEOUT/MOVEIN pair. `MoveHD` provides only header context.

### 2.5 Query pattern: "what FG does input X convert into, and how much?" — ⚠ FAN-OUT BUG RISK

This is a common support question. A naive query is **catastrophically wrong** if not written carefully.

**❌ WRONG (fan-out bug)**:
```sql
SELECT po.ProductID, SUM(po.kg)
FROM produceitem pi
JOIN produceitem po ON pi.ProductionID = po.ProductionID AND po.type='OUTPUT'
WHERE pi.type='INPUT' AND pi.rootproductid = 'DSH016'
GROUP BY po.ProductID
```
If one production run has the same RM as INPUT across many lot rows (seen up to **169 rows in a single production run**), this join multiplies every OUTPUT row once per matching INPUT row, and `SUM` adds the same output repeatedly.

**Real incident**: computing DSH016 → 305.001.001 for 2025-2026 this way gave **644,090 KG**. The correct answer is **5,030 KG** — off by ~128×. 476 production runs in the system have this multi-row-INPUT characteristic and are at risk.

**✅ CORRECT**:
```sql
SELECT po.ProductID, SUM(po.kg)
FROM (SELECT DISTINCT ProductionID FROM produceitem
      WHERE type='INPUT' AND rootproductid='DSH016') dpi
JOIN produceitem po ON dpi.ProductionID = po.ProductionID AND po.type='OUTPUT'
GROUP BY po.ProductID
```

**General principle**: whenever joining a table that may have multiple rows per parent key (produceitem INPUT lots, MF_PODetail lines, etc.) to another table through that shared key, `DISTINCT` the many-side first. `GROUP BY` alone does NOT remove fan-out duplicates — it only groups the (already-duplicated) results.

**Verified correct output** (2025-2026, FG category only):

| FG | Name | Total KG (2025+2026) |
|---|---|---|
| 305.001.001 | Dried Farm Shrimp Size:M | 5,030 |
| 211.001.003 | 8A(TIGER) Dried Shrimp | ~3,000+ |
| 302.001.001 | Dried Farmed Shrimp Mix 7A+8A | ~2,000+ |
| (8 more minor pack-size SKUs) | | small quantities |

Cross-validated against actual invoice data for 305.001.001 in the same period (~5,400 KG sold) — production and sales figures reconcile within a reasonable margin.

### 2.6 STOCKTRX — Quick Reference by ProjectID (all transaction types in one place)

Consolidated view of every `ProjectID` value and its exact join point, for quick lookup:

```
┌─ 'BUY' ──────────────────────────────────────────────────────┐
│  MF_PO.DocNO ──(StockDocNo)──> STOCKTRX (IN)                   │
│  ⚠ NOT PODetailID (always 0)                                   │
│  → CostVatDetail (lot={ProductCode}-{DDMMYY}) ★ real cost       │
└─────────────────────────────────────────────────────────────┘

┌─ 'production' ───────────────────────────────────────────────┐
│  produceHD/produceitem ──(productionid/produceitemid)──>      │
│                                            STOCKTRX (IN/OUT)   │
│  INPUT = raw material consumed / OUTPUT = finished good made    │
└─────────────────────────────────────────────────────────────┘

┌─ 'SELL' ─────────────────────────────────────────────────────┐
│  MF_Invoice.DocNO ──(Remark)──> STOCKTRX (OUT)                  │
│  ⚠ NOT InvoiceNo (holds a different reference)                 │
└─────────────────────────────────────────────────────────────┘

┌─ 'MOVE' ─────────────────────────────────────────────────────┐
│  MoveHD.MoveID ──(StockDocNo)──> STOCKTRX × 2 paired rows       │
│     ├─ MOVEOUT at source (StockPerUnit negative, orig StockID) │
│     └─ MOVEIN at destination (StockPerUnit positive, new StockID)│
│  ⚠ No separate MoveDetail table — detail lives in STOCKTRX itself│
│  rootlotno identical on both rows — only StockID/StockPlace differ│
└─────────────────────────────────────────────────────────────┘

┌─ 'ADJUST STOCK' ─────────────────────────────────────────────┐
│  No FK to any other document — triggered by physical stock count│
│  Real example: StockPlace='MF_Check Stock'                     │
│                Remark='เช็คสต็อค2026' (names the count event)   │
│                dated 31/12 (year-end) is the dominant pattern   │
│  One of 3 origins of a fresh rootlotno (with BUY, production)   │
└─────────────────────────────────────────────────────────────┘

┌─ 'RETURN' / 'PREMIUM' (linked to SELL via ClaimID) ─────────────┐
│  ClaimID format: CL{MMYY}-{seq} e.g. CL0321-0001                │
│  Ties together every row belonging to one claim event:          │
│     - original SELL row (OUT)                                   │
│     - RETURN row (IN, customer sends goods back)                │
│     - PREMIUM row (OUT, goodwill/replacement stock given)        │
│     - CANCEL row(s) if any cancellation occurred in the chain    │
└─────────────────────────────────────────────────────────────┘

┌─ Cancel flow (cuts across every ProjectID above) ───────────────┐
│  Never modifies the original row — inserts a new reversal row    │
│  original row: Remark='CANCEL' (literal string)                 │
│  new row: StockINOUT='CANCEL', Remark=StockRunning of orig. row  │
│  ⚠ queries must filter BOTH: StockINOUT<>'cancel' AND Remark<>'cancel'│
└─────────────────────────────────────────────────────────────┘
```

---

## 3. DATA DICTIONARY `[VERIFIED]`

### 3.1 `inventory.dbo.STOCKTRX` — all stock movement (buy/produce/sell/move/adjust)

| Column | Type | Meaning |
|---|---|---|
| StockID | nvarchar | Sub-lot code (differs from rootlotno after a move/split) |
| ProjectID | nvarchar | Transaction type: BUY, production/PRODUCTION, SELL, MOVE, ADJUST STOCK, RETURN, PREMIUM `[VERIFIED — RETURN/PREMIUM confirmed 2026-07-04]` |
| StockINOUT | nvarchar | IN / OUT / MOVEIN / MOVEOUT / CANCEL |
| InvoiceNo | nvarchar | ⚠ Does NOT reliably store the main invoice number — holds other doc refs |
| Remark | nvarchar | In practice stores the sale invoice DocNO (join target for MF_Invoice.DocNO); ALSO used to store the literal string `'CANCEL'` or a `StockRunning` reference number — see §3.1.1 cancel flow |
| Cost / InventoryCost | decimal | Per-unit cost (THB/kg) stamped at transaction time |
| productionid / produceitemid | nvarchar | Points to produceHD/produceitem when row relates to production |
| rootproductid / rootlotno | nvarchar | True origin lot — used for backward tracing |
| PODetailID | int | ⚠ Always 0 in real data — never trust as FK to MF_PODetail.PODetailID |
| cycletime, smcost, LossKG, LGCost | decimal | Feed CalculateNewCycletime / CalculateNewLoss |
| StockPerUnit | decimal | Quantity for THIS transaction (signed: + IN, − OUT) |
| StockPerKG | decimal | For FG: 1 = carton-based unit (multiply by NetWeight); ≠1 = actual per-unit weight already (use BagSize/1000 instead) |
| StockLotBalanceUnit | decimal | Remaining balance of this specific lot line RIGHT NOW (not the original transaction qty) |
| Active | int | Mostly 0 or 1 (one stray -49 row seen, negligible) |
| SupplierID / SupplierName | int/nvarchar | Populated for ProjectID='BUY' rows; SupplierName may be abbreviated vs MF_Supplier's full legal name |
| ClaimID | nvarchar | `[VERIFIED 2026-07-04]` Format `CL{MMYY}-{seq}` e.g. `CL0321-0001`. Links together the multiple STOCKTRX rows involved in a single customer claim/return event (the original SELL row, a RETURN row bringing stock back IN, sometimes a PREMIUM row for goodwill/replacement stock, and CANCEL rows) — all rows sharing one claim carry the same ClaimID. |
| StockPlace | nvarchar | `[VERIFIED 2026-07-04]` **No clean master list exists** — over 100 distinct free-text values found, mixing structured warehouse-zone codes (e.g. `MF_(C1)_5`, `MF_(D1)_2`) with ad-hoc one-off labels for stock-count events (e.g. `checkstock251023`, `chek stock shopee 1.12.23` — note the typo, `MF_Check Stock` for annual physical counts). Do not assume this field is a clean enum; treat it as free text when querying. |

**Gotcha**: `PODetailID = 0` always — must join PO via `StockDocNo = MF_PO.DocNO` then match product code manually.

### 3.1.1 STOCKTRX cancel flow `[FULLY VERIFIED — resolved 2026-07-04]`

Cancelling a transaction does **NOT** modify the original row in place. It inserts a **new row** that reverses the original row's quantity, and cross-references it via `Remark`. Verified real example — full lifecycle of `StockID='310720-0002'` (ordered by `StockRunning`, the true chronological sequence key):

| StockRunning | ProjectID | StockINOUT | StockPerUnit | Remark | Meaning |
|---|---|---|---|---|---|
| 00072382 | PRODUCTION | IN | +2 | C310720-2 | Original stock created from production |
| 00115110 | SELL | OUT | -2 | **CANCEL** | A sale that is being cancelled — note Remark holds the literal string `'CANCEL'`, not an invoice number |
| 00115112 | SELL | **CANCEL** | +2 | **00115110** | The reversal row — positive quantity undoes the -2 above; `Remark` holds the `StockRunning` of the row it reverses |
| 00141861 | SELL | OUT | -2 | INV2021-07-D332 | A fresh, legitimate sale against a real invoice, after the cancellation |

**Implication for queries**: this is exactly why every cost/inventory query in this document filters `WHERE StockINOUT <> 'cancel' AND Remark <> 'cancel'` — the pair of rows (original marked `Remark='CANCEL'`, plus the new reversal row with `StockINOUT='CANCEL'`) must BOTH be excluded, otherwise the net effect is correct (they cancel out arithmetically) but including "cancelled-sale" noise rows in an unaggregated per-transaction report would be misleading.

### 3.1.2 ProjectID='ADJUST STOCK' `[VERIFIED 2026-07-04]`

Real example: rows with `ProjectID='ADJUST STOCK'`, `StockPlace='MF_Check Stock'`, `Remark='เช็คสต็อค2026'` (i.e. "2026 stock check"), dated `31/12/2025` (year-end). This confirms adjust-stock entries are generated during the **annual year-end physical stock count** reconciliation, tagged with a Remark describing which stock-count event they belong to and a dedicated `StockPlace='MF_Check Stock'` marker (not a real warehouse location).

### 3.2 `inventory.dbo.produceitem`

| Column | Type | Meaning |
|---|---|---|
| ProductionID | nvarchar | Production run ID, links to produceHD and STOCKTRX.productionid |
| ProductionItemID | nvarchar | Line-item ID, links to STOCKTRX.produceitemid |
| Type | nvarchar | INPUT / OUTPUT / REMAINING / SUBMAT |
| KG / Carton | decimal | Quantity |
| rootproductid / rootlotno | nvarchar | Origin lot of this line (both input and output sides) |

### 3.3 `MildFoods.dbo.MF_Invoice` / `MF_InvoiceDetail`

| Column | Type | Meaning |
|---|---|---|
| InvoiceID | int | PK of MF_Invoice |
| DocNO | nvarchar | e.g. `INV2026-03-D0248` — join target for STOCKTRX.Remark |
| ActiveFlag ('A'/'R') | char | 'A' = active/counted, 'R' = revised/excluded from totals |
| ProductID (detail) | int | → MF_Product.ProductID (NOT the RM code) |
| SellQty | decimal | Sale quantity (can be 0 or negative depending on front-end logic) |
| TotalUSD / UnitPriceUSD | decimal | ⚠ Name has "USD" but value currency depends on invoice's actual CurrencyID — do not assume |
| UnitPrice / TotalPrice | decimal | Used when TypeID flow does NOT go through MM (see §7.3) |
| ActualUnitPriceUSD / ActualTotalUSD | decimal | Used when TypeID flow DOES go through MM (see §7.3) |
| TypeID | int | → MF_Type (trade route) |
| INVTypeID | int | → MF_INVType (market + unit basis) |
| CurrencyID | int | → MF_Currency — the ONLY reliable currency indicator |

**Gotcha**: Header `TotalPrice`/`SubTotal` = SUM of detail `TotalUSD` for `ActiveFlag='A'` rows ONLY — not a straightforward sum of all detail rows, and not from the detail-level `TotalPrice` column.

**`PI` document prefix** `[RESOLVED — likely legacy, 2026-07-04]`: Only 2 invoices in the entire system use a `PI` prefix instead of `INV` — `PI19-001` (2019) and `PI2020-03-EX001` (2020), both from the system's earliest years. Both have the same `TypeID`/`INVTypeID` as ordinary invoices, with no other distinguishing structural difference found. Most likely explanation: an early/legacy numbering convention used before the system standardized on the `INV` prefix, not an active special document type — but this has not been confirmed with the system owner.

### 3.4 `MildFoods.dbo.MF_PO` / `MF_PODetail` / `MF_ProductPO` / `MF_Supplier`

| Column | Type | Meaning |
|---|---|---|
| MF_PO.DocNO | nvarchar | e.g. `PO2026-03-00139` — join target for STOCKTRX.StockDocNo |
| MF_PO.CompanyID | int | ⚠ MISLEADING NAME — actually references `MF_Supplier.SupplierID`, NOT `MF_Company` |
| MF_PODetail.ProductCode | nvarchar | RM/SM code (e.g. DGS001, DGS010) matching inventory.dbo.product.ProductID |
| MF_PODetail.UnitPrice / TotalPrice | decimal | ⚠ Usually 0 — order-time reference price, NOT real cost |
| MF_ProductPO | — | Product spec master referenced at PO time — not a cost source |
| MF_Supplier.SupplierID | int | PK; matches MF_PO.CompanyID and STOCKTRX.SupplierID for ProjectID='BUY' rows |
| MF_Supplier.SupplierName | nvarchar | Full legal name — STOCKTRX.SupplierName may store an abbreviation |
| MF_Supplier.SupplierCode | nvarchar | Short code, e.g. 'SUKSAWAT' |

**Verified chain**: `PO2026-03-00139` → `MF_PO.CompanyID=7` → `MF_Supplier.SupplierID=7` = "บริษัท สุขสวัสดิ์ พืชผล จำกัด" → `STOCKTRX.SupplierID='7'`/`SupplierName='สุขสวัสดิ์'` for the BUY rows of that PO. All match.

**Gotcha**: Supplier is only recorded at the PO header level — every line item on one PO shares the same supplier (no per-line SupplierID in MF_PODetail).

### 3.5 `inventory.dbo.CostVatHD` / `CostVatDetail` ★ real cost source

| Column | Type | Meaning |
|---|---|---|
| CostVatHD.PODocno | nvarchar | → MF_PO.DocNO |
| CostVatDetail.PODocno | nvarchar | Same PO as header |
| CostVatDetail.lot | nvarchar | Pattern `{ProductCode}-{DDMMYY}` e.g. `DGS010-050326` — matches STOCKTRX.rootlotno/ProductCode |
| RMCost | decimal | Per-unit RM cost (THB/kg) — this is the value stamped into STOCKTRX.Cost |
| InventoryCost / IncomeCost / PurchasingCost | decimal | Various cost views, normally = RMCost unless SharedCost applies |
| SharedCost / ratio | decimal | Used when cost is shared/allocated across lots/POs — non-zero SharedCost or ratio≠1 means non-trivial calculation |
| CostType | nvarchar | 'Main' seen in examples |
| Objective | nvarchar | e.g. 'BUY' — matches STOCKTRX.ProjectID |

**Golden rule**: if STOCKTRX cost looks wrong, check CostVatDetail first — never MF_PODetail.

**VAT/WHT columns** `[PARTIAL — verified columns exist, values look anomalous]`: `CostVatHD` has `TotalVatAmount` and `TotalWHTAmount` (both decimal). Checked real rows with non-zero `TotalVatAmount` — e.g. `PODocno='PO2026-05-00293'`, `TotalAmount=52,804.50`, `TotalVatAmount=0.099`. The VAT figure looks far too small to be a standard 7% VAT amount on that total (~3,696 expected) — it may represent a rate/ratio rather than a currency amount, or apply to a different base than TotalAmount. `TotalWHTAmount` was 0 in every row checked. **Do not assert a specific VAT/WHT calculation to a user based on this alone** — the exact formula/meaning of these two columns still needs clarification from the system owner.

---

## 4. COST / CYCLETIME / LOSS / DL STORED PROCEDURES `[MAJOR AUDIT 2026-07-09]`

### 4.1 Overview

| SP | Computes | Called by | CASE-sign-flip status |
|---|---|---|---|
| `CalculateNewCost` | OutputCost, OutputInventoryCost, SMCost, CycleTime, LGCost (per `@productionid`) | `UpDateRecalCost` cursor (row-by-row) | CycleTime bug `[FIXED, deployed 2026-07-09]`; others correct |
| `Create_Recal_Cal` | Same 5 metrics for ALL productions at once → physical table `Recal_Cal` (detection cache) | `UpDateRecalCost` (top + bottom of each loop iteration) | Same CycleTime bug, same fix, `[FIXED, deployed 2026-07-09]` |
| `UpDateRecalCost` | Orchestrates: `UpDateCostFromRootlotno` → `UpdateRemainingCost` → loop up to 10× (`Create_Recal_Cal`, cursor over view `Recal`, per-row `CalculateNewCost`, `UPDATE stocktrx`) | Manual/scheduled | n/a — pure orchestration, inherits correctness from the 2 SPs above |
| `CalculateNewLoss` | `AccLoss%`, `NewLoss`, `AvgInputLoss` (per `@productionid`) | `UpDateRecalLoss` cursor | Correct — sign-flip done inline via `CASE WHEN pi.type='INPUT' THEN kg ELSE -kg END` throughout |
| `UpDateRecalLoss` | Orchestrates loss recalc via view `RecalLoss`/`Recal` naming — loops up to 4×, cursor over `recalLoss`, per-row `CalculateNewLoss`, `UPDATE stocktrx.losskg` | Manual/scheduled | n/a |
| `UpDateRecalDL` | Orchestrates DL (direct-labor) cost recalc via view `RecalDL` — no separate `CalculateNewDL` SP, formula is inlined directly in the cursor loop | Manual/scheduled | Correct — sign-flip done inline |

**Design note**: `SellCompleted` filtering is intentionally applied ONLY at the Profitloss_R2 stage — upstream production-level SPs (CalculateNewCost etc.) do not filter on it.

**Tooling gotcha**: `EXEC` of a stored procedure directly via the mssql MCP tool is blocked when using the read-only `execute_read_query` tool (rollback-wrapped) — inline the SP body as a plain `SELECT` with the parameter substituted to verify results in that mode. When `execute_write_query` is enabled (`MSSQL_ENABLE_WRITES=true`), `EXEC` and DDL (`CREATE OR ALTER PROCEDURE` etc.) run directly — confirmed 2026-07-20, §17.16.

### 4.2 `CalculateNewCost` — formula breakdown

Base CTE `pi` = `produceitem` rows of `type IN ('INPUT','REMAINING')`, signed: `INPUT → +kg`, `REMAINING → -kg`. Joined to `stocktrx` filtered to `stockinout<>'cancel', remark<>'cancel', projectid='production'`. `op` = output KG excluding items with a `ProductionCostFix` override; `fc` = the FixCost pool for items that DO have an override (subtracted before allocation).

| Output | Formula | Notes |
|---|---|---|
| `OutputCost` | `(Σ(pi.kg×st.cost) − FixCost) / op.outputKG` | |
| `OutputInventoryCost` | Same, using `st.InventoryCost` | Parallel valuation, same allocation logic |
| `SMCost` | `(Σ(pi.kg×st.smcost) − FixCost)/outputKG + AvgSMCost` | `AvgSMCost` is a SEPARATE subquery over `produceitem.type='submat'` — see §4.9 for its known fan-out risk |
| `CycleTime` | `ph.manhour/outputKG + newCycleTime` | `newCycleTime` = weighted-avg cycletime inherited from INPUT/REMAINING lots — see §4.3 for the bug found here |
| `LGCost` | `Σ(pi.kg×st.lgcost)/outputKG` | No FixCost subtraction, no separate "new this round" term — pure pass-through (see §4.5) |

### 4.3 🔴 CRITICAL BUG — CycleTime sign-flip missing in nested subquery `[FOUND, FIXED & DEPLOYED 2026-07-09]`

**Root cause**: the `cy` subquery inside both `CalculateNewCost` and `Create_Recal_Cal` re-declares alias `pi` pointing directly at raw `produceitem` (unsigned `kg`) instead of reusing the outer signed-kg CTE:
```sql
-- BUGGY (both SPs had this identical subquery, copy-pasted):
sum(st.cycletime * kg) / op.outputkg    -- 'kg' here = produceitem.kg, ALWAYS POSITIVE
```
So REMAINING rows get **added** instead of **subtracted** — `INPUT×cycletime + REMAINING×cycletime` instead of `INPUT×cycletime − REMAINING×cycletime`.

**Verified against real data** (`PR260626-0021`: InputKG=4.68, RemainingKG=4.37, net consumed=0.31kg, lot cycletime=36.658):
```
Buggy:   (36.658×4.68 + 36.658×4.37) / 0.3 + 0.04/0.3 = 1105.850 + 0.133 = 1105.983  ← matched live EXEC result exactly
Correct: (36.658×4.68 − 36.658×4.37) / 0.3 + 0.04/0.3 =    37.880 + 0.133 =    38.013
```

**Direction of the fix — proven algebraically, not just empirically**: since `cycletime ≥ 0` and `kg ≥ 0` always,
```
New = Old − 2 × Σ(cycletime × kg)_REMAINING
```
→ **fixing this bug can only ever DECREASE CycleTime (or leave it unchanged if RemainingKG=0), never increase it.** The distortion is worst exactly when `RemainingKG` is close to `InputKG` (net consumption near zero) — this is a very common pattern (leftover from a lot that wasn't fully used), so expect the fix to touch a wide swath of productions system-wide, not just isolated cases.

**Fix**: replace `kg` with `(case when pi.type='INPUT' then pi.kg else -pi.kg end)` inside the `cy` subquery, in BOTH `CalculateNewCost` and `Create_Recal_Cal` (identical bug, copy-pasted code). **Both deployed 2026-07-09** — confirmed via `sys.objects.modify_date` (11:31:09 and 11:42:10 respectively).

**Why both SPs needed the fix, not just `CalculateNewCost`**: `Create_Recal_Cal` builds the detection cache (`Recal_Cal`) that the view `Recal` compares against live `stocktrx` to decide what `UpDateRecalCost`'s cursor should correct. If only `CalculateNewCost` were fixed, `Recal_Cal.cycleTime` would keep being computed wrong, would never match the now-correctly-updated `stocktrx.cycletime`, and every production with a REMAINING line would re-trigger on every future run forever (burning all 10 loop iterations every time, and burying genuine Cost/SMCost mismatches under thousands of false cycletime ones).

**AvgSMCost subquery was deliberately NOT touched by this fix** — it has a separate, previously-flagged fan-out risk (§4.9) unrelated to the sign bug; don't conflate the two when reviewing.

### 4.4 Gotcha: `Recal_Cal` is a cached physical TABLE, not a live view — must rebuild after redeploying `Create_Recal_Cal`

Hit this directly during verification: after deploying the fix above, `PR260626-0021`'s stocktrx cycletime was still 1105.983 with no change. Root cause: `Create_Recal_Cal` does `DROP TABLE Recal_Cal; SELECT ... INTO Recal_Cal` — a physical rebuild, not a view. `Recal_Cal`'s `modify_date` (00:13:58) was OLDER than the SP fix's `modify_date` (11:42:10), proving `Create_Recal_Cal` had not actually been re-executed since the fix went in — so the cache still held pre-fix values, `Recal_Cal.cycleTime` (stale, wrong) happened to equal `stocktrx.cycletime` (also still wrong from before), the view `Recal` saw no mismatch, and the cursor never touched the production.

**Rule going forward**: after ANY redeploy of `Create_Recal_Cal` (or `CalculateNewCost`), you must actually run `EXEC Create_Recal_Cal` (or the full `UpDateRecalCost`) at least once before checking whether a specific production got corrected — checking `Recal_Cal`/`Recal` immediately after an `ALTER` without a fresh execution will show stale, pre-fix data and falsely suggest the fix "did nothing."

### 4.5 Cost accumulation architecture — CycleTime, DLThbKg, LGCost all inherit via the production chain, independently

All three follow the same "own-metric-per-lot, weighted-averaged forward through the chain" pattern, but differ in whether they add a fresh term each round:

| Metric | "New this round" term | "Inherited from input lots" term | Has FixCost subtraction? |
|---|---|---|---|
| CycleTime | `ph.manhour / outputKG` | `Σ(cycletime × signed_kg) / outputKG` | No |
| DLThbKg | `(ph.manhour/mh.Totalmh) × mh.Amount / outputKG` (mh = `ManHourRate`, a monthly labor-cost-pool table) | `Σ(st.DLThbKg × signed_kg) / outputKG` | No |
| LGCost | **none** — pure pass-through | `Σ(st.lgcost × signed_kg) / outputKG` | No |

Because production output frequently becomes another production's input (multi-stage manufacturing), all three metrics **compound across the chain** — a value stamped on a deep upstream lot (BUY lot has none; the first internal production lot starts the chain) flows forward into every downstream production that consumes it, the same way the CycleTime bug's distortion would have propagated system-wide before the fix.

**Confirmed fully independent**: separate `stocktrx` columns (`cycletime`/`DLThbKg`/`lgcost`), separate correction SPs (`UpDateRecalCost`/`UpDateRecalDL`/none-separate-for-LG — LGCost is corrected as part of `UpDateRecalCost`'s same UPDATE statement), no subquery in any of the three formulas references either of the other two. **The CycleTime sign-bug fix (§4.3) has zero effect on DLThbKg or LGCost.**

`ManHourRate` (source of the DL "new this round" term) verified to have non-overlapping monthly date ranges (96 rows, 96 distinct `startdate`, sequential 26th-to-25th cycles) — no fan-out risk from that join.

### 4.6 `UpDateRecalDL` / `RecalDL` — audited, formula correct, 2 minor unfixed inconsistencies

Formula correctly sign-flips REMAINING inline (`case when pi.type='INPUT' then kg else -kg end`, done directly where used — never re-declares a raw-kg alias like the CycleTime bug did). No calculation fix needed.

Flagged but **not fixed** (low severity, defensive-coding items):
- `RecalDL` (detection view) filters `ph.Status = 1`; `UpDateRecalDL`'s inline duplicate of the same `main` subquery (inside its cursor loop) does NOT have this filter. Currently harmless because `@productionid` values fed into the cursor already came from `recalDL`, which is already Status=1-filtered — but the two copies of the query are not kept in sync, which is a maintenance trap.
- `RecalDL`'s `val` subquery joins `main` to `stocktrx` output values via `ProductionID` only (not `ProductionItemID`) — risks fan-out (duplicate/redundant flagged rows) for productions with more than one OUTPUT item. Not confirmed to cause wrong corrections (idempotent re-correction), just wasted cursor iterations.

### 4.7 `CalculateNewLoss` / `UpDateRecalLoss` — loss% formula correct, 2 process bugs found & fixed `[FIXED & DEPLOYED 2026-07-09, confirmed via sys.objects.modify_date + source inspection]`

Loss% formula (`@AccLoss = (@TotalLoss/@InputLoss)×100`) correctly sign-flips REMAINING throughout (`CASE WHEN pi.type='remaining' THEN -pi.kg ...`) — no CycleTime-style bug here.

**Bug 1 — unguarded `DROP TABLE pdloss`** in `CalculateNewLoss`: no existence check before dropping, so a cold session (or any run following a prior failed run that never got to create the table) throws `Cannot drop the table 'pdloss', because it does not exist.` and aborts the whole SP, killing `UpDateRecalLoss`'s cursor on that row. **Fix**: `IF OBJECT_ID('dbo.PDLoss','U') IS NOT NULL DROP TABLE PDLoss;`. Delivered as `fix_CalculateNewLoss.sql`, **deployed 2026-07-09 13:05:21** (confirmed: guard clause present in live `OBJECT_DEFINITION`).

**Bug 2 — `UpDateRecalLoss` infinite-loop risk**: the `else if (@rnd=2)` branch (comment: `"Recal production บางตัวไม่ผ่าน"` — "some productions won't pass") was clearly intended as a bail-out exit (same idea as the `@newCNT=0` branch right below it), but only zeroed `@newCNT` and never set `@rnd`:
```sql
else if (@rnd = 2)
    begin
        set @newCNT = 0        -- @rnd stays at 2 forever if row count plateaus here
    end
```
If the flagged-row count in `recalLoss` plateaus exactly when `@rnd` reaches 2 (i.e. some productions genuinely never converge — UPDATE doesn't match any lot, or they're excluded by a filter), `@rnd` never advances and `while(@rnd<4)` never becomes false → infinite loop. **Fix**: add `set @rnd = 6` to this branch, matching the sibling exit pattern. Delivered as `fix_UpDateRecalLoss.sql`, **deployed 2026-07-09 13:05:52** (confirmed: `set @rnd = 6` present in live `OBJECT_DEFINITION`).

**End-to-end verification (2026-07-09)**: after all 3 related SPs (`CalculateNewCost`, `Create_Recal_Cal`, `CalculateNewLoss`, `UpDateRecalLoss` — `UpDateRecalCost` itself needed no code change, its bug was entirely inherited from its dependencies) were confirmed deployed, `PR260626-0021`'s `stocktrx.cycletime` was re-checked and now correctly reads **38.013** (previously 1105.983) on both the PRODUCTION and SELL rows for that output lot — confirming the full `CalculateNewCost → Create_Recal_Cal → Recal (view) → UpDateRecalCost` pipeline works correctly end-to-end, not just in isolated SELECT-level testing.

**Also noted, not fixed (cosmetic, confirmed harmless)**: `@inputKGRevert`/`@remainKGRevert` are computed in `CalculateNewLoss` but never consumed anywhere downstream (dead code, wastes a query each call). The output column `'AvgInputloss'` actually selects `@inputloss` (= `@InputLoss`, a gross-kg-equivalent quantity, decimal(15,3)) — NOT `@AvgInputLoss` (the real average-loss percentage, decimal(10,5)); these are two different variables with confusingly similar names. Harmless in practice because `UpDateRecalLoss` only reads `[AccLoss%]` from the result table, never `[AvgInputLoss]`.

### 4.8 `RecalLoss` view performance — indexes deployed, SP-level tuning proposed but not yet deployed

Root cause of slowness (large data volume): `STOCKTRX` (1.13M rows) had no index supporting the `(productionid, produceitemid)` join or `(projectid, stockinout, remark)` filter used repeatedly across `RecalLoss`'s 3 separate stocktrx references; `ProduceHD` (34K rows) was a HEAP with zero indexes.

**Deployed** `[CONFIRMED via sys.indexes 2026-07-09]`:
```sql
CREATE NONCLUSTERED INDEX IX_stocktrx_production_join ON dbo.STOCKTRX (productionid, produceitemid)
    INCLUDE (StockINOUT, Remark, ProjectID, LossKG, Cost, InventoryCost, smcost, cycletime, StockTrxDate);
CREATE UNIQUE CLUSTERED INDEX PK_ProduceHD_ProductionID ON dbo.ProduceHD (PRODUCTIONID);
```

**Proposed, NOT yet deployed** (relevant specifically to `UpDateRecalCost`'s performance, since it calls `CalculateNewCost` once per flagged production inside a cursor):
- `CalculateNewCost` only filters `@productionid` via a final `HAVING` clause — the internal subqueries (`op`, `fc`, `sm`, `cy`) have no `WHERE productionid=@productionid`, so each per-production call risks aggregating across ALL productions before discarding all but one row. Recommendation: push the filter explicitly into each subquery rather than relying on the optimizer to infer it.
- `UpDateRecalCost` calls `EXEC Create_Recal_Cal` (a full-table rebuild) TWICE per loop iteration (once for `@oldCNT`, once for `@newCNT`) even though the second call's result is identical to what the next iteration's first call will recompute — carrying `@oldCNT = @newCNT` forward instead would cut full-table rebuilds roughly in half across the up-to-10-iteration loop.

### 4.9 Known open bug: `AvgSMCost` fan-out (unresolved, carried forward from earlier)

`CalculateNewCost`'s `sm` subquery (`AvgSMCost`, feeds into `SMCost`) joins `produceitem(type='submat')` to `stocktrx` without de-duplication — if a single submat `produceitemid` matches more than one `stockinout='production'`-tagged `stocktrx` row (e.g. the same packaging/consumable line drawn from 2 different stock lots), `SUM(st.cost*carton)` double-counts. Not yet reproduced against a concrete real example the way the CycleTime bug was (§4.3) — needs its own investigation before attempting a fix. Do not conflate with the CycleTime fix; they are separate subqueries with separate bugs.

---

## 5. TABLE RELATIONSHIP MAP — full FK/join reference

### 5.1 Domain diagram

```
SALES
  MF_Company ──< MF_SaleOrder ──< MF_SaleOrderDetail
       │                              │ ProductID
       └──< MF_Invoice ──< MF_InvoiceDetail ──> MF_Product
                  (SaleOrderID nullable back to SaleOrder)
  MF_CustomerType ──< MF_Company (CustomerTypeID)
  MF_Type / MF_INVType / MF_Currency ──< MF_Invoice

PURCHASING
  MF_PO ──< MF_PODetail ──> MF_ProductPO
     │           (ProductCode → inventory.product)
     ├──(CompanyID)──> MF_Supplier
     └──(DocNO)──> CostVatHD ──< CostVatDetail  ★real cost
                        (lot = ProductCode-DDMMYY)

PRODUCT MASTER
  MF_Product  ←(ProductCode_MildFoods)→  inventory.product
  (sales side, numeric ID)      (production/stock side, RM code)
                                    │ ProductID
                              BOMMain ──< BOMdetail (MAIN/ALTERNATIVE)
                          (one FG)   (RM/SM needed)

PRODUCTION & INVENTORY
  produceHD ──< produceitem
       │  (ProductionID)   (INPUT/OUTPUT/REMAINING/SUBMAT)
       ▼ (productionid, produceitemid)
  STOCKTRX  ─── ProjectID indicates source:
     ├─ 'BUY'         → MF_PO (StockDocNo = DocNO) → MF_Supplier (via CompanyID)
     ├─ 'production'  → produceHD/produceitem
     ├─ 'adjust stock'→ manual stock count correction
     ├─ 'SELL'        → MF_Invoice (Remark = DocNO)
     └─ 'MOVE'        → MoveHD (StockDocNo = MoveID)

  NeedToBuy ←── SaleOrderDetail + BOM + STOCKTRX + MF_PO
            ←── product.rop/lotsize (MAIN='ROP')

ACCOUNTING
  PR ──(ID)──< PRProduct (AccCode, WorkCode per line)
  PR.Docno ──(reference_doc)──> TotalAcct ←(UNION ALL)── ACCT_A..ACCT_G
  workcode (master) ──> P&L BT allocation (§10.3)
  S_HIS$ (salary) ──> P&L SB commission (§10.4)
  MF_Invoice.PaymentStatus='Completed' ──> AR (ARMain_R/ARDetail_R, ARMMMain_R/ARMMDetail_R — TVFs)
  MF_PO.PaymentStatus ──> AP (APMain_R/APDetail_R — TVFs)
  STOCKTRX(BUY) ──> UpdatePODetailQtyReceive(_Batch) ──> MF_PODetail.QTYReceived/ReceivingStatus
```

### 5.2 Full relationship table

| From | To | Join / notes |
|---|---|---|
| MF_Company.CompanyID | MF_SaleOrder / MF_Invoice | CompanyID |
| MF_Company.CustomerTypeID | MF_CustomerType.CustomerTypeID | 5 values: DOMESTIC_M/OVERSEA/ONLINE/DOMESTIC_F/WHOLESALE |
| MF_SaleOrder.SaleOrderID | MF_SaleOrderDetail.SaleOrderID | SaleOrderID |
| MF_SaleOrderDetail.ProductID | MF_Product.ProductID | int |
| MF_Invoice.InvoiceID | MF_InvoiceDetail.InvoiceID | filter ActiveFlag='A' when totaling |
| MF_InvoiceDetail.SaleOrderID | MF_SaleOrder.SaleOrderID | nullable — not every invoice has a source SaleOrder |
| MF_Product.ProductCode_MildFoods | inventory.product.ProductID | ⚠ cross-database, COLLATE THAI_CI_AS both sides |
| MF_PO.DocNO | MF_PODetail.POID | POID |
| MF_PODetail.ProductCode | inventory.product.ProductID | RM/SM code |
| MF_PO.DocNO | STOCKTRX.StockDocNo | ⚠ STOCKTRX.PODetailID is always 0, use this join instead |
| MF_PO.CompanyID | MF_Supplier.SupplierID | ⚠ misleading column name (CompanyID) but really SupplierID |
| MF_Supplier.SupplierID | STOCKTRX.SupplierID (ProjectID='BUY') | verified consistent |
| CostVatHD.PODocno | MF_PO.DocNO | PODocno |
| CostVatDetail.lot | STOCKTRX.rootlotno + ProductCode | pattern `{ProductCode}-{DDMMYY}` ★ real cost source |
| STOCKTRX.productionid | produceHD.PRODUCTIONID | ProductionID |
| STOCKTRX.produceitemid | produceitem.ProductionItemID | ProductionItemID |
| produceitem.rootproductid/rootlotno | STOCKTRX (INPUT side) | root product trace |
| BOMMain.ProductID | produceHD (output ProductID) | both point to same FG — BOM=plan, produceitem=actual |
| BOMMain.BOMID | BOMdetail.BOMid | BOMID |
| STOCKTRX.Remark (NOT InvoiceNo) | MF_Invoice.DocNO | ⚠ invoice link is via Remark only |
| STOCKTRX.StockDocNo (ProjectID='MOVE') | MoveHD.MoveID | no separate MoveDetail table |
| NeedToBuy.ProductID (MAIN='ROP') | product.rop / product.lotsize | TotalNeed = lotsize always when MAIN='ROP' |
| PRProduct.PRID | PR.ID | ⚠ uses PR.ID (hash), NOT PR.Docno |
| TotalAcct.reference_doc | PR.Docno | closes the expense loop: request→approval→actual payment |

**Top 3 join mistakes to avoid** (verified repeatedly across this document):
1. `STOCKTRX.PODetailID` is always 0 — use StockDocNo instead
2. Invoice lookup must use `Remark`, never `InvoiceNo`
3. Cross-database joins (MildFoods ↔ inventory) always need `COLLATE THAI_CI_AS`

---

## 6. PRODUCT MASTER & BOM

### 6.1 Product categories `[VERIFIED]`

`inventory.dbo.product.ProductCategory` has exactly 3 real values (confirmed — no others exist):

| Category | Meaning | Example |
|---|---|---|
| RM | Raw Material | DGS001 (mung bean), DGS010 (white sesame) |
| SM | Sub Material / packaging | bags, boxes used in packing |
| FG | Finished Goods | 177.004.001 (White Sesame), 177.004.003 (Mung Bean) |

### 6.2 BOMMain / BOMdetail `[VERIFIED]`

BOM defines what RM/SM one unit of FG needs — this is the SAME source of truth NeedToBuy uses (§8), NOT produceitem (which is actual production history).

| Table | Column | Meaning |
|---|---|---|
| BOMMain | BOMID | PK, links to BOMdetail.BOMid |
| BOMMain | ProductID | The FG this BOM describes |
| BOMMain | RootBOMID | References a parent BOM if nested |
| BOMdetail | BOMid | Links to BOMMain.BOMID |
| BOMdetail | ProductID | RM/SM required |
| BOMdetail | ProductCategory | RM/SM of this line |
| BOMdetail | Amount / Unit | Quantity needed per 1 unit of FG |
| BOMdetail | Type | MAIN or ALTERNATIVE (see below) |
| BOMdetail | Calculate | 'Y'/'N' — controls whether this line is counted in NeedToBuy calculation (see §8.2) |
| BOMdetail | RefBOMItemID | For ALTERNATIVE rows, points back to the BOMitemID of the paired MAIN row |
| BOMdetail | priority | Groups MAIN+ALTERNATIVE pairs together |

Verified example (177.004.001/177.004.003):

| FG | BOMID | RM | Amount/Unit | Type |
|---|---|---|---|---|
| 177.004.001 (White Sesame) | 7057 | DGS010 | 30 KG | MAIN |
| 177.004.003 (Mung Bean) | 7059 | DGS001 | 30 KG | MAIN |

This matches the root-product traceback in §2.2 exactly (177.004.001↔DGS010, 177.004.003↔DGS001, 30 KG/lot).

### 6.3 Main / Alternative substitution `[FULLY VERIFIED — resolved 2026-07-04]`

Real example (BOMid=6, FG=101.001.004):

| BOMitemID | RM | Type | Amount | RefBOMItemID | priority |
|---|---|---|---|---|---|
| 1289 | DSH020 | MAIN | 10 | 1289 (self) | 1 |
| 1290 | DSH020-C | ALTERNATIVE | 0 | 1289 (→MAIN) | 1 |

ALTERNATIVE rows always have `Amount=0` — the actual quantity is inherited from the MAIN row when substitution happens.

**`[RESOLVED]`** The long-open question of whether NeedToBuy checks ALTERNATIVE stock before flagging MAIN as short is answered. The VB.NET method `BOMTrxCls.CreateBOMTrx()` does NOT contain this logic itself — it merely calls `EXEC CheckBuyQtyR2`, a SQL stored procedure. `CheckBuyQtyR2` in turn calls `EXEC CreateBOMTrx` — a **separate, same-named SQL stored procedure** (not the VB.NET method) that does the real work:

```sql
CREATE PROCEDURE [dbo].[CheckBuyQtyR2] AS BEGIN
  exec soidetail2 'noshow'
  exec stockplacefull2 '2030-12-31'
  exec CreateBOMTrx        -- ★ this is a SQL SP, confusingly same name as the VB.NET method that calls it
  exec [CreateNowPO]
END
```

Inside the SQL `CreateBOMTrx` SP, for every BOM MAIN item the query self-joins BOMdetail via `RefBOMItemID` to pull in **both** the MAIN row and its ALTERNATIVE row(s) as separate lines in the `BOMTrx` working table — both sharing the same `MainRMID` (the anchor/MAIN ingredient's product code), but each with its own `ProductID` (MAIN row's ProductID = the main ingredient itself; ALTERNATIVE row's ProductID = the substitute ingredient). The MAIN row carries the real `TotalNeed`; the ALTERNATIVE row's `TotalNeed` is always 0.

Row ordering is `... type desc ...` — since `'MAIN' > 'ALTERNATIVE'` alphabetically, **the MAIN row is always processed first** within a `MainRMID` group. Back in the VB.NET consumption loop (`UpdateRefBOMTrx`), `Need_Remain` only resets when `MainRMID` changes — so:

1. MAIN row processed first: checks stock of the main ingredient, consumes what it can, leaves `Need_Remain` = whatever is still short.
2. Since the next row (ALTERNATIVE) has the **same** `MainRMID`, `Need_Remain` is **carried over, not reset**.
3. ALTERNATIVE row processed next: checks stock of the *substitute* ingredient (`GetRemainingStock(ObjBOMtrx.productID)` — now `productID` = the alternative's own code) to cover the leftover `Need_Remain`.
4. Only if both MAIN's and ALTERNATIVE's stock are exhausted does the shortfall propagate to the PO-reservation step and finally to NeedToBuy.

This confirms: **yes, ALTERNATIVE stock is checked, and it is checked in a genuine fallback order (MAIN first, ALTERNATIVE second) via a shared running-need variable** — not via any explicit "IF alternative available" branch, but as an emergent property of processing order + carried-forward `Need_Remain`.

There is also a secondary de-duplication step in the SQL `CreateBOMTrx` SP: it deletes MAIN-type BOMTrx rows whose `ProductID` also appears as an ALTERNATIVE product for the same `refDocno`, to avoid double-counting a product that plays both roles across different BOM lines within the same SaleOrder.

Minor data quality note: one stray BOMdetail.Type value `'ctn0915'` exists (looks like a product code accidentally entered in the Type column) — negligible, does not affect overall logic.

---

## 7. SALES FLOW

### 7.1 Documents

| Document | Tables | Role |
|---|---|---|
| Sale Order | MF_SaleOrder / MF_SaleOrderDetail | Advance customer order (before actual delivery) |
| Invoice | MF_Invoice / MF_InvoiceDetail | Delivery/billing document |
| Customer | MF_Company | Master customer data, linked to both via CompanyID |

### 7.2 Two order-intake patterns `[VERIFIED]`

1. **SaleOrder → Invoice**: customer orders in advance; SaleOrder issued first, Invoice issued at actual delivery. `MF_InvoiceDetail.SaleOrderID` points back to `MF_SaleOrder.SaleOrderID` (verified: INV2026-03-D0248 has SaleOrderID=1354).
2. **Direct Invoice** (no SaleOrder): typically routine/repeat orders that skip advance planning. `MF_InvoiceDetail.SaleOrderID = NULL`.

**Design note**: SaleOrder's real purpose is giving production advance warning to prepare RM — the entry point to NeedToBuy (§8). Skipping SaleOrder and going straight to Invoice means production gets no advance signal from this system.

### 7.3 Customer Type `[VERIFIED]`

`MF_CustomerType` — 5 values, linked via `MF_Company.CustomerTypeID`:

| CustomerTypeID | Code |
|---|---|
| 1 | DOMESTIC_M |
| 2 | OVERSEA |
| 3 | ONLINE |
| 4 | DOMESTIC_F |
| 5 | WHOLESALE |

**Verified gotcha (2026-07-03)**: currently **zero customers** have `CustomerTypeID='WHOLESALE'` (DOMESTIC_F has 109). If a user asks about "wholesale customers," clarify whether they mean this exact field or a behavioral/volume-based grouping — the field itself is unused right now.

### 7.4 MF_Type and MF_INVType — invoice classification `[VERIFIED master data; VERIFIED W/ OWNER pricing rule]`

`MF_Type` (trade route, 6 values):

| TypeID | TypeName |
|---|---|
| 1 | MF to CUS |
| 2 | MF to MM to CUS |
| 3 | MF/SP to CUS |
| 4 | MF/SP to MM to CUS |
| 5 | SP to MM to CUS |
| 6 | SP to CUS |

`MF_INVType` (market + unit basis, 4 values):

| INVTypeID | INVTypeName |
|---|---|
| 1 | EXPORT(CTN) |
| 2 | DOMESTIC(CTN) |
| 3 | DOMESTIC(KG) |
| 4 | EXPORT(KG) |

**Confirmed pricing field rule** (verified with system owner): for a given `MF_InvoiceDetail` row, the authoritative price fields depend on TypeID:
- TypeID 1, 3, 6 (route does NOT pass through MM) → use `UnitPrice` / `TotalPrice`
- TypeID 2, 4, 5 (route passes through MM) → use `ActualUnitPriceUSD` / `ActualTotalUSD`

**Critical clarification**: none of the price-field names reliably indicate currency (a field named "...USD" is not necessarily in US dollars). Currency must always be checked separately via `MF_Invoice.CurrencyID → MF_Currency`:

| CurrencyID | Code |
|---|---|
| 1 | THB |
| 2 | USD |
| 3 | RMB |

---

## 8. NEEDTOBUY — Production Planning & Purchasing

### 8.1 Conceptual flow `[VERIFIED]`

```
MF_SaleOrderDetail → MF_Product
   │ get FG ordered and quantity
   ▼
BOMMain / BOMdetail (join on FG ProductID)
   │ explode into RM/SM required (Amount per unit FG)
   ▼
Deduct against inventory balance (FG first, then RM/SM if needed)
   │ USED = quantity stock can cover / NEED_KG = remaining shortfall
   ▼
Check MF_PO / MF_PODetail — already ordered? Enough?
   ▼
Still short → row appears in NeedToBuy (FinalStatus='NOT OK')
Enough already → FinalStatus='OK' (no action needed)
```

### 8.2 NeedToBuy table columns `[VERIFIED]`

| Column | Meaning |
|---|---|
| DocNo / CustomerDocno | Source SaleOrder number (e.g. SOI2026-08-0181), or literal 'ROP' if triggered by reorder point instead of a customer order |
| MAIN | 'MAIN' = driven by a real SaleOrder; 'ROP' = auto-triggered by minimum reorder point |
| ProductID / MainRMID | The RM/SM/FG being evaluated |
| TotalNeed / NEED_KG | Total quantity needed vs actual shortfall after stock deduction |
| USED | Quantity covered by remaining stock |
| ACCNeed | Accumulated remaining shortfall |
| FinalStatus | 'OK' = stock/PO sufficient, 'NOT OK' = must purchase more |

### 8.3 ROP runs independent of SaleOrder-driven need `[VERIFIED]`

`product.rop` (reorder point) and `product.lotsize` (reorder quantity) directly set the values in `MAIN='ROP'` rows. Crucially, ROP checking runs as a SEPARATE pass even when every SaleOrder is already fully covered.

Real example (`CTN0501`, packaging box, rop=lotsize=700):

| ProductID | MAIN | DocNo | TotalNeed | USED | NEED_KG | FinalStatus |
|---|---|---|---|---|---|---|
| CTN0501 | MAIN | SOI2026-07-0206 | 10 | 10 | 0 | OK |
| CTN0501 | MAIN | SOI2026-06-0160 | 43 | 43 | 0 | OK |
| CTN0501 | MAIN | (4 more SOs) | 182 | 182 | 0 | OK all |
| CTN0501 | **ROP** | **ROP** | **700** | 0 | **700** | **NOT OK** |

Even though all 6 SaleOrders show `OK`, a separate ROP row still shows `NOT OK` because remaining stock fell at/below the 700-unit reorder point. When explaining "why does this need buying" to a user: `MAIN` = a specific customer order is short (more urgent), `ROP` = safety-stock replenishment (no order is actually pending). Both can occur simultaneously for the same product, independently.

### 8.4 The REAL calculation engine: VB.NET, not a single SQL SP `[VERIFIED from source code]`

Earlier investigation found only `GetNeedToBuy`, a trivial SP that reads a pre-computed table (`needToBuy_PurchasingPower`) — NOT the actual calculator. The real logic spans a SQL prep stage (`CheckBuyQtyR2`) plus a VB.NET consumption loop (`BOMTrxCls.UpdateRefBOMTrx()`):

```
STEP 0 — Prepare data (SQL, EXEC CheckBuyQtyR2)
┌─────────────────────────────────────────────────────────────┐
│  1. SOIdetail2       — refreshes SaleOrder shipping-status      │
│                        working table (filters to 'ON HAND')     │
│  2. stockplacefull2  — refreshes per-StockPlace stock working    │
│                        table                                     │
│  3. CreateBOMTrx (SQL)— explodes SaleOrder × BOM into "BOMTrx"    │
│                        working table (see Step 1 below)          │
│  4. CreateNowPO      — builds "NowPO": one row per PO detail      │
│                        line not yet fully received, with          │
│                        NotReceivedQty/NotReceivedRemaining         │
│                        pre-computed — this IS the source table    │
│                        behind GetRemainingPORMStock() in Step 3   │
└─────────────────────────────────────────────────────────────┘
   ▼
STEP 1 — Explode BOM (inside the SQL CreateBOMTrx SP)
┌─────────────────────────────────────────────────────────────┐
│  MF_SaleOrderDetail → MF_Product → BOMMain/BOMdetail              │
│  For each BOM MAIN item: self-join via RefBOMItemID to also        │
│  pull in its ALTERNATIVE row(s) as separate BOMTrx lines —         │
│  both share one MainRMID, but MAIN.ProductID = itself while        │
│  ALTERNATIVE.ProductID = the substitute. MAIN carries the real     │
│  TotalNeed; ALTERNATIVE's TotalNeed is always 0.                   │
│  Row order uses "...type desc..." → 'MAIN' sorts before            │
│  'ALTERNATIVE' alphabetically, so MAIN is always processed first.  │
└─────────────────────────────────────────────────────────────┘
   ▼
STEP 2 — Consumption loop (VB.NET UpdateRefBOMTrx)
┌─────────────────────────────────────────────────────────────┐
│  Loop by Docno (SaleOrder) → ProductCode (FG) → MainRMID          │
│                                                                    │
│  ├─ if type='ROP': check RM_Stock ≤ product.rop                   │
│  │     yes → OkToCheck=True (consider for purchase)               │
│  │     no  → delete this row (not yet at reorder point)           │
│  │                                                                 │
│  └─ if not ROP (real SaleOrder demand):                            │
│        Need_Remain ≤ 0        → RM_Used = 0                       │
│        Need_Remain ≥ RM_Stock → RM_Used = all of RM_Stock (short)  │
│        Need_Remain < RM_Stock → RM_Used = Need_Remain (stock left) │
│                                                                     │
│        ⚠⚠ Need_Remain only resets when MainRMID changes — so:      │
│        MAIN row processed first, consumes its own ingredient's      │
│        stock. If STILL short, the very next row (ALTERNATIVE,       │
│        same MainRMID) does NOT reset Need_Remain — it carries       │
│        the leftover forward and checks the SUBSTITUTE ingredient's  │
│        stock instead. This is the mechanism that answers §6.3's      │
│        question: yes, ALTERNATIVE stock is checked, as a genuine     │
│        fallback after MAIN is exhausted.                             │
│                                                                       │
│        ⚠⚠ Stock is ALSO consumed WATERFALL-STYLE across multiple      │
│        SaleOrders sharing the same RM — whichever SaleOrder is        │
│        processed first gets first claim; later ones get the rest.     │
│                                                                        │
│        Only BOMdetail.Calculate='Y' lines enter this calculation at    │
│        all — 'N' lines are skipped entirely.                          │
└─────────────────────────────────────────────────────────────┘
   ▼
STEP 3 — Reserve against existing PO
┌─────────────────────────────────────────────────────────────┐
│  If still short after Step 2 (Need_Remain > 0):                   │
│  GetRemainingPORMStock() reads the NowPO table (built in Step 0)   │
│  — any already-placed PO with remaining unreserved quantity?       │
│  Reserve against it (UpdatePORMReserved) until the PO is exhausted  │
│  or Need_Remain reaches 0.                                          │
└─────────────────────────────────────────────────────────────┘
   ▼
STEP 4 — Final result
┌─────────────────────────────────────────────────────────────┐
│  Whatever remains unmet after BOTH stock AND existing PO are        │
│  exhausted becomes the real NeedToBuy shortfall (FinalStatus=       │
│  'NOT OK'). MAIN=SOI-number rows mean a real customer order is      │
│  short (more urgent); MAIN='ROP' rows mean safety-stock             │
│  replenishment (no order pending) — both can appear simultaneously  │
│  for the same product, independently of each other.                 │
└─────────────────────────────────────────────────────────────┘
```

This fully confirms the conceptual flow in §8.1 and closes every previously-open question about this algorithm end-to-end: the BOM MAIN/ALTERNATIVE fallback mechanism, the waterfall stock consumption across SaleOrders, the `BOMdetail.Calculate` gate, and the exact source table (`NowPO`) behind the existing-PO reservation check.

---

## 9. INVENTORY VALUATION FORMULA `[VERIFIED]`

```
TotalCost = SUM( StockPerKG × [multiplier] × (Cost + ISNULL(SmCost,0)) × conversion_factor )
```

### 9.1 conversion_factor by category

| ProductCategory | conversion_factor | Why |
|---|---|---|
| RM | 1 | StockPerUnit already in KG, no conversion needed |
| FG (StockPerKG = 1) | MF_Product.NetWeight | Row counted in cartons (StockPerKG=1 is a flag, not real weight) — multiply by net weight/carton, e.g. 177.004.001 NetWeight=30 |
| FG (StockPerKG ≠ 1) | MF_Product.BagSize / 1000 | Row counted by actual weight already — use BagSize (grams) ÷ 1000 instead |
| SM | 1 | Packaging cost counted per piece, not weight |

**Gotcha (legacy code)**: an old commented-out line `--WHEN ProductCategory = 'FG' THEN pr.netweight` used NetWeight unconditionally for ALL FG rows — replaced by the current StockPerKG-conditional logic. If a stock valuation report looks wrong, check which query version is running.

**Dead code note**: the original query LEFT JOINs `MF_ProductPO` (alias `prpo`) but never uses it in SELECT/WHERE/GROUP BY — harmless leftover, safe to remove, doesn't affect results.

**Critical distinction — current value vs period movement**:

| Use case | Multiplier field | Extra WHERE | Meaning |
|---|---|---|---|
| Current point-in-time value | `StockLotBalanceUnit` | `StockLotBalanceUnit > 0 AND Active = 1` | True remaining balance of each lot line right now |
| Period accumulation (e.g. monthly report) | `StockPerUnit` | filter stocktrxdate range | Net movement (IN positive, OUT negative) during that period |

Verified real numbers using identical WHERE clauses but swapping the multiplier — the difference is large, especially for SM (~4-5×):

| Category | Using StockPerUnit (WRONG for "now") | Using StockLotBalanceUnit (CORRECT for "now") |
|---|---|---|
| RM | 17,420,822.97 | 19,655,418.97 |
| FG | 2,718,602.84 | 4,748,088.51 |
| SM | 605,049.81 | 2,693,142.26 |

If a user asks "what's my stock worth right now," using `StockPerUnit` understates the true value because it reflects the original lot-entry quantity, not what remains after usage.

**GROUP BY caveat**: the base formula grouped by `stocktrxdate` gives net movement PER DAY, not a running cumulative balance. For a true running total, sum across all days from the start, or filter the date range and drop the daily GROUP BY.

---

## 10. P&L PIPELINE (SSIS — `ImportExcelFile` package) `[VERIFIED from source XML, 43 components]`

> **`[VERIFIED W/ OWNER]` Company-wide financial health is checked via 3 DELIBERATELY SEPARATE systems, not one combined statement — do not try to union them:**
> 1. **Profitloss** (this section, §10) — P&L at **transaction level**, per invoice/product.
> 2. **IncomeStatement** (§11.3, `CreateBackUpStatement`) — P&L from an **annual/yearly view**.
> 3. **BalanceSheet** (§13, `GetBalanceSheet`) — **financial position** (assets/liabilities/equity), not a profit/loss view at all.
>
> Each answers a different question at a different granularity. `APIncomeStatement`'s `MAINCAT='09/AP'` numbering (§13.2) does NOT mean it's unioned with `Profitloss_R2`'s `01`-`08` sequence — that numbering is internal/coincidental to its own system, confirmed by the owner.

Connects 3 databases: `MF.INVENTORY`, `MF.MildFoods`, `MF.STAGING`. **Full-refresh every run** — truncates all `staging.pl.*` tables then rebuilds, no incremental logic, no mid-run checkpoint (a failure mid-pipeline leaves truncated tables empty until the next successful run).

### 10.1 Seven-stage architecture

```
1. Clean up table / Clean up FixCost
   truncate staging.pl.* (invoiceDetails, DiscountDetails, ActiveStocktrx,
   CNProfitloss, CMProfitloss, SalaryProfitloss, QAProfitloss, ProfitlossS1,
   Profitloss, ProfitlossSB, ProfitlossBT, Profitloss_R2, SummaryBusType,
   FixCostMF, usdprofitloss, invSellAmount, TSAmountKG)
   ▼
2. Prepare data (11 sub-flows pulling raw data into staging)
   GetActiveStocktrx, GetInvoiceDetails, GetDiscountDetails, GetCNProfitloss,
   GetCMProfitloss, GetQAProfitloss, GetSaleSalary, GetSummaryBusType,
   GetTSAmountKG, GetUSDProfitloss, GetInvSellAmount
   ▼
3. Create FixCost — UNION: FixCostAsset + FixCostDirector + FixCostMF + FixCostMM + FixCostProject
   ▼
4. Create ProfitLoss (Get Profitloss → Staging Profitloss)  ★ base table
   ▼
5. Create ProfitlossBT  (fixed-cost allocation by Business Type — §10.2)
6. Create ProfitLossS1  (invoice detail + currency + KG conversion side) [not fully documented yet]
7. Create ProfitLossSB  (Get ProfitlossSB01-04 → UNION → sales commission by CustType — §10.3)
   ▼
8. Create ProfitLoss_R2 ★ FINAL — UNION ALL of everything above
   Get Profitloss_R2 → Staging Profitloss_R2  (the table users/reports actually see)
```

### 10.2 Final output taxonomy (`Profitloss_R2`) `[VERIFIED from live query, corrected 2026-07-04]`

| MAINCAT | Category | Source |
|---|---|---|
| 01.Revenue(USD) | 01 Product Rev / 02 Other Rev | staging.pl.profitloss (currencycode='USD' × AvgFwdByYear) |
| 02.Expense(USD) | 01 CGS Exp / 02 LP Exp | staging.pl.profitloss (negative — expense) |
| 03.Revenue(THB) | 01 Product Rev / 02 Other Rev | staging.pl.profitloss (currencycode='THB', excludes invoices starting 'CN') |
| 04.Expense(THB) | 01 Discount, 02 RM Exp, 03 RM(LG) Exp, 04 DL Exp, 05 SM Exp, 06 TS Exp, 07 CN Exp, 08 CM Exp, 09 QA Exp, 10 SA Exp | staging.pl.profitloss — see business meanings below |

**Business meaning of each Category** (`[VERIFIED W/ OWNER]`, 2026-07-04):

| Category | Business meaning |
|---|---|
| RM Exp | Raw material cost |
| DL Exp | Direct labor cost |
| SM Exp | Sub-material cost |
| TS Exp | Transport cost |
| CN Exp | Cost from customer returns (Credit Note issued) |
| CM Exp | Sales commission |
| SA Exp | Salesman salary (from S_HIS$/SalesCode), allocated into each invoice by whichever salesperson is responsible for it |
| QA Exp | Quality-system cost (HACCP / GMP / NFI compliance) |

`[VERIFIED W/ OWNER]`: `SA Exp` (inside `04.Expense`) and the entire `07.SBExp(THB)` MAINCAT are **completely unrelated**, despite both touching CustType:
- **SA Exp** = salesman payroll cost, sourced from `S_HIS$`/`SalesCode`, distributed onto invoices by whichever salesperson is responsible for that invoice.
- **SBExp** = operating expense of a business/channel group (Online / Oversea / Domestic_F / Domestic_M) — has nothing to do with salesman salaries. Recorded via the `[workcode]` account (see §11 Account Subsystem).
| 06.BTExp(THB) | BT01(durian) / BT02(seafood) / BT03(dryfood) | staging.pl.profitlossBT — GROUP BY workcode (only these 3 groups, NOT BT00 general) |
| 07.SBExp(THB) | [SBType] | staging.pl.ProfitlossSB |
| 08.FIXExp(THB) | BT00(general) | staging.pl.profitlossBT — general fixed cost, split into its OWN MAINCAT, not merged into 06.BTExp |

`[CORRECTION 2026-07-04]`: Earlier version of this document (matching docx v2.6) had this wrong — it listed `05.BTExp`/`06.SBExp` and had no `08.FIXExp` at all. Live query against `staging.pl.Profitloss_R2` confirmed the real numbering is `06.BTExp`/`07.SBExp`/`08.FIXExp`, and critically that `BT00(general)` is NOT part of `06.BTExp` — it has its own separate MAINCAT (`08.FIXExp`). Always verify MAINCAT strings against live data before hardcoding them in a query rather than trusting a prior static read of the SSIS SQL, since the deployed table can drift from what the package XML shows.

### 10.2.1 Worked example: checking company-wide profit/loss `[VERIFIED 2026-07-04]`

Standard pattern for "is the company profitable this year" type questions:

```sql
SELECT MAINCAT, SUM(THBTotalValue) AS TotalTHB
FROM STAGING.pl.Profitloss_R2
WHERE yy = 2026
GROUP BY MAINCAT
```

Real result (2026 year-to-date, queried 2026-07-04):

| MAINCAT | THB |
|---|---|
| 01.Revenue(USD) | +20,041,366.20 |
| 02.Expense(USD) | -839,378.77 |
| 03.Revenue(THB) | +38,831,996.10 |
| 04.Expense(THB) | -37,710,618.99 |
| 06.BTExp(THB) | -806,819.59 |
| 07.SBExp(THB) | -2,967,339.74 |
| 08.FIXExp(THB) | -11,406,865.75 |
| **Net (profit)** | **+5,142,339.47** |

**Design note**: `yy` filters are Year-To-Date, not full calendar year, when the year in question is still in progress — always caveat this to the user rather than presenting it as a final annual figure. For a genuine full-year comparison, filter a fully-elapsed past year (e.g. `yy = 2025`) instead.



### 10.3 BT — fixed cost allocation by Business Type `[VERIFIED]`

Not allocated per-invoice directly — `Categoryratio` = an invoice's revenue share within its business-type group, multiplied against that group's FixCost/TotalExp. **Note**: in the final `Profitloss_R2` output, `BT00(general)` lands in MAINCAT `08.FIXExp(THB)`, not `06.BTExp(THB)` — see §10.2.

| Workcode_R | Condition | Meaning |
|---|---|---|
| BT00 (general) | `[work code]='BT00'` | Company-wide fixed cost, allocated by total company revenue |
| BT01 (durian) | Maincategory = 'Frozen Fruit' | Allocated only within frozen fruit/durian group |
| BT02 (seafood) | Maincategory = 'dried seafood' | Allocated only within dried seafood group |
| BT03 (dryfood) | Maincategory not in the above two | Allocated within other dry food group |

**Formula gotcha**: `Categoryratio` splits by currency — non-THB uses `TotalUSD/KG × kg × AvgRate` ÷ group revenue; THB uses `(PriceTHB/KG(B/F) + Other/KG(THB) − DiscountPerItem) × kg` ÷ group revenue. Getting the currency branch wrong corrupts the entire batch's cost allocation.

### 10.4 SB — sales commission by customer type `[VERIFIED]`

`GetSaleSalary` allocates sales salary/commission into invoices by `salesCode` mapped to `CustType`:

| salesCode | CustType (in formula) | Matches MF_CustomerType |
|---|---|---|
| SB01 | Domestic_F | DOMESTIC_F |
| SB02 | Oversea | OVERSEA |
| SB03 | Online | ONLINE |
| SB04 | Domestic_M | DOMESTIC_M |

`salesCode` is stored as an XML string delimited by '/' in `googleEmployee.salesCode` (one employee can sell to more than one CustType) — must XML `CROSS APPLY` to unpack before it maps cleanly to CustType.

`[OPEN QUESTION]`: this SB mapping has NO entry for WHOLESALE (5 CustTypes exist, only 4 salesCodes) — unclear whether WHOLESALE customers are excluded from SB commission entirely or bucketed elsewhere. Moot for now since 0 customers currently have that type (§7.3), but would matter if that changes.

### 10.5 ProfitlossS1 `[PARTIAL — high-level structure confirmed, not every branch verified]`

`Get ProfitlossS1` is a large, complex per-invoice-line query computing USD/KG cost & price metrics for margin analysis. Confirmed structure:
- Joins invoice detail, credit-note data (CN), exchange rates, and a per-product weight override table
- Computes columns: `USD/KG`, `Other/KG(USD)`, `TotalUSD/KG`, `CGS/KG(USD)`, `LP/KG(USD)`, `AfterTotalUSD/KG` (margin after cost)
- KG figure has a 3-way fallback: uses `invpr.NetWeight × AmountCarton` normally, but overrides with a per-invoice-line `productWeight` field when present and materially different (handles cases where the actual shipped weight differs from the nominal net weight)
- Contains several **hardcoded special-case branches** for specific product code prefixes (e.g. a branch specifically for codes starting `'115%'`) and a channel-specific override referenced in a code comment as "THE MALL case" — these look like one-off business exceptions bolted on over time, not general rules
- Currency/type logic mirrors §7.4: branches on `INVTypeID` and `CurrencyID` to decide which price field feeds the calculation

`[OPEN QUESTION]`: the full set of special-case branches (there appear to be several beyond the two spotted) has not been exhaustively enumerated — treat any per-product-line USD/KG margin figure as needing double-checking against this SP if the product code is unusual or the customer is "THE MALL"-like.

---

## 11. ACCOUNT SUBSYSTEM (ACCT / TotalAcct) `[VERIFIED from SP source]`

Four real stored procedures (`inventory.dbo`), forming a general-ledger import/consolidation system that connects directly to the P&L pipeline via `workcode` and salary data.

### 11.1 `ImportACCT` — Excel ETL

Maps a network share (`\\nas`), copies Excel files locally, then `OPENROWSET`/ACE OLEDB imports named sheets:

| Destination table(s) | Source file | Meaning |
|---|---|---|
| ACCT_A … ACCT_G | MF Account_R1.xlsm (sheets A_EXIM, B_KB_MF, C_KB584, D_KB577, E_U(T), F_U(S), G_KB993) | 7 bank account statements (EXIM, multiple KBank accounts) |
| ACC_MM, ACC_MM_B | MM_ACC_R4.xlsm | Sub-entity "MM" accounts |
| ACC_AP, ACCT_MM_AP | AP_Report.xlsx / _R1.xlsx | Accounts Payable |
| ACCT_AR, ACC_MM_AR | AR Report.xlsm / _R1.xlsx | Accounts Receivable |
| S_HIS$ | TIMERECORD_R3.xlsm | Employee salary history — ★ same table `GetSaleSalary` (§10.4) uses for commission |
| ACCCODE | ACC_Code_2022_backup.xlsx | Chart of accounts |
| dtsoi | Document Status_R4.xlsm | SOI (sale order) document status |
| NCRList | FM-NC-003.xlsx | Non-Conformance Report register (quality/BPI) |
| cash, FWD, PN | Cash Management_R1.xlsm | Cash, forward contracts, promissory notes |
| workcode | Total Work Code.xlsx | Work code master — ★ same table used in BT allocation (§10.3) |

**Design note**: end-of-SP logic resets ID generators for `MF_Invoice`/`MF_PO` to 0 every January 1st — explains why new-year document numbers appear to restart (not a bug).

**🔴 SECURITY GOTCHA**: this SP has network-share credentials hardcoded in plaintext directly inside `xp_cmdshell` (`net use` with a real user/password). Should be flagged to infra/security to move to a credential store. Actual credential value intentionally NOT reproduced in this document.

### 11.2 `Create_TotalAcct` / `Create_TotalAcct_check`

`Create_TotalAcct` unions ACCT_A through ACCT_G into one `TotalAcct` table, building a `join` key = `{acc code}-{month}-{year}-{prefix}`, prefix depending on work code type:

| Work code prefix | join key prefix | Meaning |
|---|---|---|
| SB% | `SELLBUDGET-` | Sales/commission budget (links to §10.4) |
| BT% | (none) | Business Type fixed cost (links to §10.3) |
| anything else non-blank | `NEWPROJECT-` | New project budget |

`Create_TotalAcct_check` is a parallel reconciliation version keeping extra detail (STATEMENT_DATE separate from DOC DATE, exchange rates, dep, doc_type, currency) — not the main reporting table. Related but not-yet-deeply-verified SPs in the same family: `CheckBalancedAcct`, `GetNOTBalancedTotalAcct`, `GetNOTCompletedTotalAcct(_NODoc)`.

**Gotcha**: `STATEMENT_DATE` (bank statement date) and `DOC DATE` (accounting document date) can differ — `Create_TotalAcct` uses DOC DATE as primary; if `TotalAcct` and `TotalAcct_check` numbers disagree, check this date mismatch first.

### 11.3 `CreateBackUpStatement` — daily snapshot

Snapshots the financial statement summary (from view/table `SumByAccCode`) into `backupstatement`/`backupstatementDetail`, deleting today's existing backup first (idempotent for multiple runs same day). Necessary because `TotalAcct`/`TotalAcct_check` are dropped and rebuilt every `ImportACCT` run (same full-refresh pattern as the P&L pipeline) — without this backup, prior-day numbers would be permanently lost.

### 11.4 PR / PRProduct — origin of Work Code and Acc Code `[VERIFIED]`

Purchase Request system, closing the loop from request to actual payment:

```
PR (Docno='PRQ...', RequestDate, RequestUser, Approve, Workcode)
   │  approval workflow document
   ▼  join: PRProduct.PRID = PR.ID  ⚠ NOT PR.Docno
PRProduct (AccCode, WorkCode, Amount, TotalAmount per line)
   │  each line has its own AccCode+WorkCode (doesn't always inherit from PR header)
   ▼  join: TotalAcct.reference_doc = PR.Docno
TotalAcct (actual bank payment record, from ACCT_A-G)
```

Verified real example: `PR.Docno='PRQ0925-0051'` (Workcode='BT00') → `PRProduct` (PRID='b23017bd'=PR.ID, WorkCode='BT00', Amount=3500). Other `TotalAcct` rows found with `reference_doc` pointing back to PRs, e.g. `PRQ0825-0053` (work code BT00, acc code ADM-QA-OP-002, detail "ค่าตรวจประจำปีฮาลาล" — annual halal inspection fee).

**Gotcha**: `PRProduct.PRID` joins to `PR.ID` (a hash like `b23017bd`), NOT `PR.Docno` (`PRQ0925-0051`) — same failure pattern as `PODetailID` elsewhere in this document; easy to join the wrong column since both "look like" identifiers.

**Design note**: `PR.Workcode` (header level) and `PRProduct.WorkCode` (line level) are different columns — they matched in the verified example but should not be assumed to always match, since one PR can have multiple lines with different work codes (same pattern as one PO having multiple RMs, §2.3).

### 11.5 AP / AR Subsystem `[VERIFIED]`

Accounts Payable and Accounts Receivable reporting is exposed via **table-valued functions** (`SQL_TABLE_VALUED_FUNCTION`, not base tables — confirmed via `sys.objects`), each parameterized by `@YearMonth`:

| Concept | Function names | Parameter |
|---|---|---|
| Accounts Payable (เจ้าหนี้ — what Mildfoods owes suppliers) | `APMain_R(@YearMonth)`, `APDetail_R(@YearMonth)` | `@YearMonth` in **`YYYY/MM`** format, e.g. `APMain_R('2026/06')` |
| Accounts Receivable (ลูกหนี้ — what customers owe Mildfoods) | `ARMain_R(@YearMonth)`, `ARDetail_R(@YearMonth)` | same format |
| Accounts Receivable — MM entity | `ARMMMain_R(@YearMonth)`, `ARMMDetail_R(@YearMonth)` | same format |

**`[VERIFIED — exact parameter format from source]`**: `APDetail_R`'s body parses the parameter as `@yyyy = SUBSTRING(@YearMonth,1,4)` and `@mm = SUBSTRING(@YearMonth,6,2)` — position 5 must be a separator character (any single character works positionally, but `/` is the convention used everywhere else in this codebase) with the 2-digit month at positions 6-7. **Always call with `'YYYY/MM'`** (e.g. `'2026/06'`). A no-separator string like `'202606'` happens to also work today only because SQL Server's date parser tolerates the resulting single-digit month string (`'2026-6-01'`) — this is incidental, not a supported format, and should not be relied upon.

Real output columns confirmed from `APMain_R('202606')`: `SupplierName`, `TotalPOAmount`, `TotalDeliveryAmount`, `TotalPaid`, `PP`, `AP`, `TotalDiff`, `Completed` — one row per supplier for that year-month, comparing what was ordered/delivered vs what's been paid, with a completion flag and a diff column to spot discrepancies. `APMain_R` is a thin aggregation wrapper over `APDetail_R` (groups by supplier, sums amounts, and derives `Completed`/`Not Completed` by comparing per-supplier completed-row-count vs total-row-count).

`APDetail_R` itself (the real per-document detail) pulls from `APTrans` (raw PAY/STOCKIN/NON-STOCKIN transaction log per PO docno) joined to `MF_PO`/`MF_PODetail`/`CostVatHD`/`MF_Currency`, computing: `%delivery(STOCKIN)` (received qty ÷ ordered carton qty), `%completed(MFIT)` (share of PO detail lines with `ReceivingStatus='Complete'`), `%paid` (paid ÷ delivered amount), a `SOURCE` column reflecting `CostVatHD.approve` status (`APPROVE`/`NOT APPROVE`/`NOT REQUEST`/`PO`), and a `STOCKIN-TYPE` flag distinguishing normal stock-in POs from `NON-STOCKIN` ones (e.g. service/expense POs with no physical goods receipt).

**Cross-reference**: this function family is the same one the `AP_Report.xlsx` VBA reporting tool (built earlier in this support engagement) queries by `YearMonth` parameter — confirms that spreadsheet tool is a thin front-end over these TVFs rather than its own separate query logic.

**`[VERIFIED W/ OWNER + root cause found]` How to get an accurate historical AP figure.** A real discrepancy was found: for June 2026, `BalanceSheet`'s `ACCOUNT PAYABLE` line showed 2.03M THB while a standalone call to `APMain_R('2026/06')` showed 6.99M THB for the same month (~3× difference). Root cause: `Create_APTrans`'s docno-inclusion subquery uses `GETDATE()` (today), not `@YearMonth`, to decide which documents qualify — only the per-row date filter respects the parameter. Since `APTrans` is a single shared full-refresh table, calling `APMain_R`/`APDetail_R` for a past month WITHOUT first re-running `Create_APTrans` for that same month reads whatever `APTrans` happens to currently contain (usually last rebuilt for the most recent month), giving an inflated or otherwise wrong figure for the month you actually asked about.

**Corrected rule — always do both, in this order, for any historical month**:
```sql
EXEC Create_APTrans '2026/06'          -- rebuild APTrans scoped to this month first
SELECT * FROM APMain_R('2026/06')      -- THEN query — now accurate for that month
```
Never call `APMain_R`/`APDetail_R` alone for a past month and trust the result — this is exactly the pattern `GetBalanceSheet`'s internal cursor loop follows correctly (§13.1.1), which is why capturing each month's value *during* that loop is reliable, while an independent standalone call afterward is not.

**Underlying transaction log tables** `[VERIFIED]`: both `_R` functions ultimately read from raw ledger tables, one row per credit/debit event tied to a PO or Invoice `Docno`:

| Table | Key columns | `Trans` values observed |
|---|---|---|
| `APTrans` | SupplierName, PurchasingType, Docno, TrxDate, Trans, Acct, Acct_Code, Detail, TotalValue, Currency, Credit, Debit, Balance, Result | `PAY`, `STOCKIN`, `NON-STOCKIN` |
| `ArTrans` | CustomerName, Docno, TrxDate, Trans, Acct, Acct_Code, Detail, TotalValue, Currency, CompanyID, Credit, Debit, Balance, Result | `DELIVERY`, `RECEIVED`, `CN`, `MFIT_SOI`, `ISSUE`/`ISSUED`, `PAYMENT.DONE` |

**Source of `APTrans`/`ArTrans`/`AR_MMTrans`** `[VERIFIED]`: these three tables are each rebuilt from scratch (DROP + rebuild, same full-refresh pattern as everywhere else in this system) by dedicated stored procedures — `Create_APTrans`, `Create_ARTrans`, `Create_AR_MMTrans`. All three:
- Take a single `@YearMonth` parameter and look up a cutoff date from a `lastdate` master table (keyed by `[year]`/`[month]`)
- Call `Create_DocnoCompany` first as a shared dependency
- `Create_APTrans` additionally calls `Create_APDATA` (populates a staging table `apdata`) and filters the final `APTrans` to transactions from `2023-01-01` onward through `@lastdate`

**⚠ Parameter format inconsistency between layers**: unlike the `_R` reporting functions above (which require a rigid position-based `'YYYY/MM'` with a separator at position 5), these `Create_*Trans` SPs parse the parameter with `LEFT(@YearMonth,4)` (year) and `RIGHT(@YearMonth,2)` (month) — separator-agnostic, since `RIGHT` always grabs the last 2 characters regardless of what's in between. The convention documented in the SP itself is `'YYYY-MM'` (dash), not `'YYYY/MM'` (slash) — **do not assume the two families of functions/procedures share one calling convention**; always check which one you're calling.

**`ARTrans` build logic** `[VERIFIED]`: computes a running `Balance` per `docno` using a window function (`SUM(...) OVER (PARTITION BY docno ORDER BY TRXDATE, ACCT ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`), summing `CN` as negative and most other transaction types as positive (excluding `DELIVERY`/`MFIT_SOI` rows from the running total). The resulting `Result` column translates the sign of that running balance into Thai status text: positive → `'ลูกค้าจ่ายเกิน'` (customer overpaid), negative → `'ลูกค้าค้างจ่าย'` (customer still owes), zero → `'Completed'`.

**`NON-STOCKIN` PO** `[VERIFIED]`: real examples found are all service/testing suppliers with no physical goods receipt — e.g. Intertek Testing Services, an IT/equipment vendor — i.e. POs raised for services, testing/certification fees, or other non-inventory expenses rather than raw material purchases. These still flow through the AP payment-tracking process but are excluded from `STOCKIN`-based delivery-percentage calculations.

**`CostVatHD.Approve`** `[PARTIAL]`: confirmed 3 real values — `'APPROVE'`, `'NOT APPROVE'`, `''` (blank) — used by `APDetail_R`'s `SOURCE` column to flag whether a PO's cost has gone through approval. Which SP/UI action sets this value has not been traced.

**`ARDetail_R` notable extra logic** `[PARTIAL]`: considerably more complex than `APDetail_R`. Independently cross-validates the TypeID pricing rule from §7.4 in its own `sellPercent` subquery (`TypeID=1 → TotalUSD`, `TypeID=2 → ActualTotalUSD`), links back to the originating SaleOrder via `SOIdocno`, and has a special foreign-exchange reconciliation branch that looks up `TotalAcct_check` (§11.2) where `[acc code] LIKE 'RE-RE-SA-001'` to get a "real" THB-received amount when currency ≠ THB. Filters out `CN%` and `%(D)%` doc numbers and only considers `deliverydate` from 2023 onward.

**Payment status triggers on the underlying documents** `[VERIFIED]`:

| Table.Column | Meaning | Affects |
|---|---|---|
| `MF_Invoice.PaymentStatus` | Set to `'Completed'` once the customer has paid in full (only observed values: `NULL` / `'Completed'`) | AR (feeds into `ARMain_R`/`ARDetail_R`) |
| `MF_PO.PaymentStatus` | Set once Mildfoods has paid the supplier in full | AP (feeds into `APMain_R`/`APDetail_R`) |

**Goods-receipt tracking on PO** `[VERIFIED]`:

| Column | Meaning |
|---|---|
| `MF_PODetail.QTYReceived` | Quantity actually received so far for this PO line — populated FROM `STOCKTRX` via stored procedures `UpdatePODetailQtyReceive` / `UpdatePODetailQtyReceive_Batch` (i.e. driven by real BUY-in stock movements, not manually entered) |
| `MF_PODetail.ReceivingStatus` | Changes to `'Completed'` once the full ordered quantity has been received |

This closes another loop in the purchasing flow: `MF_PO` → goods physically arrive → `STOCKTRX` (ProjectID='BUY') → `UpdatePODetailQtyReceive(_Batch)` writes back to `MF_PODetail.QTYReceived`/`ReceivingStatus` → once fully received AND paid, `MF_PO.PaymentStatus` flips and the PO is fully closed out on both the receiving and payment sides, visible via `APMain_R`'s `Completed`/`TotalDiff` columns.

### 11.6 MF_Invoice document number convention `[VERIFIED]`

Current invoices use 4 prefix families: `INV` (regular invoice), `CN` (credit note — customer return/adjustment), and `TF`/`TSX` (both tied specifically to `TypeID=5`, "SP to MM to CUS" trade route — see below). Within `INV`/`CN` numbers, a trailing letter code indicates market:

| Suffix | Meaning | Correlated `INVTypeID` |
|---|---|---|
| `EX` | Export (ต่างประเทศ) | 1 (EXPORT-CTN) or 4 (EXPORT-KG) |
| `D` | Domestic (ภายในประเทศ) | 2 (DOMESTIC-CTN) or 3 (DOMESTIC-KG) |

Verified by cross-checking real `DocNO` values against `INVTypeID` — e.g. `INV2018-03-EX008` has `INVTypeID=1` (export), `INV2017-03-D21` has `INVTypeID=3` (domestic).

**`[VERIFIED W/ OWNER]` `TF`/`TSX` prefixes are legitimate current invoice numbers, not legacy noise.** Both are consistently `TypeID=5` ("SP to MM to CUS"), `INVTypeID=1` (EXPORT-CTN), `CurrencyID=2` (USD) — e.g. `TF25-005` and `TSX22605-0043` (the latter dated as recently as June 2026, confirming this is an actively-used pattern, not dead data). `TF{YY}-{seq}` and `TSX{...}-{seq}` appear to be a dedicated numbering series for this specific trade route, parallel to the main `INV`/`CN` series used by other `TypeID`s.

**Gotcha**: Beyond `INV`/`CN`/`TF`/`TSX`, historical documents from the system's early years (2017–2020) show genuinely inconsistent one-off legacy prefixes (`QUO`, `ABC`, `D2-`, `D18-INV...`, `PI19-001`, and others, most with very low row counts and no ongoing use) — do not assume every `DocNO` in the table cleanly starts with one of the 4 main prefixes; filter or pattern-match defensively if scanning historical data across all years, but don't lump `TF`/`TSX` in with these — they're current and structurally consistent.

### 11.7 Real Exchange Rate System — `ExchByINV` / `AvgExchByINV` `[VERIFIED W/ OWNER, confirmed from view source]`

`SetExchangeRate` (the scheduled task that appeared stale in §13.3, last ran 2026-01-08) was NOT abandoned/broken — it was **deliberately superseded** by a more accurate per-invoice actual-rate system. `MF_Invoice.ExchangeRate` only holds a rough/nominal rate set at invoice creation time; the REAL rate used for reporting is derived from actual bank receipts.

**`ExchByINV`** (view) — real exchange rate per invoice:
```
For each invoice (non-THB currency):
  Find matching bank receipts in ACCT_A..ACCT_G where dep IN ('REVENUE','SALE')
  and reference_doc = this invoice's DocNO
  Actual rate = SUM(pay/receive [THB received]) / SUM(pay/receive2 [USD received])
```
Fallback order when no actual receipt exists yet: `TF%`/`TSX%` prefixed docs → invoice's own `ExchangeRate` or the annual `AvgExchByINV`; else if `PaymentStatus='Completed'` → the computed actual rate; else → invoice's nominal `ExchangeRate` or the `lastdate` table's rate for that month.

**`AvgExchByINV`** (view) — same real-rate concept but aggregated per **year** instead of per invoice, for expenses/transactions that aren't tied to any specific customer invoice (e.g. general operating costs in foreign currency). Computed from the same ACCT_A-G receipt data, restricted to completed export invoices (`docno LIKE 'TF%' OR docno LIKE '%EX%'`), plus a separate `AvgExchMFMM` figure specifically for MM-entity invoices (`TypeName LIKE '%MM%'`).

**Design principle**: the system prefers a bottom-up, transaction-verified exchange rate (actual THB received ÷ actual USD received, per invoice) over any static daily/nominal rate — the old `SetExchangeRate` task presumably set a single daily nominal rate, which this newer mechanism makes largely redundant for reporting purposes. The nominal rate likely still matters at invoice-creation time (before any payment has been received to compute an actual rate from).

---

## 12. AUTOEMAIL — Scheduled Automation (VB.NET) `[VERIFIED from source code]`

Despite the name, this VB.NET project (`.vbproj`) is a general scheduled-task orchestrator, not just an email sender. Launched via `runscript.bat` → `runscript.ps1` (Windows Task Scheduler). Entry point `Module1.vb`.

### 12.1 Task scheduling pattern

Every named task (e.g. `"UpdateNeedToBuy 00:00"`, `"BackupNeedToBuy"`, `"NotifyNeedToBuy"`) is idempotency-guarded via a `tracking` SQL table. Note: this VB.NET-side task list is not the complete picture — a separate, larger set of daily task types (`LastDate`, `SetExchangeRate`, `ImportAcct`, `UpdatePriceFromPODocno`, `CreateInventory`, `CreateAcct`, `CreateBalanceSheet`, `BackupStatement`) is tracked in the same `tracking` table by the Balance Sheet module's orchestration SPs — see §13.3. A third group — `UpdateRecalCost`, `UpdateRecalDL`, `UpdateRecalLoss`, `UpdateCostFromRootlotno` — is documented in §12.1.1.

```
TrackCls.IsDone(date, type) → SELECT COUNT(*) FROM tracking WHERE date=... AND type='...'
   count > 0 → already done today, skip
   count = 0 → run the task, then InsertTracking(type) to record it
```

**Design note**: actual notification channel is **LINE** (`LineMsgAPINotify` / `Objemail.GetGroupID`, posting into LINE groups like `PDQCLINE`, `STORAGE`, `T`) — NOT email, despite the project's name. Older code using LINE Notify's single-token API (`LineNotify`/`GetLineToken`) is commented out, superseded by the LINE Messaging API (group-based). This is a different system from the customer-facing Line OA support bot being separately developed.

### 12.1.1 `UpdateRecalCost` / `UpdateRecalDL` / `UpdateRecalLoss` / `UpdateCostFromRootlotno` — confirmed as daily scheduled tasks `[VERIFIED 2026-07-09]`

Previously treated (earlier in this KB, §4) as SPs run manually/on-demand for auditing and one-off correction. Querying the `tracking` table directly shows they are in fact **part of the same daily-scheduled automation** as `UpdateNeedToBuy`/`ImportAcct` — not previously listed in §12.1's example task list or §13.3's Balance Sheet task list, so this is a genuinely new finding, not a restatement:

```sql
SELECT type, MAX(CONVERT(date,date,103)) as last_run, COUNT(*) as run_count
FROM tracking WHERE type IN ('UpdateRecalCost','UpdateRecalDL','UpdateRecalLoss','UpdateCostFromRootlotno')
GROUP BY type
```

| Task type | Last run (as of query) | Historical run count |
|---|---|---|
| `UpdateRecalCost` | 2026-07-09 | 1,378 |
| `UpdateRecalDL` | 2026-07-09 | 1,371 |
| `UpdateRecalLoss` | 2026-07-09 | 1,052 |
| `UpdateCostFromRootlotno` | 2026-07-09 | 824 |

Run counts in the thousands confirm these have been running as daily routine automation for a long time (years), not occasional manual maintenance.

**Operational implication for the §4.3/§4.7 bug fixes**: because these SPs are invoked automatically every day via the same `tracking`-gated mechanism, once `CalculateNewCost`/`Create_Recal_Cal`/`CalculateNewLoss`/`UpDateRecalLoss` were fixed and deployed (§4.7/§4.8), the very next automatic daily run picks up the corrected logic without requiring any manual re-trigger — the manual `EXEC` runs done during this KB's audit session were for verification/immediate correction, not a substitute for or in addition to a separately-required scheduling step.

`[OPEN QUESTION]`: the exact schedule/time-of-day these 4 tasks are configured to run at (analogous to `UpdateNeedToBuy`'s multiple time slots `00:00`/`05:00`/`12:00`/`12:15`/`17:00`) has not been confirmed — only that they run once somewhere in each calendar day. Not yet checked whether related notification tasks seen in the same `tracking` type list (`NotifyCycleTimeExcessAvg`, `NotifyProductionLoss`) are downstream consumers of these Recal* runs.

### 12.2 NeedToBuy real calculation engine — see §8.4 for full algorithm detail.

Summary: `BOMTrxCls.UpdateRefBOMTrx()` explodes SaleOrder×BOM into a working table, waterfall-consumes stock across SaleOrders sharing an RM, then reserves against existing PO before concluding a real shortfall. Backed up daily via `StockCls.CreateBackupNeedToBuy()`. Notified via LINE split by ProductCategory (RM/SM) to groups PDQCLINE and STORAGE.

### 12.3 🔴 Critical security gotcha

`ConStr.vb` has a SQL connection string with a hardcoded, plaintext `sa` (System Administrator — highest privilege SQL Server account) credential directly in VB.NET source code. This is worse than the `xp_cmdshell` credential found in `ImportACCT` (§11.1) because (a) `sa` is the highest possible database privilege, and (b) it lives in application source code, which is more likely to end up in version control or be shared inadvertently. Should be escalated to infra/security urgently — least-privilege account + externalized secret (not hardcoded) recommended. Actual password value intentionally NOT reproduced here.

### 12.4 Email.vb — Report/Notification Text Generator `[PARTIAL — function catalog confirmed, bodies not individually verified]`

Despite its name, `Email.vb` (3,792 lines, the largest file in the project) is a library of `GetXXX()` functions that each pull a `DataTable` from SQL and format it into human-readable text for LINE notifications (not literal email) — dozens of report types. Confirmed function inventory (names only; bodies not individually verified) covers areas already documented elsewhere in this KB plus several not previously known:

| Area | Example functions |
|---|---|
| NeedToBuy notifications | `GetNotifyNeedToBuyByCategory` (used in §12.2/13.1) |
| Stock health | `GetDeathStock`, `GetDefaultRisk`, `GetStockplace`, `GetProductionLoss`, `GetProductionLossByType` |
| Sales planning vs actual | `GetSalePlanAcutalGP`, `GetSalePlanActual`, `GetSaleYearly`, `GetSaleTarget`, `GetSaleTargetExpense`, `GetSBPlanActual` |
| Budget/expense tracking | `GetExpenseBudget`, `GetExpenseBudgetByAccCode`, `GetOverViewBudget_GPASB`, `GetOverViewBudget_FixCost`, `GetOverViewBudget_AdjustStock`, `GetSummaryOverviewBudget` |
| Production metrics | `GetCycleTimeExcessAvg` |
| Cash / FWD | `GetCurentFWD` (forward contracts — connects to §11 Account Subsystem `cash, FWD, PN` tables) |
| Account reconciliation | `GetNotCompletedTotalAcct`, `GetNotBalancedTotalAcct`, `GetCheckBalancedTotalAcct` (LINE-notification wrappers around the SPs mentioned in §11.2) |
| AP/AR | `GetNotifyAPDiff`, `GetNotifyAPPayment`, `GetNotifyARPayment` |
| Quality / approvals | `GetQCNotify`, `GetUnApproveListNotify` |
| Data-quality alerts | `GetNoActualPriceNotify`, `GetNoExchShipmentNoNotify`, `GetNeedUpdateStockTrxPODocno` |
| Misc | `GetLINENotifyCompanyHoliday`, `GetFoodlandNotify`, `GetJobAlert` |

`[OPEN QUESTION]`: none of these individual function bodies have been read — only the name list. If a user asks about the exact logic behind any specific report (e.g. "how is `GetDefaultRisk` calculated"), this still needs to be opened and read individually.

### 12.5 `UploadPOAuto` — Google Drive Attachment Upload `[VERIFIED from source code, googleCls.vb]`

The actual function name is **`UploadPOAuto()`** (class `googleCls`) — despite the name suggesting it's PO-only, it's a general document-attachment uploader covering **8 document types**, not just Purchase Orders.

```
UploadPOAuto():
  CreateService()                     — OAuth to Google Drive
  SearchFolder("SOI_PO")               — target folder (hardcoded ID, name lookup is dead/commented-out code)
  dt = NeedUpdateDT()                  — SELECT * FROM needUploadPO (the upload queue)
  Loop while queue has rows:
    Translate network path → local path:
      "\\mf\"              → "D:\"
      "\\mfit.dyndns.biz\" → "E:\MindF\D\"
      (for type='CostVat', converted back to "\\nas\" instead)
    UploadFile(path) → Google Drive, returns "https://drive.google.com/file/d/{fileID}/view"
    UpdatePO(docno, url, ID, subID, doctype, filename, path) → writes URL back to the right table
    If doctype='CostVat': delete the local file after successful upload
    On failure: UpdateNullAttachFile() clears the pending attachment reference
```

**Per-document-type destination table** (branches inside `UpdatePO`):

| DocType | Destination table | Linked to |
|---|---|---|
| `SOIPI` | `img_soi` | MF_SaleOrder (via SaleOrderID) — base attachment, source: `MF_SaleOrder.AttachFileName`/`AttachFilePath` |
| `PRPO` | `img_ProductPO` | MF_ProductPO |
| `SOIPO` | `img_soipo` | MF_SaleOrder — source: `AttachFileNamePO`/`AttachFilePathPO` |
| `SOIPIPO` | `img_soipipo` | MF_SaleOrder — source: `AttachFileNamePIPO`/`AttachFilePathPIPO` (+ `PIPOReceivingDate`) |
| `INVOICE` | `img_inv` | MF_Invoice |
| `STX` | `stxapprove` | STOCKTRX (writes `url` + sets `notify='1'`) |
| `CostVat` | `CostVatHD` | writes directly onto `CostVatHD.url`/`uploadFile`, not a separate img_ table |
| (default/else) | `img_PO` | MF_PO |

**`[VERIFIED W/ OWNER]` Business purpose of `SOIPO`/`SOIPIPO`**: whenever a customer issues their own purchase order to buy goods, that customer PO document (a scan/photo, e.g. `.jpg`/`.pdf`) is uploaded as proof-of-order evidence, attached directly on the corresponding `MF_SaleOrder` record. `MF_SaleOrder` has 3 parallel attachment slots for this:

| Column pair | DocType | Purpose |
|---|---|---|
| `AttachFileName`/`AttachFilePath` | SOIPI | Base sale-order-level attachment |
| `AttachFileNamePO`/`AttachFilePathPO` | SOIPO | Customer's own PO document specifically |
| `AttachFileNamePIPO`/`AttachFilePathPIPO` (+ `PIPOReceivingDate`) | SOIPIPO | Combined Proforma Invoice + PO, with a receiving-date confirmation field |

Verified real examples: `SOI2026-08-0181` → `PO6906078.jpg`; `SOI2026-08-0162` → `PO6905319 กุ้งขาวใหญ่016 มายฟู้ดส์.pdf` — actual scanned customer PO files stored at `D:\MildFoodsDatabase\Web\MFUpload\...` before being picked up by the `needUploadPO` queue and pushed to Google Drive.

Three **cleanup subs** (`DeleteUnUsedSOI`/`DeleteUnUsedPO`/`DeleteUnUsedINV`) remove orphaned image records when the parent SaleOrder/PO/Invoice's `DocNO` changed or the parent record became inactive (`ActiveFlag<>'A'`) — called automatically at the start of every `UpdatePO` call.

**🔴 Security gotcha (3rd hardcoded credential found in this codebase)**: `CreateService()` has a Google OAuth **Client ID and Client Secret hardcoded in plaintext** directly in `googleCls.vb`. Alongside the `sa` SQL credential in `ConStr.vb` (§12.3) and the network-share credential in `ImportACCT` (§11.1), this is the third instance of a hardcoded secret in this codebase — worth a single consolidated remediation pass rather than three separate fixes. (OAuth "installed application" client secrets are somewhat less sensitive than a database admin password since they still require an interactive user consent step to do anything, but hardcoding is still poor practice and the value is not reproduced here.)

**Gotcha**: `UpdatePO`/`UpdateNullAttachFile` build every SQL statement via raw string concatenation (e.g. `"...where docno = '" & Docno & "'"`) — classic SQL-injection-shaped code, though in practice `Docno`/`ID`/`subID` originate from the internal `needUploadPO` queue (populated by trusted internal processes) rather than raw user input, so real-world exploitability is low. Still worth flagging alongside the other credential/security gotchas in this project.

---

## 13. BALANCE SHEET MODULE `[VERIFIED — major discovery 2026-07-04]`

An entire financial-statement subsystem parallel to the P&L Pipeline (§10) that had not been touched in any earlier pass of this document. Discovered while investigating an unrelated `APIncomeStatement` object. **Owner-confirmed division of responsibility**: Income Statement coverage = `CreateBackUpStatement` (§11.3); Balance Sheet coverage = `GetBalanceSheet` (this section).

### 13.1 Core tables and objects

| Object | Type | Purpose |
|---|---|---|
| `BalanceSheet` | table | Monthly snapshot of balance-sheet line items — one row per account `Category` per `mm`/`yy`, tagged `MAINCAT` = **`ASSET` / `LIABILITY` / `EQUITY`** (all 3 confirmed — see §13.1.1). Carries raw `VALUE` plus scaled variants `Value_R1` (÷1000) and `Value_R2`. |
| `BalanceSheet_AP` | table | Working table populated by a monthly cursor loop inside `GetBalanceSheet` — holds `PPValue`/`APValue` per month, later folded back into `BalanceSheet` as the `'ACCOUNT PAYABLE'` and `'PREPAID EXPENSE'` lines (see §13.1.1) |
| `BalanceSheetIncome` | table | Same shape as `BalanceSheet` but is the working table `APIncomeStatement` (§13.2) reads from to compute period-over-period movement |
| `APIncomeStatement` | view | Computes the **change in Accounts Payable balance** between the start and end of a period (`bsEnd.Value - bsBeg.Value`, using `ROW_NUMBER()` over `[BUDGET CODE]` ordered by year/month to find "previous period") — i.e. this is the AP *movement* line that would feed a cash-flow or expanded income statement, not a simple balance |
| `ARBalanceSheet`, `ARMMBalanceSheet` | table-valued functions | AR-side balance figures — `ARBalanceSheet()` supplies the Trade Receivable (`AR`) and Deferred Revenue (`-DFR`) lines directly into `GetBalanceSheet`'s union (confirmed, §13.1.1); parameter convention not confirmed |
| `GetBalanceSheet` | stored procedure | ★ Builds the entire `BalanceSheet` table from scratch every run — see §13.1.1 |
| `GetBalanceSheetIncome` (+ `_backup`) | stored procedure | Presumably populates `BalanceSheetIncome` — not opened in detail |
| `RefreshBalanceSheet`, `RefreshIncomeStatement` | stored procedures | Orchestration/reset SPs — see §13.3 |

Real `BalanceSheet` rows confirmed: e.g. `Category='Trade Receivable(MM)'`, `MAINCAT='ASSET'`, monthly values from 2024 through 2026 — confirming this is a genuine time-series balance sheet, not a one-off report.

### 13.1.1 `GetBalanceSheet` — full architecture `[VERIFIED — opened full SP body]`

This single stored procedure is the entire Balance Sheet construction engine. It drops and rebuilds `BalanceSheet` from scratch (full-refresh, same pattern as everywhere else) by `UNION`-ing many independent ledger sources, then computes `EQUITY` as a derived plug figure:

```
GetBalanceSheet:
  1. EXEC Create_Trade_Receivabale   (⚠ typo in real object name — missing a 'b')
  2. EXEC Create_Trade_ReceivableMM
  ▼
  3. Build BalanceSheet from UNION ALL of:
     ├─ Inventory value          ← inventorybylastdate
     ├─ Cash(MF)                 ← ACCT where CategoryTmp in ('ACCT-A'..'ACCT-G')
     ├─ Cash(MM)                 ← ACCT where CategoryTmp = 'ACCT-MM'
     ├─ Trade Receivable(MM) /
     │  Deferred Revenue(MM)     ← pre-2023: ACC_MM_AR (split by USD sign)
     │                             2023+: Trade_ReceivableMM table
     ├─ Trade Receivable (MF)    ← 2023+: trade_receivable table;
     │  / Deferred Revenue (MF)    2023+: ARBalanceSheet() TVF (AR / -DFR columns)
     ├─ Account Payable(MM)      ← acct_mm_ap (USD × exchange rate)
     ├─ Prepaid Expense(MM)      ← acc_mm where dep='LIABILITY', split by payment sign
     ├─ Per-acc-code running     ← ACCT_A..ACCT_G / ACCT_MM / ACCT_MM_B, running
        balances                   SUM() OVER (...) cumulative by [acc code]
     └─ (older AP-%/APM-% lines from legacy acc_ap/acct_mm_ap logic, pre-2023)
  ▼
  4. Hardcoded Category → sort-order ('no', 1-27+) and Category → MAINCAT
     (ASSET/LIABILITY) mapping — embedded directly as a big CASE expression
     in the SP itself, not read from a lookup table
  ▼
  5. "UPDATE PP AND AP VALUE" — a CURSOR loop over every month since 2023:
       for each month: EXEC Create_APData → EXEC Create_APTrans(@YearMonth)
       → SELECT SUM(PP), SUM(AP) FROM APMain_R(@YearMonth)
       → store into BalanceSheet_AP
     Then DELETE + re-INSERT the 'ACCOUNT PAYABLE' and 'PREPAID EXPENSE'
     lines in BalanceSheet from this monthly cursor result.
     ⚠ This re-runs the entire AP pipeline (§11.5) once PER MONTH via a
     cursor, calling Create_APTrans(@YearMonth) immediately before
     APMain_R(@YearMonth) each time — this sequence IS the correct way to
     get an accurate per-month AP figure (see §11.5's corrected rule).
     A standalone call to APMain_R for a past month WITHOUT first re-running
     Create_APTrans for that month will read stale/wrong data, because
     APTrans is a single shared full-refresh table last rebuilt for
     whatever month a prior call scoped it to.
  ▼
  6. Final SELECT: CROSS JOINs every (Category, MAINCAT) combination against
     every month in `lastdate` (to fill gaps with zeros), then UNIONs one
     more computed row per month:
       MAINCAT='EQUITY', Value_R2 = SUM(ASSET Value_R2) − SUM(LIABILITY Value_R2)
     — i.e. EQUITY is not sourced from any ledger; it's the balancing
     plug figure (Assets − Liabilities), the standard accounting identity.
```

This confirms `MAINCAT` in `BalanceSheet` has exactly 3 real values: `ASSET`, `LIABILITY`, `EQUITY` — resolving the earlier uncertainty. The `'Run'` column (1=ASSET, 2=LIABILITY, 3=EQUITY, 99=other) controls display order in whatever report/UI consumes this table.

**Gotcha**: the dependency `Create_Trade_Receivabale` is spelled with a typo in the actual database object (missing the second 'b' in "Receivable") — searching for `Create_Trade_Receivable` (correct spelling) will NOT find it.

The view joins `BalanceSheetIncome` to `ACCCODE` (chart of accounts, §11.4) and `lastdate`, computes the balance difference between consecutive periods for each `[BUDGET CODE]`, and outputs it tagged `MAINCAT='09/AP'`, `running=9`. This numbering is internal to the Balance Sheet / Income Statement systems' own display ordering — **confirmed by the owner it is NOT meant to extend `Profitloss_R2`'s `01`-`08` MAINCAT sequence** (§10 intro); the three financial-health systems (Profitloss, IncomeStatement, BalanceSheet) are deliberately kept separate rather than unioned.

**Special-case override found**: for `yy >= 2022` AND `category LIKE 'CGS-LP-DM%'`, the movement is forced to `0` instead of the calculated balance difference — a code comment marks this "logic ใหม่" (new logic), indicating a business process changed starting 2022 for this specific account category. The nature of that change is not documented in the code itself.

Also distinguishes `'AP-MF'` (main company) vs `'AP-MM'` (MM subsidiary) by whether the account category contains `'-MM-'`.

### 13.3 Scheduled task list — larger than previously documented `[VERIFIED]`

Both `RefreshBalanceSheet` and `RefreshIncomeStatement` work by deleting today's `tracking` rows for a specific list of task `type` values — which, per the idempotency mechanism documented in §12.1, forces those scheduled tasks to re-run instead of being skipped as "already done today". The task list revealed is **larger than what was documented in §12.1's AutoEmail walkthrough**:

| Task type | Previously documented? |
|---|---|
| `LastDate` | No — sets the `lastdate` master table used throughout §11 and §13 |
| `SetExchangeRate` | No |
| `ImportAcct` | Yes (§11.1, `ImportACCT`) |
| `UpdatePriceFromPODocno` | No |
| `CreateInventory` | No |
| `CreateAcct` | No — distinct from `ImportAcct`; likely builds `TotalAcct` (§11.2) |
| `CreateBalanceSheet` | No (this section) |
| `BackupStatement` | Yes (§11.3, `CreateBackUpStatement`) |

A commented-out reference to `EXEC msdb.dbo.sp_start_job 'Balance Sheet'` inside `RefreshIncomeStatement` additionally confirms there is a **SQL Server Agent Job** named `'Balance Sheet'` that runs this whole pipeline on a schedule (separate from the Windows Task Scheduler + `tracking` table mechanism used by the AutoEmail VB.NET project in §12).

`[OPEN QUESTION]`: none of `GetBalanceSheet`, `GetBalanceSheetIncome`, `BalanceSheet_AP`, `ARBalanceSheet`, `ARMMBalanceSheet`, `CreateInventory`, `CreateAcct`, `SetExchangeRate`, or `UpdatePriceFromPODocno` have been opened yet — only their existence and role in the orchestration chain are confirmed. Whether `Profitloss_R2` (§10) and `BalanceSheet`/`BalanceSheetIncome` (this section) are ever unioned into one combined financial statement is unconfirmed.

---

## 14. MASTER LESSONS LEARNED / GOTCHAS (flat reference list)

- Invoice lookup in STOCKTRX must join via `Remark`, never `InvoiceNo`.
- `ActiveFlag='R'` in MF_InvoiceDetail means excluded/revised — filter `ActiveFlag='A'` only when totaling.
- `STOCKTRX.PODetailID` is always 0 — link to PO via `StockDocNo = MF_PO.DocNO` then match ProductCode manually.
- Prices in `MF_PODetail` are not real cost (usually 0) — real cost is `CostVatDetail.RMCost/InventoryCost`.
- `CostVatDetail.lot` uses pattern `{ProductCode}-{DDMMYY}` as a key, not a numeric FK.
- ProductID has (at least) 2 parallel standards: numeric ID in `MF_Product` (finished goods) vs RM code (e.g. DGS001) in `inventory.dbo.product` (raw material) — bridged via `ProductCode_MildFoods`.
- `rootlotno` has 3 origins: BUY (purchased in), production (output of manufacturing), adjust stock (inventory count correction) — indicated by `ProjectID`.
- Database `inventory` (lowercase) is separate from `MildFoods` — always prefix database name correctly for cross-database queries.
- `EXEC`/`CREATE OR ALTER`/other DDL cannot run through the read-only `execute_read_query` MCP tool (rollback-wrapped, SELECT-style only) — but DOES work through `execute_write_query` once `MSSQL_ENABLE_WRITES=true` is set on the connector (confirmed 2026-07-20, §17.16 — deployed a `CREATE OR ALTER PROCEDURE` directly, no SSMS needed). When writes are disabled, fall back to inlining the SP body as SELECT to debug/verify, or deliver a `.sql` file for SSMS.
- `product.ProductCategory` has exactly 3 values: RM / SM / FG.
- BOM (BOMMain/BOMdetail) = "what SHOULD be used"; produceitem = "what WAS actually used" — mismatches signal real substitution or over/under-use, not necessarily a data error.
- `MF_InvoiceDetail.SaleOrderID` can be null — not every invoice has a source SaleOrder.
- NeedToBuy has 2 sources distinguished by `MAIN` column: 'ROP' = auto-triggered by reorder point (no SaleOrder link), 'MAIN' = tied to a real SaleOrder (DocNo starts with SOI).
- `ProjectID='MOVE'` has no separate `MoveDetail` table — detail lives in STOCKTRX itself as MOVEOUT/MOVEIN pairs joined via `StockDocNo = MoveHD.MoveID`.
- ROP (`MAIN='ROP'`) runs independently of SaleOrder-driven need — even if every SaleOrder shows `FinalStatus='OK'`, a separate ROP row can still appear if remaining stock hits `product.rop`; quantity always equals `product.lotsize`.
- `MF_CustomerType` has 5 values (DOMESTIC_M, OVERSEA, ONLINE, DOMESTIC_F, WHOLESALE); no Thai label field — must interpret from code.
- Stock valuation for FG must branch on `StockPerKG`: =1 uses NetWeight, ≠1 uses BagSize/1000. Using NetWeight unconditionally (old logic) gives wrong values.
- Stock valuation GROUP BY date = net movement per day, not cumulative balance — must sum across days for a running total.
- Valuation multiplier rule: current-point-in-time value uses `StockLotBalanceUnit` (with `StockLotBalanceUnit>0 AND Active=1`); period-accumulated value uses `StockPerUnit`. Swapping these gives wrong numbers — SM differs by ~4-5×.
- P&L Pipeline is full-refresh every run (truncates all `staging.pl.*` first) — no mid-run checkpoint; a failed run leaves truncated tables empty until the next successful run.
- BT (Business Type fixed-cost allocation) has 4 groups: BT00 (general, company-wide), BT01 (durian), BT02 (seafood), BT03 (dryfood) — `Categoryratio` formula must branch by currency (USD uses AvgRate multiplier, THB doesn't) before dividing by group revenue.
- SB (sales commission by CustType) maps salesCode SB01-04 → Domestic_F/Oversea/Online/Domestic_M matching MF_CustomerType — but has no WHOLESALE mapping; unconfirmed how (or if) wholesale commission is handled.
- Currently zero customers have `CustomerTypeID='WHOLESALE'` (verified 2026-07-03) — if a user mentions "wholesale customers," clarify whether they mean this field or a behavior/volume-based grouping.
- `ImportACCT` is a full-refresh Excel ETL (bank statements, AP/AR, salary, cash, workcode) — depends on `CreateBackUpStatement` for daily snapshots, or prior-day data is permanently lost.
- `TotalAcct.join` key ties work codes (SB=SELLBUDGET, BT=no prefix, other=NEWPROJECT) directly to the P&L Pipeline — `workcode` and `S_HIS$` (salary) are the SAME tables shared by both the Account Subsystem and the P&L Pipeline.
- `STATEMENT_DATE` (bank statement date) vs `DOC DATE` (accounting document date) can differ — check this first if `TotalAcct` and `TotalAcct_check` numbers disagree.
- `PRProduct.PRID` joins to `PR.ID` (hash), not `PR.Docno` (PRQ...) — easy to join the wrong column; `TotalAcct.reference_doc = PR.Docno` closes the PR→PRProduct→TotalAcct loop.
- 🔴 **Critical**: joining `produceitem` INPUT to OUTPUT directly on `ProductionID` without `DISTINCT` causes fan-out when a production run has the same RM as INPUT across multiple lot rows (seen up to 169 rows in one production) — inflates `SUM(kg)` by tens to hundreds of times. Always `SELECT DISTINCT ProductionID` on the many-side before joining (see §2.5 for correct pattern). 476 production runs in the system are at risk of this.
- `MF_PO.CompanyID` is a misleading column name — it actually references `MF_Supplier.SupplierID`, NOT `MF_Company` (a completely different table used on the sales side).
- `BOMdetail.Type` has MAIN/ALTERNATIVE — substitutable RM/SM, linked via `RefBOMItemID` (ALTERNATIVE points back to the paired MAIN's BOMitemID). ALTERNATIVE rows always have `Amount=0` (inherits from MAIN). Not yet confirmed whether NeedToBuy's calculation checks ALTERNATIVE availability before flagging MAIN as needed.
- MF_Invoice has TypeID (6 values: MF/SP/MM trade route) and INVTypeID (4 values: EXPORT/DOMESTIC × CTN/KG) as separate classifications. Confirmed pricing rule: TypeID not through MM (1,3,6) → `UnitPrice`/`TotalPrice`; TypeID through MM (2,4,5) → `ActualUnitPriceUSD`/`ActualTotalUSD`.
- Price field names containing "USD" do NOT reliably indicate currency — always check `MF_Invoice.CurrencyID → MF_Currency` (THB/USD/RMB) separately.
- NeedToBuy's real calculation is VB.NET (`BOMTrxCls.UpdateRefBOMTrx`), not a single SQL SP — uses waterfall stock consumption across SaleOrders sharing an RM, checks existing PO reservations before concluding a real shortfall, and gates BOM lines via `BOMdetail.Calculate='Y'`.
- 🔴 **Critical security**: `AutoEmail/ConStr.vb` has a hardcoded plaintext `sa` (SQL Server admin) credential in source code — higher risk than the `xp_cmdshell` credential in `ImportACCT` because it's the highest privilege level and lives in application source that could leak via version control.
- The "AutoEmail" automation system's real notification channel is LINE Messaging API, not email — a separate system from the customer-facing Line OA support bot.
- ⚠ `Profitloss_R2` MAINCAT (corrected 2026-07-04, verified against live data): `06.BTExp(THB)` contains only BT01/02/03 (durian/seafood/dryfood) — `BT00(general)` has its own separate MAINCAT `08.FIXExp(THB)`. SB commission is `07.SBExp(THB)`. To compute net profit/loss, SUM across ALL MAINCATs (01, 02, 03, 04, 06, 07, 08).
- Business meaning of `04.Expense(THB)` categories (verified with system owner): QA Exp = quality-system cost (HACCP/GMP/NFI), CN Exp = cost from customer returns (Credit Note). `SA Exp` and the separate `07.SBExp` MAINCAT are unrelated despite both touching CustType — SA Exp is salesman payroll allocated by responsible salesperson (from S_HIS$/SalesCode); SBExp is business/channel-group operating expense recorded via `[workcode]`, unrelated to salesman pay.
- BOM ALTERNATIVE substitution IS checked before NeedToBuy concludes MAIN needs purchasing — confirmed via SQL `CreateBOMTrx` (called from `CheckBuyQtyR2`, itself called from the VB.NET method of the same confusing name). MAIN and ALTERNATIVE rows share one `MainRMID` group and are processed MAIN-first (via `type desc` ordering), with `Need_Remain` carried forward uncleared between them — so ALTERNATIVE stock only gets checked as a genuine fallback after MAIN's stock is exhausted.
- STOCKTRX cancellation never modifies the original row — it inserts a new row with the opposite-signed quantity and `StockINOUT='CANCEL'`, whose `Remark` references the `StockRunning` of the row being reversed. The original cancelled row's own `Remark` is set to the literal string `'CANCEL'`. Both rows must be excluded (`StockINOUT<>'cancel' AND Remark<>'cancel'`) for correct aggregates.
- `StockPlace` has no clean master list — over 100 free-text values in production, mixing structured warehouse-zone codes with ad-hoc one-off labels (including typos and embedded dates) for stock-count events. Treat as free text, not an enum.
- `ClaimID` format is `CL{MMYY}-{seq}`; ties together the SELL, RETURN, PREMIUM, and CANCEL rows belonging to one customer claim event. Revealed 2 previously undocumented `ProjectID` values: `RETURN` and `PREMIUM`.
- `ProjectID='ADJUST STOCK'` real-world trigger confirmed: annual year-end physical stock count, tagged with `StockPlace='MF_Check Stock'` and a Remark describing the count event (e.g. `เช็คสต็อค2026`).
- AP (Accounts Payable) and AR (Accounts Receivable) reporting is exposed via **table-valued functions** `APMain_R`/`APDetail_R`, `ARMain_R`/`ARDetail_R`, `ARMMMain_R`/`ARMMDetail_R` — NOT base tables, so they won't show up in an `INFORMATION_SCHEMA.TABLES` search; check `sys.objects` for `SQL_TABLE_VALUED_FUNCTION` instead. Parameter `@YearMonth` MUST be `'YYYY/MM'` format (e.g. `'2026/06'`) — the SP parses year from position 1-4 and month from position 6-7, requiring a separator at position 5. Same functions the `AP_Report.xlsx` VBA tool queries.
- `APTrans`/`ArTrans` are the raw ledger transaction logs behind the AP/AR TVFs — one row per credit/debit event tagged with a `Trans` type (`PAY`/`STOCKIN`/`NON-STOCKIN` for AP; `DELIVERY`/`RECEIVED`/`CN`/`MFIT_SOI` for AR).
- `NON-STOCKIN` POs are for services/testing/non-inventory expenses (e.g. Intertek Testing Services) — they flow through AP payment tracking but are excluded from STOCKIN-based delivery-percentage calculations.
- `CreateNowPO` (the last of 4 steps inside `CheckBuyQtyR2`) builds the `NowPO` table — one row per not-yet-fully-received PO detail line with remaining quantities pre-computed — confirmed as the actual source table behind `GetRemainingPORMStock()`/`UpdatePORMReserved()` in the VB.NET NeedToBuy algorithm (§8.4).
- `ARTrans`/`APTrans`/`AR_MMTrans` are built by `Create_ARTrans`/`Create_APTrans`/`Create_AR_MMTrans` (full-refresh, depend on a `lastdate` master table + `Create_DocnoCompany`) — ⚠ these SPs parse `@YearMonth` via `LEFT(4)`/`RIGHT(2)` (separator-agnostic, convention `'YYYY-MM'` with a dash) which is a DIFFERENT convention from the `_R` reporting functions' rigid `'YYYY/MM'` slash requirement — the two layers do not share a calling convention, always verify which one is being called before assuming a date-parameter format.
- Company-wide financial health is checked via 3 DELIBERATELY SEPARATE systems (owner-confirmed) — Profitloss (§10, transaction-level P&L per invoice/product), IncomeStatement (§11.3, annual P&L view), BalanceSheet (§13, asset/liability/equity position) — do not assume any of these union together; `APIncomeStatement`'s `MAINCAT='09/AP'` numbering is internal to its own system, not a shared cross-system sequence.
- 🔴 **AP figures for a historical month require `EXEC Create_APTrans '@YearMonth'` immediately before querying `APMain_R`/`APDetail_R('@YearMonth')`** — `APTrans` is a single shared full-refresh table, and `Create_APTrans`'s docno-inclusion logic uses `GETDATE()` not the parameter, so calling the `_R` functions standalone for a past month reads whatever `APTrans` was last rebuilt as (usually the most recent month), NOT that historical month. A real ~3× discrepancy (June 2026: 2.03M in BalanceSheet vs 6.99M from a standalone APMain_R call) was traced to exactly this — `GetBalanceSheet`'s internal cursor loop does this correctly (Create_APTrans then APMain_R, per month, in sequence); replicate that pattern for any historical-month AP query.
- `tracking.date` is stored as TEXT (`DD-MM-YYYY`) — `ORDER BY date DESC` sorts alphabetically and gives WRONG results (e.g. `'31-12-2025'` before `'04-07-2026'`). Use `MAX(CONVERT(date, date, 103))` to check task recency correctly.
- `SetExchangeRate` appearing stale is NOT a broken job — it was deliberately superseded by `ExchByINV`/`AvgExchByINV` (§11.7), which derive the REAL exchange rate per invoice from actual bank receipts (THB received ÷ USD received) rather than trusting a static nominal daily rate. `UpdatePriceFromPODocno` remains a genuinely unexplained stale task (3+ years) by contrast — don't assume every stale-looking task has a benign explanation without checking.
- The real function name is `UploadPOAuto()` (class `googleCls`), not "UpdateLoadPOAuto" — it uploads attachments for 8 document types (not just PO) to Google Drive via a `needUploadPO` queue table, writing the resulting URL back to type-specific tables (`img_soi`, `img_ProductPO`, `img_soipo`, `img_soipipo`, `img_inv`, `stxapprove`, `CostVatHD`, `img_PO`). Found a 3rd hardcoded credential (Google OAuth Client ID/Secret) alongside the `sa` password and network-share credential found earlier — recommend a single consolidated security remediation pass covering all three.
- There is an entire Balance Sheet module (`BalanceSheet`/`BalanceSheetIncome` tables, `APIncomeStatement`/`ARBalanceSheet`/`ARMMBalanceSheet`, `RefreshBalanceSheet`) parallel to the P&L Pipeline.
- The full daily scheduled-task list (via `tracking` table types) is: `LastDate`, `SetExchangeRate`, `ImportAcct`, `UpdatePriceFromPODocno`, `CreateInventory`, `CreateAcct`, `CreateBalanceSheet`, `BackupStatement` — larger than what §12's AutoEmail walkthrough alone covers; there is also a separate SQL Server Agent Job named `'Balance Sheet'`, distinct from the Windows Task Scheduler mechanism used by the AutoEmail VB.NET project.
- `BalanceSheet.MAINCAT` has exactly 3 values: `ASSET`, `LIABILITY`, `EQUITY` — EQUITY is not sourced from any ledger, it's computed as the plug figure `SUM(ASSET) − SUM(LIABILITY)` per month (the standard accounting identity), added as a final UNION step inside `GetBalanceSheet`.
- `GetBalanceSheet`'s Account Payable / Prepaid Expense figures come from a **monthly CURSOR loop** that re-runs `Create_APData`→`Create_APTrans(@YearMonth)`→reads `APMain_R(@YearMonth)` for every month since 2023 — i.e. the Balance Sheet's AP line is sourced from the same AP pipeline documented in §11.5, just invoked once per historical month rather than once for the whole range.
- Watch for the typo'd real object name `Create_Trade_Receivabale` (missing a 'b') when searching for it — the correctly-spelled `Create_Trade_Receivable` does not exist.
- `MF_Invoice.PaymentStatus='Completed'` triggers AR closure (feeds `ARMain_R`), `MF_PO.PaymentStatus` triggers AP closure (feeds `APMain_R`).
- `MF_PODetail.QTYReceived`/`ReceivingStatus` are populated from real STOCKTRX BUY-in movements via `UpdatePODetailQtyReceive`/`_Batch`, not manual entry — `ReceivingStatus='Completed'` once the full ordered qty has arrived.
- Current MF_Invoice DocNO convention: 4 legitimate prefix families — `INV`/`CN` (with `EX`=export / `D`=domestic suffix correlating to `INVTypeID`) plus `TF`/`TSX` (both TypeID=5 "SP to MM to CUS", still actively used through mid-2026) — genuine legacy one-offs (`QUO`, `ABC`, `PI19-001` etc.) are a separate, truly-dead-data bucket from 2017-2020; don't lump TF/TSX in with those.
- 🔴 **Critical**: when a query has an outer signed-kg CTE (`case when type='INPUT' then kg else -kg end`) AND a nested subquery that re-declares the SAME alias name (e.g. `pi`) against the raw base table, the nested subquery's `kg` silently resolves to the UNSIGNED raw column, not the outer signed value — SQL Server does not warn about this. This caused the CycleTime sign bug (§4.3): REMAINING got added instead of subtracted, inflating CycleTime up to ~29× in real cases. Always check whether a derived-table alias is genuinely the same derived table or just a same-named fresh reference to the base table.
- A SP that does `DROP TABLE X` (no existence guard) followed by `SELECT ... INTO X` will fail on the very first cold-session run (table doesn't exist yet to drop) — always guard with `IF OBJECT_ID('X','U') IS NOT NULL DROP TABLE X`. Found in `CalculateNewLoss` (§4.7).
- A `WHILE (@rnd < N)` loop with multiple `IF/ELSE IF` exit branches must set the loop-terminating variable in EVERY branch that's meant to exit — a branch that only resets a secondary counter (like `@newCNT=0`) without also advancing `@rnd` can leave the loop spinning forever if that exact branch condition keeps re-triggering. Found in `UpDateRecalLoss`'s `@rnd=2` branch (§4.7) — the sibling `@newCNT=0` branch right below it did this correctly (`set @rnd=6`), making the omission easy to miss on a quick read.
- A table built via `DROP TABLE X; SELECT ... INTO X` inside a stored procedure (e.g. `Recal_Cal` via `Create_Recal_Cal`) is a CACHE, not a live view — redeploying the SP that builds it does NOT retroactively fix the cache's current contents. You must re-execute the building SP at least once before the cache reflects the fix, or comparisons against it will silently keep using stale pre-fix values (§4.4).
- CycleTime, `DLThbKg`, and `LGCost` are 3 independent but structurally parallel accumulation systems that compound across the production chain (a value stamped on one production's output becomes the "inherited" input to the next production that consumes it as raw material) — CycleTime and DLThbKg both add a fresh `manhour`-based term each round; LGCost is pure pass-through with no fresh term. Fixing a bug in one has zero effect on the others (verified: separate stocktrx columns, separate SPs, no cross-reference) — don't assume a fix needs to be mirrored across all three without checking each one's actual formula first (§4.5).
- `UpdateRecalCost`/`UpdateRecalDL`/`UpdateRecalLoss`/`UpdateCostFromRootlotno` are NOT manual-only maintenance SPs — they are tracked in the `tracking` table as daily scheduled automation (run counts in the 800-1,400+ range, confirmed running as of 2026-07-09), same idempotency mechanism as `UpdateNeedToBuy`/`ImportAcct` (§12.1.1). Before assuming any SP is "only ever run by hand," check `SELECT DISTINCT type FROM tracking` first.
- `UpdateApproveProduction`'s post-insert negative-balance guard (§17.1, deployed 2026-07-08) checks EVERY row on every rootlotno a production touches, not just that production's own newly-inserted rows — so an approval can fail because of an unrelated, much OLDER production's leftover anomaly, with zero problem in the production actually being approved. Always run the 2-step diagnostic (§17.2: check `chkproduction` for the production itself, THEN check for pre-existing negative balances on its rootlotnos) before assuming the failing production itself is at fault.
- When remediating a negative `StockLotBalanceUnit` that's blocking `UpdateApproveProduction`'s guard, an offsetting `ADJUST STOCK IN` insert ALONE is not enough — the guard counts every row with a negative balance, so the ORIGINAL anomalous row's own `StockLotBalanceUnit` must also be UPDATEd to 0 ("balance override"). Two steps, both required (§17.4).
- `Active=0` on an IN/MOVEIN stocktrx row is normal once that row's balance has been fully drawn down by a later OUT/MOVEOUT on the same StockID — it is NOT a "this lot was deactivated/invalid" signal (an earlier hypothesis in this KB's own investigation history got this wrong before being corrected against a full 36-row chain). The real signal of a double-draw anomaly is a negative `StockLotBalanceUnit` on an `Active=1` row, not the `Active` value on the row that created the lot (§17.3).

---

## 15. OPEN QUESTIONS / KNOWN GAPS — do not answer these confidently

Most items from earlier versions of this document were resolved on 2026-07-04 (see Change Log). Remaining genuine gaps:

| Topic | Status |
|---|---|
| `CostVatHD.TotalVatAmount` / `TotalWHTAmount` exact formula | Columns exist and were checked with real data, but the VAT figures found look too small to be standard 7% VAT amounts — meaning/formula unclear, needs system-owner clarification (§3.5) |
| `ProfitlossS1` special-case branches | High-level structure confirmed (§10.5) but several hardcoded per-product-code and per-channel exception branches not fully enumerated |
| `Email.vb` individual function bodies | Full function name catalog confirmed (§12.4) but none of the ~40 function bodies have been read individually |
| SB commission for WHOLESALE customers | No SB salesCode maps to WHOLESALE; moot while 0 such customers exist, but unresolved in principle (§10.4) |
| `CalculateNewLoss` — `AccLoss%` across chained productions | Core `NewLoss` formula verified with real data (§4); the accumulated/recursive loss-across-productions part of the same SP was not independently re-verified beyond the simple zero-loss case checked |
| `AvgSMCost` fan-out bug in `CalculateNewCost`'s `sm` subquery | `[OPEN — carried from earlier, still unresolved 2026-07-09]` Suspected fan-out when a submat `produceitemid` matches multiple `stocktrx` rows, inflating `SMCost`; not yet reproduced against a concrete real example the way the CycleTime bug was (§4.3/§4.9) — needs its own investigation before fixing |
| `UpDateRecalCost` performance optimizations | `[PROPOSED, NOT DEPLOYED 2026-07-09]` Pushing `@productionid` filter into `CalculateNewCost`'s internal subqueries, and eliminating `UpDateRecalCost`'s redundant double-call of `Create_Recal_Cal` per loop iteration — both designed and explained (§4.8) but not yet written into a deployable script or confirmed by the owner |
| `RecalDL`'s `val` subquery `ProductionID`-only join (§4.6) | Flagged as a fan-out risk for multi-output productions but not reproduced against a concrete real example or fixed — low priority unless a production with 2+ OUTPUT items is confirmed affected |
| Schedule time-of-day for `UpdateRecalCost`/`UpdateRecalDL`/`UpdateRecalLoss`/`UpdateCostFromRootlotno` (§12.1.1) | Confirmed these run daily via the `tracking` idempotency mechanism, but the exact time-of-day (or whether they run at a single fixed time vs. multiple slots like `UpdateNeedToBuy`) has not been checked |
| System-wide sweep for pre-2026-07-08 double-draw anomalies (§17.5) | 2 confirmed instances found incidentally (§17.3); no proactive `StockLotBalanceUnit < 0 AND Active = 1` scan across the whole database has been run yet — likely more exist undiscovered until a new production happens to touch the same affected rootlotno |
| `stockplacefull2` full logic | Opened only to confirm it exists as a working-table-prep step in `CheckBuyQtyR2` (§8.4); its own internal SQL not read in detail |
| `CostVatHD.Approve` — who/what sets this | Confirmed 3 values exist (`APPROVE`/`NOT APPROVE`/blank) and that `APDetail_R` reads it, but not which SP or UI action writes it |
| `ARMMDetail_R`/`ARMMMain_R` internals | Confirmed to exist as TVFs with the same `@YearMonth` parameter convention as the main AR/AP functions, but body not yet opened — likely parallels `ARDetail_R` for the MM entity |
| `tracking.date` sort order | `[RESOLVED]` — column is stored as TEXT (`DD-MM-YYYY`), not a real date type. `ORDER BY date DESC` sorts alphabetically and gives WRONG chronological order (e.g. `'31-12-2025'` sorts before `'04-07-2026'`). Always use `MAX(CONVERT(date, date, 103))` or equivalent when checking "most recent" task runs. |
| `UpdatePriceFromPODocno` scheduled task | Confirmed genuinely stale as of 2026-07-04: last ran 2023-02-13 (3+ years) — worth escalating to whoever owns the AutoEmail/Balance-Sheet scheduler to determine if it's abandoned/replaced (like `SetExchangeRate` turned out to be, §11.7) or a real gap |
| `GetBalanceSheetIncome`/`BalanceSheet_AP` internals | `GetBalanceSheet` itself is now fully documented (§13.1.1); `GetBalanceSheetIncome` (the `BalanceSheetIncome` builder) and the standalone logic of `BalanceSheet_AP` beyond what `GetBalanceSheet`'s cursor loop does are still unopened |
| `ARBalanceSheet`/`ARMMBalanceSheet` parameter convention | Confirmed their output columns (`AR`, `DFR`) are consumed directly by `GetBalanceSheet` (§13.1.1), and confirmed to exist as TVFs, but whether they take a `@YearMonth` parameter (and in which format) was not confirmed — they were called with no visible parameter in the `GetBalanceSheet` body (`ARBalanceSheet()`), suggesting they may compute for ALL months internally rather than being date-parameterized like the AP/AR `_R` functions |
| The 2022 `CGS-LP-DM%` special-case override in `APIncomeStatement` | Code comment marks it "new logic" from 2022 onward but doesn't explain what changed in the underlying business process |
| `UpdatePriceFromPODocno`, `CreateInventory`, `CreateAcct` (scheduled task types) | Confirmed to exist as tracked daily task types (§13.3) via the `Refresh*` SPs, but none of their own stored procedures have been opened (`SetExchangeRate`'s replacement mechanism is now documented — see §11.7) |
| `Create_Trade_Receivabale`/`Create_Trade_ReceivableMM`, `inventorybylastdate`, `trade_receivable`/`Trade_ReceivableMM` | All confirmed as real dependencies of `GetBalanceSheet` (§13.1.1) supplying Trade Receivable and Inventory figures, but none of their own bodies have been opened |

---

## 16. KB DISTRIBUTION ARCHITECTURE & MCP ACCESS `[IN PROGRESS — 2026-07-07]`

Context: this KB (`Mildfoods_KB_AI.md`) was originally consumed only via Claude Project's built-in `project_knowledge_search` (semantic chunked search, scoped to this Project). This section covers the parallel effort to make the same KB readable by **other AI clients / MCP-compatible tools**, not just this Project.

### 16.1 Three architecture options considered

| Option | How it works | Trade-off |
|---|---|---|
| **Filesystem MCP server** (official `@modelcontextprotocol/server-filesystem`) | Points at a folder (e.g. the network share); client gets `read_file`/`list_directory`/`search_files` (filename search only, not content search) | Zero code to write, but no content search — client must read the whole file every time; also client-side must be able to reach/mount the network share |
| **Custom Python MCP server** | Hand-written server exposing a `search_kb(query)` tool that chunks by markdown header (`##`/`###`) and returns matching sections, plus `get_kb_info()` for version/size metadata | Closest to `project_knowledge_search` behavior (chunked search vs. whole-file dump); requires maintaining a small Python service |
| **`OPENROWSET` inside SQL Server** ✅ **chosen direction** | SQL Server itself reads the `.md` file (via `BULK`/`SINGLE_CLOB`) and returns it as a query result; wrapped in a view (`dbo.vw_Mildfoods_KB`) | Client only needs a normal SQL Server connection (reuses the existing `mssql-mcp-node` connector already in use for DB access) — no separate MCP server or network-share mount needed on the client side |

**Reason for choosing OPENROWSET**: the existing MCP setup already has a working SQL Server connector (`mssql-mcp-node`). Serving the KB through the same channel avoids standing up and maintaining a second MCP server, and means any client that can already query the databases (MildFoods/inventory/STAGING) automatically gets KB access too.

### 16.2 OPENROWSET implementation requirements `[VERIFIED — tested query pattern, not yet confirmed working in production]`

Two **separate** permission layers are involved — a common point of confusion:

1. **SQL Server *service account*** (the Windows/domain identity the SQL Server process itself runs as — check via `Services.msc` → SQL Server instance → "Log On" tab) must have OS-level read permission on whatever path the `.md` file lives at. If the service account is `Local System`/`Network Service`, it typically **cannot** reach a UNC/network path — needs a domain service account instead. If SQL Server and the file share are the same physical machine, this is a non-issue (local path, no network auth involved).
2. **The SQL *login* executing the query** (e.g. `mcp_kb_reader`) needs the `ADMINISTER BULK OPERATIONS` server permission — this is separate from and in addition to ordinary `SELECT` grants, and is specifically required for `OPENROWSET(BULK...)` to work.

Working query pattern (once both permission layers are satisfied):
```sql
SELECT BulkColumn AS kb_content
FROM OPENROWSET(BULK N'\\path\to\Mildfoods_KB_AI.md', SINGLE_CLOB) AS kb;
```
Wrapped in a view (`dbo.vw_Mildfoods_KB`, created in `inventory` DB) so clients just run `SELECT kb_content FROM dbo.vw_Mildfoods_KB;` without needing to know the underlying file path.

**Note**: this does NOT require enabling server-wide `Ad Hoc Distributed Queries` — that setting only applies to OPENROWSET's linked-server/provider mode, not the `BULK`/`SINGLE_CLOB` file-read mode used here.

### 16.3 Integration point: `ImportACCT` / AutoEmail scheduling `[OPEN QUESTION — design decision not yet made]`

Considered folding the KB read directly into `ImportACCT` (§11.1) as a new step (copy `.md` from `\\nas` locally via the same `xp_cmdshell` pattern already used for the 12 Excel sources in that SP, then `OPENROWSET` it into a new `KB_Content` table). This would piggyback on the existing full-refresh pattern.

**Unresolved trade-off**: `ImportACCT` runs as the `ImportAcct` tracked task type inside the `tracking`-table idempotency mechanism (§12.1/§13.3) — guarded to run **once per day only**. If the KB read is folded into this SP, editing the KB mid-day would not be reflected in `KB_Content` until the next day's run (or a manual `DELETE FROM tracking WHERE type='ImportAcct' AND date=...` to force a re-run, same trick `RefreshBalanceSheet`/`RefreshIncomeStatement` use). Alternative: give the KB its own independent tracked task type (e.g. `ImportKB`) so its refresh cadence isn't coupled to the accounting data's cadence. **Not yet decided as of this entry — pending owner preference.**

### 16.4 Security finding from this work: third hardcoded credential

While reviewing the existing `mssql-mcp-node` MCP connector config, found the SQL Server **`sa`** account (highest privilege) configured with a plaintext password directly in `claude_desktop_config.json`. This is a **third** instance of hardcoded/plaintext credentials found in this project, alongside:
- `xp_cmdshell` network-share credential inside `ImportACCT` (§11.1)
- `ConStr.vb` hardcoded `sa` connection string in application source (§12.3)

**Remediation applied**: created a dedicated least-privilege login `mcp_kb_reader` (SELECT-only on `dbo` schema across MildFoods/inventory/STAGING, plus `ADMINISTER BULK OPERATIONS` for the OPENROWSET use case in §16.2; explicit `DENY` on INSERT/UPDATE/DELETE/EXECUTE) for all MCP/AI-client SQL connections going forward. `sa` should not be used for this purpose, and its password should be rotated separately (not yet confirmed done as of this entry).

**Pattern to flag to the owner going forward**: every time a new integration/connector is set up (MCP, ETL, VB.NET tool, etc.), check whether it defaults to `sa` or a shared admin account before wiring it in — this is now the 3rd occurrence of the same anti-pattern across genuinely different subsystems (SSIS-era ETL, legacy VB.NET app, and now modern MCP tooling), suggesting it's a standing habit rather than an isolated mistake.

### 16.5 SSMS client connection failure: "target principal name is incorrect" `[VERIFIED — 2026-07-20]`

**Symptom**: SSMS's "Connect to Server" dialog fails against `mfit.dyndns.biz` (Database Engine, SQL Server Authentication, login `sa`) with:

> Cannot connect to mfit.dyndns.biz. A connection was successfully established with the server, but then an error occurred during the login process. (provider: SSL Provider, error: 0 - The target principal name is incorrect.)

**Root cause**: the Connection Security tab had `Encryption: Mandatory` set with "Trust server certificate" unchecked. In that mode, SSMS validates the server's SSL/TLS certificate against the hostname actually dialed — `mfit.dyndns.biz` (a DynDNS name). If the certificate installed on the SQL Server instance was issued for a different name (the machine's real Windows hostname, an internal name, or a self-signed cert with no SAN entry for the DynDNS name), certificate validation fails with exactly this message. This is a client-side TLS/certificate-hostname mismatch, not a network or credentials problem — the dialog's own wording confirms the network connection succeeded and login/handshake is what failed.

**Fixes** (any one resolves it; ordered quick fix → permanent fix):
1. Quick: on the Connection Security tab, check "Trust server certificate", then Connect.
2. Or: change `Encryption` from `Mandatory` to `Optional`.
3. Or: enter the certificate's actual CN/SAN name in the "Host name in certificate" field (the name the cert was really issued for — not `mfit.dyndns.biz`).
4. Permanent: reissue/install an SSL certificate on the SQL Server instance with `mfit.dyndns.biz` in the CN or SAN list, so it matches the DynDNS hostname every client actually connects through.

**Scope note**: this only affects TLS-strict clients (SSMS with mandatory encryption) connecting via the DynDNS hostname. The MCP `mssql-mcp-node` connector (§16.4, `mcp_kb_reader` login) documented elsewhere in this KB connects successfully to the same `mfit.dyndns.biz:1433` endpoint — confirming this is an SSMS client-configuration issue, not a server-side outage, and has no impact on live MCP/AI-client database access.

**Relationship to §18's DNS-redirect finding**: this is a separate mechanism from the LAN Static-DNS misroute documented in §18. §16.5 is a certificate/hostname-identity mismatch that can happen on *any* network path once you reach a SQL Server whose cert doesn't match the name you dialed; §18 is LAN clients being sent to the *wrong physical machine* entirely (NAS instead of "MF") before a TLS handshake even starts. A LAN client could in principle hit both in sequence — misrouted to the NAS proxy (§18), then if that hop terminates/forwards TLS in a way that still presents a non-matching cert, also see the §16.5 symptom — but fixing one does not fix the other; they were diagnosed and resolved independently.

---

## 17. PRODUCTION APPROVAL & STOCKTRX NEGATIVE-BALANCE REMEDIATION `[NEW 2026-07-09]`

### 17.1 `UpdateApproveProduction` — the real approval gate for a production run

Previously undocumented in this KB. Called as `EXEC UpdateApproveProduction '@productionid'` to formally approve a production (flips `produceHD.status` to 1, inserts the real `stocktrx` rows from a staging table `chkproduction`). Current version carries 2 fixes dated 2026-07-08, per its own header comment:

```
1) RACE CONDITION FIX: sp_getapplock (Exclusive, @LockOwner='Transaction') acquired on every
   distinct rootlotno this production touches, BEFORE the pre-check, sorted alphabetically to
   avoid deadlocks between two approvals sharing multiple rootlotnos. Prevents two concurrent
   approvals from both passing a check against the same stale snapshot and both inserting →
   double-draw → negative balance.
2) POST-INSERT NEGATIVE-BALANCE GUARD: after inserting the approved production's stocktrx rows,
   a hard check scans EVERY row (not just this production's own) on every rootlotno touched —
   if ANY row has StockLotBalanceUnit < 0 (Active=1, not cancelled), the ENTIRE approval is
   rolled back (THROW 51003). This is a safety net independent of ChkProduction's own matching
   logic, so it also catches pre-existing anomalies left by OLDER approvals that predate this
   fix (see §17.2) — not just problems this specific approval would itself cause.
```

**Flow**: `ChkProduction` pre-check (per-row OK/NOT OK classification for this production's own lines) → if OK, `DELETE` + re-`INSERT` this production's own `stocktrx` rows from `chkproduction` → row-count sanity check (added 2026-06-30) → flip `produceHD.status=1` → post-insert negative-balance guard (2026-07-08) → `COMMIT`, or `ROLLBACK` + error detail on any failure.

**Tooling note**: `EXEC` is blocked via the read-only `execute_read_query` mssql tool — diagnose by querying `chkproduction` directly with the same CASE expression the SP uses internally, and by manually checking for existing negative-balance rows on the production's rootlotnos (§17.2). (When `execute_write_query` is enabled, `EXEC` itself works directly — §17.16.)

### 17.2 Diagnosing an approval failure — 2-step check

When `UpdateApproveProduction '@productionid'` fails to approve, check in this order:

**Step 1 — does this production's own consumption math pass?**
```sql
SELECT lotno, productionitemid, productid, stockinout, StockRef, StockLotBalanceUnit,
(-stockperunit) as NeedUnit, (-Stockperunit)+(StockLotBalanceUnit) as StockUnit,
CASE
    WHEN stockinout = 'IN' AND StockLotBalanceUnit >= 0 THEN 'OK'
    WHEN stockinout = 'IN' AND StockLotBalanceUnit < 0 THEN 'Unit Less than 0'
    WHEN stockinout = 'OUT' AND StockRef IS NOT NULL AND StockLotBalanceUnit >= 0 THEN 'OK'
    WHEN stockinout = 'OUT' AND stockRef IS NULL THEN 'Stock not found!!'
    WHEN stockinout = 'OUT' AND StockRef IS NOT NULL AND StockLotBalanceUnit < 0 THEN 'Stock not enough!! '
    ELSE 'NOT OK'
END as ERROR
FROM chkproduction WHERE productionid = '@productionid'
```
If every row shows `ERROR='OK'`, the production itself is fine — the block is coming from Step 2, not this production's own math.

**Step 2 — does any rootlotno this production touches already have a pre-existing negative balance?**
```sql
-- get the rootlotnos this production touches
SELECT DISTINCT rootlotno FROM produceitem WHERE productionid='@productionid' AND rootlotno IS NOT NULL

-- then check for existing negative-balance rows across ALL of them (not just this production's own rows)
SELECT StockID, rootlotno, ProductID, ProjectID, StockINOUT, StockLotBalanceUnit, Active, Remark
FROM stocktrx 
WHERE rootlotno IN (...)
  AND StockLotBalanceUnit < 0 AND active = 1 AND StockINOUT <> 'cancel' AND remark <> 'cancel'
```
If Step 1 passes but Step 2 finds a row, THAT row is what's blocking — not the production being approved. It's almost always a pre-existing anomaly left by a **different, older** production's approval.

### 17.3 Root cause pattern: "double-draw at approval, before the 2026-07-08 guard existed"

**Confirmed via 2 real examples** (2026-07-09): every negative-balance row found this way traces back to a production that was approved **before 2026-07-08** (when neither the lock fix nor the post-insert guard existed). In both cases, the anomalous `StockID` has exactly 2 stocktrx rows — one IN creating 1 unit, one OUT consuming 1 unit — yet the OUT row's own balance shows -1 instead of the expected 0, which is mathematically only possible if a second, untracked draw against that same single-unit lot happened concurrently at approval time (exactly the scenario the 2026-07-08 lock fix was designed to prevent going forward).

| Case | StockID | rootlotno | Product | Anomaly created by | Date |
|---|---|---|---|---|---|
| 1 | `150925-3280` | `150925-0749` | DSQ108 | `PR290626-0004` (approved) | 29/06/2026 |
| 2 | `140924-1232` | `140924-0049` | DSQ088 | `PR150126-0023` (approved) | 15/01/2026 |

Both productions **successfully approved** at the time — there was no contradiction: the guard that would have caught this didn't exist yet. The anomaly then sat silently (undetected, since nothing was checking for it) until a **later, unrelated** production happened to draw from the same rootlotno and got blocked by the new guard.

**Corrected note on the `Active` column** (an earlier hypothesis in this same investigation was wrong and is corrected here): `Active=0` on an IN/MOVEIN row that later gets fully consumed by a subsequent OUT/MOVEOUT on the same StockID is the NORMAL pattern throughout this system's stocktrx history — it does **not** mean "deactivated" or "invalid." Confirmed by scanning an entire 36-row rootlotno chain: essentially every IN/MOVEIN row shows `Active=0` once its balance has been drawn down by a later transaction, while the drawing (OUT/MOVEOUT) transaction itself shows `Active=1`. Do not use `Active=0` alone as evidence a lot was improperly deactivated — the real tell for this bug is a **negative `StockLotBalanceUnit` on an `Active=1` row**, not the `Active` value on the row that created the lot.

### 17.4 Established remediation pattern — 2 steps, both required

**Critical**: the post-insert guard in `UpdateApproveProduction` (§17.1) counts **every row** with `StockLotBalanceUnit < 0`, not the net/current balance of the rootlotno. Adding an offsetting `ADJUST STOCK` entry alone is **not sufficient** — the original negative row's own stored value must also be corrected, or the guard will still trip on that old row.

**Step 1 — INSERT a compensating `ADJUST STOCK IN`** of +1 unit (or whatever the shortfall is), sourcing every cost-related field from the exact row that recorded the anomaly (same `rootlotno`, same `StockID` where practical) — never from a different rootlotno sharing only ProductID:
```sql
-- Fresh MAX(StockRunning) immediately before insert (live table, concurrent inserts)
SELECT MAX(StockRunning) FROM stocktrx WHERE ISNUMERIC(StockRunning)=1 AND LEN(StockRunning)=8

INSERT INTO stocktrx (...)
-- ProjectID = 'ADJUST STOCK'
-- Remark = 'ADJUST BY CLAUDE - reconcile double-draw'   (≤50 chars, mandatory prefix)
-- InvoiceNo = 'CLAUDE AI'                                (mandatory)
-- StockTrxDate = one day before the earliest transaction in this StockID's own partition
-- Cost / InventoryCost / cycletime / smcost / LGCost / SupplierName / SupplierID / CostID /
--   CostDetailID / etc. = copied from the anomalous row itself (same rootlotno, same lot)
```

**Step 2 — UPDATE the original negative row's own `StockLotBalanceUnit` to 0** (a "balance override," one of the explicitly-sanctioned UPDATE categories in this system's data-correction conventions), and set `InvoiceNo='CLAUDE AI'` on it too:
```sql
UPDATE stocktrx SET StockLotBalanceUnit = 0, InvoiceNo = 'CLAUDE AI' WHERE StockRunning = '@the_anomalous_row'
```

**Verify** before telling the owner it's fixed:
```sql
SELECT COUNT(*) FROM stocktrx 
WHERE rootlotno='@rootlotno' AND StockLotBalanceUnit < 0 AND Active=1 AND StockINOUT<>'cancel' AND remark<>'cancel'
-- must return 0
```

Both cases in §17.3 were fixed this way on 2026-07-09 and verified to return 0.

### 17.5 Open item: likely more pre-2026-07-08 double-draw anomalies exist system-wide

Both cases found so far were discovered incidentally (a new, unrelated production happened to reuse the same rootlotno and got blocked). No systemic sweep for this specific pattern (`StockLotBalanceUnit < 0 AND Active = 1`, system-wide, not scoped to any one rootlotno) has been run yet as of this KB version — proposed but not yet executed. Given 2 confirmed instances found purely by coincidence within one day of each other, and given one dates back to January 2026 (nearly 6 months undetected), there are very likely more sitting undiscovered until a production happens to touch the same lot again. A proactive system-wide query would surface all of them at once rather than waiting for each to be discovered one at a time via a blocked approval.

### 17.6 `GetIncompletedProduction` — a second, independent diagnostic SP `[NEW 2026-07-10]`

Previously undocumented. Full body:

```sql
CREATE procedure [dbo].[GetIncompletedProduction]
as
begin
select count(*) 'cnt', ph.productionid into #pi
from produceHD ph left join produceitem pi on ph.PRODUCTIONID = pi.ProductionID
group by ph.productionid

select count(*) 'cnt', productionid into #st
from stocktrx
where stockinout <> 'cancel' and remark <> 'cancel' and projectid = 'production'
group by productionid

select top 1000 pi.productionid from
 #pi pi left join #st st on pi.productionid = st.productionid
 left join producehd ph on pi.productionid = ph.productionid
where pi.cnt <> st.cnt and year(convert(Date,ph.trxdate,103)) > 2021
order by pi.productionid asc
end
```

**What it checks**: for every `ProductionID`, does the row-count of `produceitem` (all lines: INPUT/OUTPUT/SUBMAT/REMAINING) equal the row-count of its **non-cancelled** `STOCKTRX` rows (`ProjectID='production'`, excluding both `StockINOUT='cancel'` AND `Remark='cancel'` rows — same double-filter convention as §3.1.1)? If not, the `ProductionID` is flagged. `EXEC` is blocked by the mssql tool as usual — replicate with inline subqueries (temp tables `#pi`/`#st` don't survive across the tool's separate calls anyway):

```sql
SELECT pi.productionid, pi.cnt AS pi_cnt, st.cnt AS st_cnt
FROM (
  SELECT count(*) 'cnt', ph.productionid
  FROM produceHD ph LEFT JOIN produceitem pi ON ph.PRODUCTIONID = pi.ProductionID
  GROUP BY ph.productionid
) pi
LEFT JOIN (
  SELECT count(*) 'cnt', productionid
  FROM stocktrx
  WHERE stockinout <> 'cancel' AND remark <> 'cancel' AND projectid = 'production'
  GROUP BY productionid
) st ON pi.productionid = st.productionid
LEFT JOIN produceHD ph ON pi.productionid = ph.productionid
WHERE pi.cnt <> st.cnt AND YEAR(CONVERT(date, ph.trxdate, 103)) > 2021
ORDER BY pi.productionid ASC
```

**Critical join-key finding**: `stocktrx.produceitemid` links directly and exactly to `produceitem.ProductionItemID` (1:1, confirmed) — this is the reliable way to find which specific STOCKTRX row (if any) corresponds to a given produceitem line, far more reliable than matching `produceitem.LOTNO` to `stocktrx.StockID` by value alone. **Trap**: `produceitem.ProductionItemID` is **NOT unique** across the whole table — the same ID string (e.g. `PI090626-0001`) is reused across dozens of unrelated `ProductionID`s (confirmed: one ID value matched 46+ rows across different productions). Any lookup or join on `ProductionItemID` must always also filter by `ProductionID`, or results will be nonsense.

### 17.7 Critical data-hierarchy principle — `produceitem` is master, `STOCKTRX` must follow `[VERIFIED W/ OWNER 2026-07-10]`

**Owner's explicit, standing correction** (applies to all future work of this kind, not just this session): `produceitem` is the MASTER/authoritative record of what actually happened on the production floor. `STOCKTRX` is the derived, auto-generated ledger and must always be corrected **to match** `produceitem` — **never the reverse**. When `GetIncompletedProduction` (or any similar row-count check) flags a mismatch, the fix is to **add/restore/correct STOCKTRX entries** so they align with `produceitem` — **do not delete or alter `produceitem` rows** just to make the counts agree, even when a `produceitem` row looks like an "orphan" or "duplicate" from the STOCKTRX side.

This reverses an initial (wrong) approach taken early in this same investigation, where deleting the "extra" `produceitem` row was proposed first — flagged and corrected by the owner before any deletion was executed.

### 17.8 Two sub-patterns behind a `GetIncompletedProduction` mismatch, and how each is fixed

All mismatches found so far take the form `pi.cnt > st.cnt` (produceitem has more rows than active STOCKTRX) by exactly the shortfall amount, and trace back to a **cancel-at-the-STOCKTRX-level event that was never reflected back into `produceitem`**. Two distinct physical scenarios were found, requiring different remediation:

**Sub-pattern A — "cancel, then redo" (4 of 5 cases found)**: the input's STOCKTRX consumption was cancelled and immediately re-drawn (new `StockID`, same rootlotno, same production, same day) — `produceitem` still correctly lists the ONE input line (via its own `ProductionItemID`), and its underlying STOCKTRX row genuinely still exists, just cancelled. **Fix**: INSERT a new active `PRODUCTION`/`OUT` STOCKTRX row referencing that exact `produceitemid`, sourcing every field from (a) the `produceitem` row itself (`ProductID`, `rootlotno`, `KG`→`StockPerKG`, `Carton`→`StockPerUnit`, `Place`→`StockPlace`, `LOTNO`→`StockID`) and (b) the sibling active STOCKTRX row for the same rootlotno/production for all cost fields (`Cost`, `InventoryCost`, `cycletime`, `smcost`, `LossKG`, `LGCost`, `PricePerCarton`, `SmCostperCarton`, `OtherPerCarton`, `SupplierID`, `SupplierName`, `CostID`, `CostDetailID`). `Remark` = the production's `PLOTNO` (from `produceHD.PLOTNO`), matching the sibling row's own `Remark` convention. This **will** create a negative `StockLotBalanceUnit` in that rootlotno's partition (since the original physical unit is single and its "real" consumption already happened via the sibling row) — immediately follow with the standard §17.4 2-step remediation (compensating `ADJUST STOCK IN`, backdated to one day before the earliest transaction in that specific `(rootlotno, StockPerKG, StockPlace)` partition, + override the new negative row's `StockLotBalanceUnit` to 0).

**Sub-pattern B — "cancel, no redo, physical unit consumed by a DIFFERENT later production" (1 of 5 cases, `PR290626-0021`)**: the STOCKTRX consumption was cancelled and genuinely never redone by the SAME production — but tracing the rootlotno's full history showed the physical unit went on to be legitimately consumed by an entirely different, later production chain (confirmed multiple further MOVE/PRODUCTION hops, still active through the present day). So there is no leftover physical unit available to draw against for this fix. **Fix** (owner's directive): INSERT a compensating `ADJUST STOCK IN` (+1 unit, sourcing cost fields from the production's own now-cancelled STOCKTRX row, which still has full cost data) **first**, dated to one day before the earliest transaction in that `(rootlotno, StockPerKG, StockPlace)` partition, then INSERT the active `PRODUCTION`/`OUT` row referencing the correct `produceitemid`, dated to the original attempted transaction date. Because the ADJUST STOCK supplies a fresh unit specifically for this partition/date range, this sub-pattern does **not** produce a negative balance and needs no further correction — verify with a plain balance check on that partition.

**How to tell which sub-pattern applies**: after finding the `produceitem` row's own STOCKTRX counterpart (via `produceitemid`) and confirming it's cancelled, trace that specific `StockID`'s (or its rootlotno's) full transaction history forward in time. If an active, non-cancelled consumption of the same rootlotno by the SAME production shows up nearby (same day, sibling `ProductionItemID`) → Sub-pattern A. If instead the unit's subsequent history shows it moving/being consumed under **other** `ProductionID`s with no such sibling for the production in question → Sub-pattern B.

### 17.9 Real cases fixed 2026-07-10 (all verified `pi.cnt = st.cnt` after fix)

| ProductionID | Before (pi/st) | Sub-pattern | Orphan `ProductionItemID` | rootlotno | Fix summary |
|---|---|---|---|---|---|
| `PR080626-0067` | 24/23 | A | `PI080626-0018` | `290126-0099` | insert active OUT + ADJUST STOCK IN (backdated one day before partition start) + balance override |
| `PR110626-0048` | 66/65 | A | `PI110626-0039` | `280526-0114` | same as above |
| `PR200526-0041` | 105/104 | A | `PI200526-0067` | `040326-0048` | same as above |
| `PR240626-0041` | 42/41 | A | `PI240626-0041` | `150526-0203` | same as above |
| `PR290626-0021` | 5/4 | B | `PI290626-0001` | `110526-0098` | ADJUST STOCK IN (fresh unit, real one already consumed by a later, different production chain) + active OUT — no negative balance, no balance-override step needed |

**Open item**: the full `GetIncompletedProduction` output list has not yet been retrieved/swept end-to-end — only 5 `ProductionID`s (supplied by the owner from a partial list) have been checked and fixed so far. A systemic pass over the complete flagged list (same as the open item in §17.5 for the separate double-draw pattern) has not yet been run.

### 17.10 True root cause found: the cancel logic lives in the VB.NET application layer, not SQL `[NEW 2026-07-10]`

§17.6–17.9 documented the *symptom* (produceitem/STOCKTRX row-count mismatches) and a *policy* for fixing it after the fact. This section documents the actual *cause*, found by reviewing owner-supplied VB.NET source.

**Two entry points, one shared implementation:**
- **Per-line cancel** — `frmXXX.BtnCANCEL_Click`. Branches on the row's `type` (`IN`/`OUT`/`MOVEOUT`/`MOVEIN`) and calls `objstock.CancelStockIN`, `objstock.CancelStockOUT`, or `objstock.CancelStockMove` accordingly. A special path exists for `txtObjective = "PRODUCTION"` guarded by a hardcoded passcode (`"8114"`) that calls `CancelAll(ProductionID)` — a separate, not-yet-reviewed routine for cancelling an entire production from this same button.
- **Whole-production unapprove** — `ProductionCls.UnApprove(productionid)`. Loops every STOCKTRX row for the production (`GetAllStockByProductionID`) and calls the exact same `objStock.CancelStockIN` / `objStock.CancelStockOUT` per row, then finally `Objproduction.SetUNApproved(id)` (which just flips `produceHD.Status` back to `'0'`).

**The bug, confirmed at the source**: `StockCls.CancelStockOUT` (and `CancelStockIN`, same shape) only ever: loads the target row, builds a reversal row (`StockINOUT = "CANCEL"`, negated `StockPerUnit`), `.insert()`s it into `stocktrx`, flips `Active` flags, updates `Product.BalanceQty`, and calls `UpdateCancel` (sets `Remark = 'CANCEL'` on the original row, cascading to any row whose `Remark` pointed at it). **No line in either function references `produceitem` in any way** — not a read, not a write. This is a direct, complete confirmation of the §17.7 hypothesis.

**Smoking-gun evidence of unfinished intent**: `UnApprove` declares, right at the top of the routine:
```vb
Dim ObjProduceItem As New ProduceItemCls
Dim ObjProduceItemList As New List(Of ProduceItemCls)
```
Neither variable is ever read or written anywhere else in the function. This is dead code left over from an evidently-planned-but-never-implemented produceitem-sync step — strong direct evidence (not just inference) that the original developer intended to keep `produceitem` in sync on cancel/unapprove and never finished it.

**Corroborating finding**: a search of `sys.sql_modules` for every reference to `GetPreCanceltrx` (the table-valued function that walks the downstream cancel-cascade chain, §17.11) found **zero stored procedures calling it**. It is invoked directly from the application layer (there is a matching `StockCls.GetPreCancelTrx` wrapper function in the VB.NET code). This is consistent with — though not fully proof of — the cancel/cascade decision logic living client-side rather than being centralized in SQL, which is itself a contributing factor to why a produceitem-sync fix was never centrally enforced.

### 17.11 `GetPreCanceltrx` in practice — verified against a real 53-row chain `[VERIFIED 2026-07-10]`

Ran `GetPreCanceltrx(rootlotno, stockrunning)` for a real StockRunning (`'01109344'`, a MOVEIN of rootlotno `290126-0105`) to see its actual behavior, not just its source (§17.6 already showed the function body). Result: **53 downstream rows**, spanning **six separate productions across 18 days** (08/06/2026 → 26/06/2026):

`PR080626-0067` → (MOVE) → `PR090626-0061` → (MOVE ×2) → `PR100626-0064` → (MOVE ×2) → `PR110626-0032` → (MOVE ×2, plus a real customer **SELL invoice `INV2026-06-D0636`** partway down one branch) → `PR260626-0011` → (MOVE) → a `SAMPLE` withdrawal.

This is a live demonstration of why a "confirm before cascading" design (§17.12–17.13) is not theoretical caution — a naive auto-cascade on this specific real row would have silently unwound a real sales invoice from 10 days prior.

**Confirmed chain-widening rule, read directly from the function body and cross-checked against this real result**:
- When the walk reaches a row where `ProjectID = 'PRODUCTION'` **and** `StockINOUT = 'OUT'` (i.e. a production consuming an input), the function widens to **every other line of that same ProductionID** — because a production can only be unapproved as a whole, never line-by-line (mirrors `UnApprove`'s own all-or-nothing loop, §17.10). This is confirmed by the real result: reaching `PR110626-0032` pulled in `BG1037` and `S040` (§17.13) even though neither is on the direct rootlotno-108 chain — they're simply other lines of the same production.
- Otherwise (any other `ProjectID`, or `StockINOUT = 'IN'`) the walk continues by **following `rootlotno`** — this is how the chain crosses from one production's *output* into the next production's *input* after a MOVE.
- The walk terminates at a rootlotno/StockID once no further non-cancelled transaction references it.

### 17.12 Cascade-cancel SP design (drafted, not yet deployed) `[REQUIRES SSMS — CREATE/THROW/TRY-CATCH blocked by mssql tool]`

Three-part replacement design, agreed with the owner (auto-cascade allowed, but only after explicit user confirmation of the full impact — never silent):

1. **`sp_CancelStockTrxLine(@StockRunning, @UserName)`** — single-line cancel, drop-in replacement for `CancelStockIN`/`CancelStockOUT`/`CancelStockMove`. Same guard checks as today (`IsCancelStock`, `IsCancelBuy` equivalents), same reversal-insert mechanism, **plus a new final step**: if the row's `ProjectID = 'PRODUCTION'` and it has a `produceitemid`, delete (with an audit-log insert first — table not yet created, open decision) the matching `produceitem` row in the same transaction. This is the fix for §17.6–17.10's root cause, applied at the point of cancellation rather than patched after the fact.
2. **`sp_PreviewCancelChain(@StockRunning)`** — read-only. Wraps `GetPreCanceltrx` and adds a human-readable `ActionDescription` per row (flags `SELL` and `PRODUCTION` rows specially) for the calling application to render to the user before anything is touched.
3. **`sp_ExecuteCancelChain(@StockRunning, @UserName, @Confirmed)`** — throws unless `@Confirmed = 1` (must come from the app only after the user has seen the `sp_PreviewCancelChain` output and explicitly agreed). Walks the same `GetPreCanceltrx` chain newest-first, calling `sp_CancelStockTrxLine` on each row, then the originally-requested row, all inside one transaction — all-or-nothing, so a mid-chain failure (e.g. an un-cancellable SELL row because a credit note already exists) rolls back the entire cascade rather than leaving a partial cancel.

**Open items, not yet resolved:**
- Whether deleting a `produceitem` row on cancel should go through an audit-log table first (`produceitem_cancelled_log`, not yet created) or simply add `CancelledDate`/`CancelledStockRunning` columns to `produceitem` itself in place.
- The `MOVEOUT`/`MOVEIN` branch of `sp_CancelStockTrxLine` is stubbed only — needs full porting from `StockCls.CancelStockMove`, which pairs MOVEOUT+MOVEIN and checks `IsContinueMove` before proceeding.
- `CancelAll(ProductionID)` (the passcode-gated whole-production cancel path from `BtnCANCEL_Click`) and `CancelSMRQ` (SM-request-specific cleanup for `OVERHEAD(SM)` objective) have not yet been reviewed or ported.
- `IsCancelSellStockout`, the SELL-specific guard, has not yet been folded into the new SP's guard section.

### 17.13 Cascade scope rule: RM/FG vs SM `[VERIFIED W/ OWNER 2026-07-10]`

Confirmed with the owner, and consistent with both the `GetPreCanceltrx` source (§17.6, §17.11) and the real 53-row trace: cascade scope differs by product category.

- **RM/FG**: carries lot genealogy forward. A production's finished-good output (a new rootlotno) can itself become a later production's raw-material input (after a MOVE re-locates it). Cancelling a production therefore requires **transitively unapproving every downstream production** that consumed any of its outputs, all the way to wherever the chain terminates (no further consumption found). This is the expensive, multi-hop case the cascade design (§17.12) exists for.
- **SM** (semi-finished/overhead, e.g. `CTN0501` packaging, `BG1037`, `S040` — verified `ProductCategory = 'SM'` for all three): a simple unit-count consumable (`StockPerKG = 1` always; no weight tracking; confirmed via `GetKGbyStockid`'s `WHEN 'SM' THEN '0'` branch). **No rootlotno genealogy** — an SM lot is only ever consumed, never re-produced as someone else's input. Cancelling a production that used an SM item only requires restoring that **specific SM lot's own balance** (via the same generic reversal-insert mechanism used for everything else — the mechanism itself is identical, since each SM lot/BUY still has its own tracked `StockID`/balance) — **no cascade to any other production**, even ones that separately drew from the exact same shared SM lot at a different time. Confirmed directly in `GetPreCanceltrx`'s logic: the chain only widens to "every other line of the same ProductionID" (§17.11) — it never widens *across* unrelated productions just because they share a bulk/SM lot.

**Practical rule of thumb for anyone (human or Claude) manually reasoning about a cancel's blast radius**: trace forward only through `StockINOUT = 'IN'` rows (or any `ProjectID` other than `PRODUCTION`+`OUT`) to find genealogy continuations; treat a `PRODUCTION`+`OUT` row on an SM-category product as a dead end requiring no further tracing, but still requiring that specific production to be included in the "must unapprove" set if it's already in scope for another reason (i.e. it's covered because the whole production is being unapproved anyway, not because the SM item itself has a chain).

### 17.14 Open item: cascade design not yet load-tested against a case requiring reversing an approved sale

The 53-row trace (§17.11) surfaced a real `SELL` row 30+ transactions deep in the chain, but the design has not yet been exercised end-to-end against that specific case (i.e. actually running `sp_PreviewCancelChain`/`sp_ExecuteCancelChain` once deployed, and confirming what happens when the SELL row itself can't legally be cancelled — e.g. an invoice with a customer-side credit note already issued, per the `IsCancelSellStockout`/PO-completion-style guards that exist for BUY but haven't been mirrored for SELL yet, §17.12). This should be validated once the SPs are deployed via SSMS, before this cascade path is used on a real cancel that reaches as deep as a completed sale.

### 17.15 Duplicate-MOVEOUT-within-a-move-batch pattern, and the real STOCKTRX CANCEL row structure `[NEW 2026-07-14]`

Diagnosed a live approval failure on `PR140726-0059` — `chkproduction` (§17.2 Step 1) showed all 5 of its own lines OK, but Step 2 found a pre-existing negative-balance row (`StockID 070726-0156`, `rootlotno 070726-0144`, `StockLotBalanceUnit=-1`, `Active=1`) blocking the post-insert guard.

Traced it to the same move document (`MV140726-0002`) as the production's own consumption: that rootlotno had 2 MOVEOUT rows but only 1 MOVEIN (every other rootlotno in the same 67-row move batch has exactly 1 MOVEOUT + 1 MOVEIN) — meaning one MOVEOUT was a genuine duplicate/erroneous transaction, not a real physical shortfall like the §17.3 pattern.

**New distinction established**: when a negative balance traces to a duplicate transaction with no real physical counterpart, the fix is to properly CANCEL the duplicate row using the exact mechanism the live application itself uses — NOT the §17.4 ADJUST STOCK IN + balance-override pattern (that pattern is reserved for cases where a real physical unit was genuinely double-drawn and a compensating unit must be fabricated to match reality).

**Real STOCKTRX CANCEL row structure** (reverse-engineered from 5 live historical examples, `ProjectID='MOVE'`, `StockINOUT='CANCEL'`):

- A **new row** is inserted: same `StockID` as the row being cancelled, `StockPerUnit` negated, `StockRef` = the cancelled row's own `StockRunning`, `Remark` = the cancelled row's own `StockRunning` (NOT the literal text `'CANCEL'`), its own `StockLotBalanceUnit` recomputed net (in this case, 0), `Active=0` on the new CANCEL row.
- The **original cancelled row** is UPDATEd: `Active=0`, `Remark='CANCEL'` (literal text, overwriting whatever the row's prior Remark was) — but its own `StockLotBalanceUnit` is left unchanged (confirmed from all 5 live examples). This differs from §17.4's balance-override pattern, which zeroes the original row's balance.

Applied live: cancelled `StockRunning 01141397` (the duplicate MOVEOUT) via new row `01141920`; confirmed the negative-balance guard check (§17.2 Step 2) returns 0 rows afterward.

**Also documented (owner-directed)**: `CostID`/`CostDetailID`/`InventoryCost` are only populated by the live application on the originating BUY/IN row of a rootlotno — every subsequent MOVEOUT/MOVEIN row the app itself generates leaves these NULL (confirmed on 2 genuine, untouched app-generated rows in the same chain — this is normal system behavior, not a bug). Owner's standing preference when Claude manually inserts/corrects STOCKTRX rows on a rootlotno: backfill `CostID`/`CostDetailID`/`InventoryCost` on all touched rows to match that rootlotno's original BUY/IN row, even though the live app itself doesn't do this for ordinary MOVE rows — a deliberate divergence for manually-corrected rows, for cost-traceability.

---

### 17.16 System-wide `StockLotBalanceUnit<0` sweep + `UpdatePickUpCutlot` root cause (AUTOSELL oversell) `[NEW 2026-07-20]`

**First-ever system-wide sweep** for the full anomaly pattern (§17.5 flagged this gap as unaddressed) was run this session:

```sql
SELECT StockID, rootlotno, ProductID, ProjectID, StockINOUT, StockLotBalanceUnit, Active, Remark, StockTrxDate
FROM stocktrx
WHERE StockLotBalanceUnit < 0 AND Active = 1 AND StockINOUT <> 'cancel' AND Remark <> 'cancel'
ORDER BY CONVERT(date, StockTrxDate, 103) DESC
```

**Result: 44 rows** — PRODUCTION (16), MOVE (16), SELL (10), ADJUST STOCK (1), SM(OVERHEAD) (1). Being remediated one row at a time (newest-first, owner-directed), 42 remain as of this write-up. `StockTrxDate` is stored as free-text `DD/MM/YYYY` (§4.1-adjacent trap already documented elsewhere in this KB) — a naive `ORDER BY StockTrxDate DESC` sorts alphabetically and gives a nonsense chronology; always wrap in `CONVERT(date, StockTrxDate, 103)`.

**Root cause of the SELL-category rows: `UpdatePickUpCutlot`.** This SP is the source of every STOCKTRX row with `ID1='AUTOSELL'` — it reads `pickuplotCut` (a transient staging table the app populates with `{StockRunning, CutQty, docno, deliverydate, rootlotno, Stockplace, ...}` per pick, no status/error column) and for each row:

```sql
StockLotBalanceUnit = StockLotBalanceUnit - cutqty   -- BUG: StockLotBalanceUnit here is the STALE snapshot
                                                        -- stored on whatever row pickuplotCut.StockRunning
                                                        -- happens to reference, not the lot's live balance
```

with **no check whatsoever** that `cutqty` is actually available. This is the direct SELL-side analog of the pre-2026-07-08 `UpdateApproveProduction` gap (§17.1) — except that gap was fixed on the production-approval side and never on this side. Confirmed root cause via a real example: `StockID 050126-0912` (rootlotno `050126-1117`) was fully consumed to 0 by 26/01/2026; on 20/07/2026 invoice `INV2026-07-D0785` needed 4 units of the same product — 2 correctly drew from a fresh same-product lot (`180726-0180`, 5→3) but the other 2 hit `StockRef='01135911'` (the exhausted lot's own very-first row, a stale reference from January) and drove `StockLotBalanceUnit` to -2.

**Fix deployed same day, 2 iterations** (both via `execute_write_query` + `CREATE OR ALTER PROCEDURE` directly — no SSMS needed, §14 tooling note corrected):

- **v1** (initial): compute live per-partition balance (`rootlotno`,`StockPerKG`,`StockPlace` — the same 3-column partition key as the canonical recompute query, §17.4-adjacent) via `SUM(StockPerUnit) WHERE StockINOUT<>'cancel' AND Remark<>'cancel'`; if a cut's cumulative in-batch demand would exceed it, **reject the row entirely** (delete from the temp table before insert, cut 0).
- **v2** (owner-directed correction — "ถ้า lot ไม่พอ ต้องตัดไม่เกิน", i.e. if the lot doesn't have enough, cut no more than what's actually there, NOT reject-all): same live-balance computation, but caps each cut instead of rejecting it —

```sql
-- per row, ordered by StockRunning within (rootlotno, StockPerKG, StockPlace):
--   CumInclusive  = running SUM(CutQty) including this row
--   CumExclusive  = CumInclusive - this row's own CutQty  (i.e. prior rows' running total)
ActualCutQty = CASE
    WHEN CumExclusive >= AvailableBalance THEN 0                        -- already exhausted before this row
    WHEN CumInclusive  <= AvailableBalance THEN RequestedCutQty         -- fully covered
    ELSE AvailableBalance - CumExclusive                                -- partial: exactly what's left
END
```

Rows with `ActualCutQty=0` are dropped before insert (no zero-quantity junk rows); rows with a partial fill use `ActualCutQty` in place of `cutqty` for both `StockPerUnit` and the balance subtraction. A `#PickupCutLotShortfall` result set (`StockRunning, RequestedCutQty, ActualCutQty, ShortfallQty`) is `SELECT`ed at the very end of the SP for any row that didn't get its full request. **Known gap, not yet resolved**: `pickuplotCut` has no status/error column and this SP doesn't write back to it — whether the calling VB.NET app actually reads this trailing result set is unconfirmed; until checked, a partial/rejected pick may fail silently from the staff's point of view (no error surfaced in the UI). Follow-up: review the app-layer caller of `UpdatePickUpCutlot`.

**Two worked remediation examples from the sweep, illustrating 2 different sub-patterns** (both distinct from the §17.3 double-draw-at-approval pattern and the §17.4 standard ADJUST-STOCK-IN-plus-override pattern):

1. **Wrong-lot AUTOSELL pick** (`StockID 050126-0912` / rootlotno `050126-1117`, detailed above). Fix: (a) INSERT a compensating `ADJUST STOCK IN` of +2 backdated to one day before the lot's earliest real transaction (zeroing the wrong lot back to net 0, matching physical reality — confirmed via a separate manual stock-check adjustment already on record for this exact StockPlace/product that the physical stock there was genuinely 0), (b) UPDATE the anomalous row's `StockLotBalanceUnit` to 0 directly (§17.4 Step 2 pattern), AND (c) — the new part — INSERT a second real `SELL OUT` of -2 against the lot that actually supplied the goods (`180726-0180`, 3→1), so the sale is now correctly attributed to its true source lot, not just ledger-non-negative. **Owner-established principle from this fix**: when a true source lot can be identified, prefer redirecting/re-attributing the sale to it over blindly padding the wrong lot with phantom compensating stock — matters for food-lot traceability, not just balance arithmetic.
2. **Duplicate MOVEOUT, no physical counterpart** (`StockID 060226-7185`, move reference `MV200726-0002` — same reference had 2 MOVEOUT rows but only 1 MOVEIN, mirroring the exact §17.15 pattern). Fixed via the standard §17.15 CANCEL mechanism (new row `StockINOUT='CANCEL'`, `StockPerUnit` negated, `Remark`=the cancelled row's own `StockRunning`; original row UPDATEd to `Remark='CANCEL'`, `Active=0`) — NOT ADJUST STOCK IN, since there was no real physical unit to compensate for.

---

### 17.16.1 v2→v3 incident: guard shipped with an unbound join, live outage `[NEW 2026-07-20]`

The v2 guard deployed earlier in §17.16 built `#LiveBalance` (live per-partition balance) but the `#PickupCutLotAlloc` subquery referenced `lb.LiveBalance` without ever joining `#LiveBalance AS lb` into its `FROM` clause. SQL Server's temp-table scoping meant this wasn't caught by deploy-time checks (`CREATE OR ALTER PROCEDURE` succeeds — the identifier is only resolved at execution time, not compile time, for this kind of construct) — the break only surfaced on the first real call, which failed hard with `Msg 4104: The multi-part identifier "lb.LiveBalance" could not be bound`, fully blocking AUTOSELL cuts rather than silently misbehaving.

Fixed by adding the missing join (v3, deployed 23:18:29):
```sql
FROM #PickupCutLotSTock p
LEFT JOIN pickuplotCut pc ON p.StockRunning = pc.StockRunning
LEFT JOIN #LiveBalance lb
    ON p.rootlotno = lb.rootlotno
   AND p.StockPerKG = lb.StockPerKG
   AND p.StockPlace = lb.StockPlace
```

**Standing lesson for all future `CREATE OR ALTER PROCEDURE` deploys via `execute_write_query`**: confirming `sys.objects.modify_date` advanced and re-reading `OBJECT_DEFINITION` only proves the new code was *saved* — it does NOT prove the code *runs* without error. A real end-to-end test execution (or at minimum, tracing every temp table referenced in the final SELECT/JOIN list actually appears in that block's own FROM/JOIN clauses) is required before reporting a deploy as verified.

## 18. NETWORK / NAS DNS REDIRECT — `mfit.dyndns.biz` SQL Server LAN Routing `[VERIFIED — resolved 2026-07-20]`

### 18.1 Problem

LAN clients connecting to SQL Server via the hostname `mfit.dyndns.biz` on port 1433 (this is the same hostname referenced in the KB header and in SSIS, §header) were being resolved to the NAS's IP (192.168.1.198) instead of the actual SQL Server host, "MF" (192.168.1.56) — causing the connection to fail. Clients connecting from **outside** the LAN via the same hostname succeeded normally. It was initially assumed the NAS (Synology) itself was doing the redirect; investigation showed this was not the case.

### 18.2 Devices involved

| Device | IP | Details |
|---|---|---|
| Router | 192.168.1.1 | Huawei HG8145X6 (combined ONT/Router), ISP 3BB Broadband, WAN public IP 171.6.135.173; also the LAN's DHCP + DNS server |
| NAS | 192.168.1.198 | Synology DSM, x86_64, Python 3.8.12 preinstalled, SSH user `adminmf`; no Docker/Container Manager, no Entware/opkg, no socat |
| MF (real SQL Server target) | 192.168.1.56 | SQL Server listening on TCP 1433 normally |
| Hostname | `mfit.dyndns.biz` | DDNS host, provider `dyndns-custom` via `members.dyndns.org`, username `scholesy` |

### 18.3 Root cause `[VERIFIED]`

A **Static DNS Configuration** record on the router (Network Application > DNS Configuration > Static DNS Configuration) maps `mfit.dyndns.biz` → `192.168.1.198` (the NAS) for any query answered by the router — i.e. every LAN client, since the router is the DHCP-assigned DNS server for the whole LAN. This record was set deliberately, for the NAS's other legitimate uses, but without accounting for port 1433 needing to reach MF instead. Since DNS cannot differentiate an answer by destination port, one hostname can only point one way per network — this is why the record "breaks" SQL Server access specifically while presumably serving its other intended purpose correctly.

**Why external access always worked**: external clients never query this router for DNS — they resolve `mfit.dyndns.biz` via public DNS, which only reflects the router's own DDNS client (Network Application > DDNS Configuration, keeps the public record pointed at the router's WAN IP, 171.6.135.173). The router's Port Mapping / Forward Rule "MF" (TCP 1433→192.168.1.56:1433, TCP 8080→192.168.1.56:80) then correctly routes external WAN traffic to MF. Same hostname, two completely different resolution paths.

**Ruled out during investigation** (in order checked): client-side hosts file, Port Mapping / Forward Rules (has no effect on DNS), router's own DDNS client, NAS DHCP hostname / NAS's own (separate, unconfigured) DDNS page, the router's entire Security tab (Firewall/IP Filter/MAC Filter/URL Filter/Parental Control/DoS/Device Access/WAN Access — none relate to DNS), UPnP (disabled, empty). Decisive proof the public record itself was fine: `nslookup mfit.dyndns.biz 8.8.8.8` (bypassing the router) correctly returned `171.6.135.173`, confirming the router itself was intercepting and answering LAN queries via the Static DNS record specifically.

### 18.4 Confirmed requirement `[VERIFIED W/ OWNER]`

`mfit.dyndns.biz` should keep defaulting to the NAS (192.168.1.198) for the NAS's other uses — the Static DNS record itself should NOT be changed — but port 1433 specifically must end up at MF (192.168.1.56). Since DNS can't do this per-port, the fix is a TCP-level proxy running on the NAS: listen on `:1433`, forward to `192.168.1.56:1433`.

### 18.5 Existing workaround found (why symptoms were intermittent) `[VERIFIED]`

While checking what NAS tools could implement a proxy (no socat, no Entware/opkg, no Docker, but Python 3.8.12 present), an inline `python3` proxy script was found **already running** on the NAS (`ps aux`, PID 11773, running as root, started 08:14 the same day) — listening on ports 1433 and 8080, forwarding to `192.168.1.56:1433` and `:80` respectively. This matches the requirement exactly: someone (presumably a prior administrator) had already solved this problem, but launched it manually over SSH without ever tying it to Task Scheduler or any auto-start mechanism. Every NAS reboot or dropped SSH session killed it, requiring a manual restart — explaining the "works sometimes, fails other times" pattern reported historically.

Verified live and working:
```
PS> Test-NetConnection mfit.dyndns.biz -Port 1433
RemoteAddress    : 192.168.1.198
RemotePort       : 1433
TcpTestSucceeded : True
```
(Note: that particular test run showed `SourceAddress 192.168.1.56`, meaning it was run from MF itself rather than a normal LAN client — still needs re-verification from an ordinary LAN client per §18.6.)

### 18.6 Current status & next steps (open items)

| # | Item | Status |
|---|---|---|
| 1 | Re-verify from an ordinary LAN client (not the MF machine itself) that access works generally | Not done |
| 2 | Save the proxy logic as a permanent file, `/volume1/homes/adminmf/scripts/tcp_proxy_1433.py` (pure Python socket+threading, no external libs needed — matches the NAS's available tooling), and a `start_proxy.sh` wrapper that `pgrep`-checks before launching | Script prepared, pending execution |
| 3 | Bind to DSM Task Scheduler: Triggered Task on Boot-up (root) + recurring health-check every 5 min (root), both running `start_proxy.sh` | Not done |
| 4 | Cut over from the ad-hoc process (PID 11773) to the persistent Task Scheduler version, then reboot the NAS once to confirm auto-start actually works | Not done |
| 5 | Identify who originally set up the Static DNS record and the ad-hoc proxy, to preserve institutional knowledge | Unknown |
| 6 | Security review: the proxy runs as root; SSH/FTP/HTTP/Telnet are all enabled across LAN/WiFi/WAN on the router — consider scoping down to what's actually needed | Not done (flagged for consideration) |

### 18.7 Command reference (appendix)

**Windows client (PowerShell/cmd):**
```powershell
nslookup mfit.dyndns.biz
nslookup mfit.dyndns.biz 8.8.8.8        # bypass the router, query public DNS directly
Test-NetConnection 192.168.1.56 -Port 1433    # MF directly
Test-NetConnection 192.168.1.198 -Port 1433   # NAS directly
Test-NetConnection mfit.dyndns.biz -Port 1433
```

**NAS (SSH, user `adminmf`):**
```bash
ps aux | grep -i python
sudo netstat -tlnp | grep 1433
sudo iptables -t nat -L -n -v      # errors: nat module not usable on this NAS
which socat; which python3; python3 --version
```

**Router menus (Huawei HG8145X6):**
```
Network Application > DNS Configuration > Static DNS Configuration   <-- root cause is here
Forward Rules > Port Mapping Configuration
Network Application > DDNS Configuration
```

*§18 sourced from a standalone investigation document (`NAS_Redirect_KnowledgeBase.md`, dated 2026-07-20) merged into this KB; full investigation timeline, Mermaid traffic-flow diagrams, and the complete `tcp_proxy_1433.py` script are preserved in that source document.*

---

## 19. LOT NUMBERING SYSTEM (rootlotno / stockid) `[VERIFIED + FIXED, deployed 2026-07-20 17:15]`

### 19.1 Format & shared suffix pool
- Lot format: `DDMMYY-<suffix>`. Historically suffix was always 4 digits (`-0001`..`-9999`); all 1.14M stocktrx rows were uniformly 11 chars before 2026-07-20.
- **The numeric suffix is a SHARED pool per DDMMYY prefix between `rootlotno` AND `stockid`** — every MOVE mints a new stockid from the same prefix pool as the original lot's date, months after production. This is why a prefix can exhaust 9999 long after its date (moves of April lots consumed April prefixes in July).
- stockid prefix always equals rootlotno prefix (MOVE keeps the original date prefix; verified in data).

### 19.2 Only 3 modules consume the suffix numerically (exhaustive scan, all 3 DBs)
`FINDNEWLOT` (main generator, called directly by the app — zero SP callers), `Create_MOVEINOUT` (2nd copy of generator logic), `UpdateFixMOVE` (3rd copy, MOVEIN-repair SP).
**Everything else** in inventory parses only the front 6 chars (DDMMYY) or exact-matches the whole string — verified for all FIFO/expiry/QC/QR/traceback/cost-recalc/approval SPs (19 modules use `substring()` on lot values; every one uses positions 1-6 only). MildFoods has zero lot-suffix code (`Create_invdetailBI` touches stocktrx cost only); STAGING has zero lot references. `CostVatDetail.lot` (`{ProductCode}-{DDMMYY}`) is built from **stocktrxdate**, NOT from rootlotno — unaffected by suffix format.

### 19.3 Incident 2026-07-20: prefix 200426 exhausted 9999
StockID `200426-9999` was minted 20/07/2026 via MOVE. Old generator logic (`right('0000'+cast(right(max_stockid,4)+1),4)`) would wrap the next id to `200426-0000` and then silently mint duplicates from the 2nd move onward. Other prefixes near the cap at the time: `060226` (7183), `161225` (5907), `181125` (5870). Highest rootlotno suffix ever minted directly in one day: 4462 (`090621`) — the pool pressure comes mostly from MOVEs, not direct lot creation.

### 19.4 Fix deployed (`fix_LotFormat_NaturalGrow_v2.sql`, all 3 SPs, 2026-07-20 17:15)
- Suffix grows naturally: `..., 9999, 10000, 10001, ...` — **no 8-digit zero-padding** (owner decision). Lots <=9999 keep the exact old 4-digit padded look; only overflow lots get longer (12+ chars).
- Generators now parse the suffix **numerically**: `MAX(TRY_CONVERT(int, SUBSTRING(id, CHARINDEX('-',id)+1, 10)))` over the whole prefix pool (`stocktrx.stockid` ∪ `produceitem.lotno` [+ rootlotno columns / `produceItemtmp` in `FINDNEWLOT`]), then pad to 4 digits iff next <= 9999. Format-agnostic → no cutover date needed, old/new widths mix freely within a prefix.
- ⚠ **Never reintroduce `RIGHT(id,4)` or string `MAX(stockid)` for suffix math** — string max picks `-9999` over `-10000` and `RIGHT(...,4)` wraps `10000` to `'0000'` (silent duplicate StockIDs).
- Bonus fixes in `UpdateFixMOVE` (pre-existing collision bugs): (a) old code took `max(stockid)` per-rootlotno only → could collide with other rootlotnos sharing the prefix; now scans the whole prefix pool. (b) all rows in one INSERT batch got the same +1 → now `+ row_number() over(partition by prefix)`.
- `chk_ChainTest_A.rootlotno` widened varchar(11) → varchar(20). All other lot-bearing columns were already >= nvarchar(20) (STOCKTRX.StockID is nvarchar(50)).
- Rollback script archived (`rollback_LotFormat_NaturalGrow_v2.sql`) with verbatim pre-fix definitions; note: once any 5-digit suffix exists in a prefix, rolling back re-breaks that prefix (old code reads `right('10000',4)='0000'`).

### 19.5 Post-deploy verification (same day)
213 five-digit stockids minted contiguously (`200426-10000`..`200426-10210`), zero duplicate stockids across rootlotnos in that day's transactions. Normal-day prefix (`200726`) still mints 4-digit format (`-0096`). QR-reading SPs (`GetStockProductQR`, `GetStockProductionQR`, `QRinventory`) confirmed unaffected: exact-match on the scanned string + `left(stockid,6)` QC join only.

### 19.6 Known trade-off + remaining app-side checks
- String sort of mixed-width suffixes is wrong (`'10000'` < `'2000'` as strings): affects only the final `rootlotno ASC` tiebreak in `CalculateSMLotCut`/`CalculatePickupLotCut` FIFO ordering, and only WITHIN a same-day prefix that overflowed 9999 → negligible (same manufacturing date). `FINDCOST`'s `row_number ... order by rootlotno asc` min-pick has the same benign quirk.
- `[OPEN QUESTION]` NOT yet verified (outside DB): VB.NET scan-textbox MaxLength / fixed-position string parsing when composing QR content; 1D barcode label width if used (overflow lots are 1 char longer than the historical 11).


## 21. MOVE-CATEGORY NEGATIVE-BALANCE ROOT CAUSE — `StockCls.GetStockAllQR` NON-DETERMINISTIC SUBQUERIES `[FOUND + FIX DELIVERED 2026-07-21, applied by owner — not independently re-verified]`

### 21.1 Question that triggered this
Owner asked whether the §17.16 `UpdatePickUpCutlot` fix (SELL/AUTOSELL path) also covers MOVE — it does not. §17.16 items #15, #16, #19, #20, #21, #22, #23 all shared one pattern: a lot fully consumed by production, then MOVEd again weeks or months later once the physical stock no longer existed. This section traces that pattern past the database into the actual VB.NET application source.

### 21.2 `Create_MOVEINOUT` (SQL Server SP) is dead code in practice
The 2026-07-08-guarded SP (blocks moving a lot that is INPUT to an unapproved production) is never actually called. Confirmed in `class\StockCls.vb`, `StockMove()`: the line `'ObjConStr.ExecuteQuery("exec CreateMOVEINOUT '" & Running & "','" & MoveDocno & "','" & Trxdate & "'")` is commented out. Real MOVE creation is 100% client-side.

### 21.3 Real MOVE creation path
`STOCKMOVETMP.vb` (`txtQR_KeyPress`, the QR-scan handler) → `StockCls.GetStockAllQR(qr, place)` to resolve the scanned label to a live stockplace/balance, checks `stocklotbalanceunit >= 1` inline (a guard genuinely exists — this is not a "no guard at all" bug, correcting an earlier same-session misstatement) → if it passes, calls `ObjStock.StockMove()` → `StockMoveOUT`/`StockMoveIN`, which builds `INSERT INTO stocktrx` directly in VB.NET. No stored procedure is involved anywhere in this real path.

### 21.4 The actual bug: non-deterministic `TOP 1` without `ORDER BY`
`StockCls.GetStockAllQR` (4 call sites across 2 query blocks) derives both `@rootlotno` and the filtering `stockperkg` via separate `SELECT TOP 1 ... FROM stocktrx WHERE stockid = @X` subqueries with **no `ORDER BY`**. When a StockID's lot has been repacked/re-produced multiple times (the repacking-cascade pattern seen throughout §17.16's worked examples — e.g. a single carton going 10kg→9.9kg→9.7kg→7.7kg→... across successive productions), SQL Server can return an arbitrary historical row instead of the current one. If the returned `rootlotno`/`stockperkg` snapshot happens to resolve to a *different, still-live* weight-partition than the one physically scanned, the `stocklotbalanceunit >= 1` guard checks the wrong partition and passes incorrectly — the move proceeds against a partition that is actually already exhausted. One block (the `MoveDestinationPlace = ""` branch) additionally had a stray `·` character corrupting the literal `stockplace`, which would throw a SQL syntax error if that branch ever executed.

### 21.5 Fix delivered (inline code handed to owner, not a file — owner applies/deploys)
For all 4 call sites: collapsed the two independent `TOP 1` subqueries into a single one so `@rootlotno` and a new `@stockperkg` variable always come from the *same* current-state row:
```sql
declare @rootlotno nvarchar(20), @stockperkg decimal(15,3)
select top 1 @rootlotno = rootlotno, @stockperkg = stockperkg
from stocktrx where stockid = '<Stockid>'
order by stockrunning desc
```
then replaced `stockperkg in (select top 1 stockperkg from stocktrx where stockid = ...)` with a direct `stockperkg = @stockperkg` comparison; changed the second query block's trailing `order by stockRunning asc` to `desc` (was picking the *oldest* matching row instead of the newest); removed the stray `·`. **Owner confirmed applying this fix 2026-07-21** (via chat — not independently re-read from source this session, so exact deployed form not re-verified).

### 21.6 Relationship to §17.16
This is the MOVE-path counterpart to §17.16's `UpdatePickUpCutlot` fix: that guards SELL/AUTOSELL at the database layer (a stored procedure), this guards MOVE at the application layer (client-side VB.NET, no SP in the real path). Together they close both known negative-balance-creation vectors found by the system-wide sweep. PRODUCTION-category sweep items (§17.16.1's Sub-pattern B and remaining `#24`+ items) may have a separate, not-yet-investigated root cause — do not assume this fix covers PRODUCTION-project negative balances.

## 20. COSTVATHD / COSTVATDETAIL RELATIONSHIP TO ADJUST STOCK REMEDIATION ROWS `[VERIFIED — 2026-07-20]`

### 20.1 Question raised
During the §17.16 sweep remediation, owner asked whether every `ADJUST STOCK IN` compensating row inserted this session affects `CostVatHD`/`CostVatDetail` (documented in §3.5 as the ★ REAL COST SOURCE for RMCost/InventoryCost). Investigated live against the 6 rows inserted so far (`StockRunning` `01146360`, `01146373`, `01146375`, `01146383`, `01146439`, `01146448` — rootlotnos `050126-1117`, `250626-0184`, `050626-0091`, `050626-0093`, `070526-0120`, `211224-0011`).

### 20.2 Finding: no impact
- **No trigger path exists.** `sys.triggers` on `dbo.stocktrx` shows exactly 2 triggers (`trg_stocktrx_rootlot_lock`, `BlockRootlotnoMultiProduct`) — both only read/write `dbo.RootLotProduct` (rootlotno↔productid locking). Neither references `CostVatHD` or `CostVatDetail` in any way. A plain `INSERT INTO stocktrx` cannot reach the cost tables through triggers.
- **`CostVatDetail.lot` is keyed off the PO/claim receiving date, not any STOCKTRX transaction date.** Confirmed format `{productid}-{DDMMYY}` (e.g. `S094-231224`, `DAC055_D-180626`) where the date is when that PO/claim (`PODocno` → `MF_PO.DocNO` or `ADJ-CLAI-*`) was received/adjusted — sample rows show `PurchasingType`/`MaterialType`/`RMCost`/`InventoryCost` all tied 1:1 to a specific purchasing document, not to arbitrary stock movements.
- **Direct check against all 6 touched rootlotno/productid pairs found zero matches** in `CostVatDetail` — none of the manually-inserted `ADJUST STOCK` rows correspond to any existing PO-receiving record, confirming these compensating rows sit entirely outside `CostVatDetail`'s domain (they're production/consumption-side corrections, not RM-receiving events).
- **Conclusion**: `CostVatHD`/`CostVatDetail` are populated exclusively by the PO-receiving/claim-adjustment write path — a separate system from generic `stocktrx` inserts. The existing cost-field-copy convention (§14, §17.15 — backfill `Cost`/`InventoryCost`/`CostID`/`CostDetailID` on manual STOCKTRX inserts from the nearest real row on the same rootlotno) is confirmed safe: it inherits a value the system already computed and stamped forward, it does not re-derive from or write back into `CostVatDetail`.

### 20.3 Open item (not yet investigated)
Not checked: whether any FG cost recompute process (e.g. the `CalculateNewCost`/`Create_Recal_Cal` cascade, §4) re-averages across *all* STOCKTRX rows on a rootlotno — including manually-inserted `ADJUST STOCK` rows — in a way that could shift a downstream computed cost indirectly, without ever touching `CostVatHD`/`CostVatDetail` directly. Owner's question was answered as asked (direct CostVatHD/CostVatDetail impact = none); this adjacent question is flagged for follow-up if raised again, not yet confirmed either way.


## 22. SWEEP COMPLETION — ITEMS #29-44, NEW REMARK CONVENTION, 5 NEW SUB-PATTERNS `[NEW 2026-07-21]`

### 22.1 Sweep status: complete
The system-wide `StockLotBalanceUnit<0` sweep (§17.16, 44 rows originally found 2026-07-20) is now **fully resolved** — all 44 items fixed, 0 remain. Items #1-28 were resolved across the prior part of this session and the one before it; items #29-44 (the final 16, including one out-of-order outlier already closed earlier, `#39`, a -2100 SM OVERHEAD row) were resolved in this session's second half, documented below.

### 22.2 New standing Remark convention for ADJUST STOCK IN rows `[OWNER-DIRECTED]`
All `ADJUST STOCK IN` rows Claude inserts now use `Remark = 'ADJ{YYMM}CL-{XXXXX}'` instead of a free-text reason:
- `YYMM` — year+month of the row's own `StockDate` (the backdated insert date, not today's date).
- `CL` — literal, marks the row as Claude-authored.
- `XXXXX` — 5-digit zero-padded running counter, scoped **only** to Claude-authored rows (identified via `InvoiceNo='CLAUDE AI'`), assigned in `StockRunning`/insert chronological order.

Applied retroactively: all 44 pre-existing Claude-authored rows (`StockRunning` `01136580`–`01146558`) were renamed in one bulk `UPDATE ... FROM stocktrx st JOIN (VALUES (...)) AS v(StockRunning, NewRemark) ON st.StockRunning = v.StockRunning`. Counter is now at `#00058` after this session's items #29-44 + retroactive rows. Going forward, every new ADJUST STOCK IN must check the current max counter first (`SELECT COUNT(*) FROM stocktrx WHERE InvoiceNo='CLAUDE AI'` or parse the highest existing `XXXXX`) before assigning the next number.

### 22.3 Sub-pattern: corruption baked into `produceitem` itself, propagates identically into `stocktrx`
Found 3 times this session (items #29, #32, #35): a production's `ProductID` (and sometimes `rootlotno`) field is corrupted at the moment of creation — a stray leading character (`\DOT053`, `3DSQ085`) or an unrelated borrowed number — and this exact corrupted value is written identically into **both** `produceitem.ProductID` and the corresponding `stocktrx` OUT row's `ProductID`. Confirmed via direct comparison: `produceitem`'s row for the same `ProductionItemID` carries the identical corrupted string, proving the corruption happened once, upstream of both writes (most likely a single non-deterministic lookup, same root cause family as §21's `GetStockAllQR`), not as separate independent errors in each table.

**Rule**: when this is confirmed (produceitem matches stocktrx's corruption exactly), the compensating `ADJUST STOCK IN` must **replicate the corrupted ProductID**, not silently clean it — the `RootLotProduct` trigger-enforced lock (§22.6) will reject a row using the "correct" clean ProductID if the rootlotno is already locked to the corrupted one from the original row.

### 22.4 Sub-pattern: `stocktrx`-only corruption, NOT matching `produceitem` — the one exception to "never alter produceitem"
Found once (item #36): unlike §22.3, `produceitem`'s own `rootlotno` field was **correct**, but the `stocktrx` OUT row's copy of it was wrong (an unrelated StockID number, borrowed from a totally different transaction). Fix required correcting `stocktrx`'s `rootlotno` (and `ProductID`, to satisfy the trigger lock already established on the correct rootlotno) to match `produceitem`'s authoritative value — the reverse direction from §22.3.

While diagnosing this, owner separately verified (via `product` master table lookup — zero matches; via whole-database search — the corrupted ProductID string appears exactly once ever, in this single row of `produceitem`) that the corrupted ProductID was demonstrably not a real product code. **Owner then explicitly directed correcting `produceitem.ProductID` too** (this session's only exception to the standing "produceitem is master, never altered" rule, §17.6) — justified as fixing a data-entry typo in one field, not altering the physical facts of what happened (`LOTNO`/`KG`/`Place`/`Type` all left untouched). This should be treated as a narrow, owner-approved exception requiring the same standard of evidence (master-table lookup showing zero matches + whole-database frequency check showing a one-off occurrence) before repeating, not a general license to edit produceitem.

### 22.5 Sub-pattern: duplicate-OUT within the same production event (§17.15 family, OUT not MOVEOUT)
Found once (item #34): two `stocktrx` OUT rows for the same production shared an identical timestamp, `CostID`, and `CostDetailID` — proof of a single duplicated write rather than two genuine physical draws (a genuine second physical lot would carry a different `CostID`/`CostDetailID`, tied to its own PO-receiving record). Both rows drew against the same single-unit MOVEIN; the earlier-`StockRunning` row correctly zeroed the balance, the later one went negative. Fixed with the standard §17.15 CANCEL mechanism (new row, `StockINOUT='CANCEL'`, `StockPerUnit` negated, `StockRef`/`Remark` = the cancelled row's own `StockRunning`, `Active=0`, `StockTrxDate` unchanged/not backdated; original row `Remark='CANCEL'`, `Active=0`) — **not** ADJUST STOCK IN, since no second physical unit ever existed.

### 22.6 `RootLotProduct` trigger lock confirmed enforced live (extends §20.2)
§20.2 previously found 2 triggers on `stocktrx` (`trg_stocktrx_rootlot_lock`, `BlockRootlotnoMultiProduct`) that only write to `RootLotProduct`, with no path to `CostVatHD`/`CostVatDetail`. This session additionally confirmed their **enforcement** behavior live (not just their existence): both fire `AFTER INSERT, UPDATE` and actively `RAISERROR` + `ROLLBACK TRANSACTION` if a write would associate a `rootlotno` already locked to one `productid` with a different `productid` (hit directly — an INSERT using a "corrected" clean ProductID against a rootlotno already locked to a corrupted one failed with `RAISERROR('Invalid: rootlotno already locked to another productid.')`, silently rolling back with no other symptom than the raised message). **Practical implication for all future manual STOCKTRX inserts/updates**: always check `RootLotProduct` (or just query any existing row for the target `rootlotno`) for the locked `productid` before writing, and match it exactly — don't assume the "obviously correct" spelling will be accepted.

### 22.7 Sub-pattern: multi-unit shortage (shortfall > 1)
Found once (item #37, `StockLotBalanceUnit=-2`): a rootlotno partition received only 1 real unit (1 MOVEIN) but had 3 genuine downstream draws against it (2 OUTs individually confirmed via `produceitem`, plus 1 further legitimate MOVEOUT whose own downstream MOVE chain later closed cleanly) — a cumulative shortfall of 2, not 1. Fixed with a single `ADJUST STOCK IN` row using `StockPerUnit=+2` (matching the exact shortfall count) rather than two separate +1 rows, backdated to one day before the *earliest* draw that first pushed the running balance negative (the second of the three draws, not the first, since the first draw alone was fully covered by the one real unit).

### 22.8 Sub-pattern: `Remark='CANCEL'` on an IN row, never cleared — a silent self-exclusion
Found once (item #43): `UpdateFixRootlotno`'s balance formula filters `WHERE StockINOUT<>'cancel' AND remark<>'cancel'`. An IN row can be created with the app's provisional `Remark='CANCEL'` default and never get its Remark cleared/confirmed by any follow-up step — this silently and permanently excludes it from the running balance, indistinguishable from a row that was genuinely cancelled, even though no matching `CANCEL`-type row ever formally reversed it. A later, genuine OUT event (in this case a real physical disposal — goods returned by a customer for failing quality control, confirmed by its own descriptive Thai-language remark) can then be recorded referencing this "phantom" IN as its source, producing a negative balance identical in shape to the standard broken-StockRef genuine-shortage pattern (§17.4) but with a different underlying cause (self-excluding Remark vs. a StockRef pointing to nothing). Fixed identically to the standard pattern (ADJUST STOCK IN, backdated one day before the OUT) since the practical effect — no real balance contribution from the nominal source row — is the same either way.

### 22.9 Sub-pattern: anachronistic (chronologically-impossible) StockRef
Found once (item #44, 2017-era legacy data — the oldest row in the entire sweep): a row's `StockRef` pointed to a `StockRunning` that genuinely exists in the table, but is dated **three years later** than the referencing row itself (`StockRef` row dated 24/02/2020, referencing row dated 28/03/2017) — impossible under any real causal sequence, almost certainly an artifact of a historical data migration or StockRunning renumbering event that predates this KB's coverage. The referenced row was, separately, also cancelled (`Remark='CANCEL'`), so it contributes zero balance regardless of the date anomaly. Functionally identical to §17.4's standard broken/non-existent-reference genuine-shortage pattern for remediation purposes — fixed the same way (ADJUST STOCK IN, backdated one day before the referencing row). Flagged here as a distinct discovery because the *reference itself is not broken* (it resolves to a real row) — the anomaly is temporal, a subtler variant worth recognizing if it recurs in other pre-2020 legacy data.

---

*Document compiled from live SQL Server queries (via MCP database connector) and direct source code inspection (SSIS `Package.dtsx`, VB.NET `AutoEmail` project, and owner-supplied VB.NET `StockCls`/`ProductionCls`/button-click handlers) conducted 2026-07-03/04, with MCP/KB-distribution architecture work added 2026-07-07, a major cost/cycletime/loss/DL SP audit added 2026-07-09, production-approval/negative-balance remediation work (§17.1–17.5) added 2026-07-09, `GetIncompletedProduction`/produceitem-STOCKTRX row-count remediation work (§17.6–17.9) added 2026-07-10, and the VB.NET root-cause trace plus cascade-cancel SP design (§17.10–17.14) added 2026-07-10, a live diagnosis pass on `Create_EmpAllocate`/`CreateBackUpProfitloss`/`UpDateRecalLoss` added 2026-07-13, the duplicate-MOVEOUT/STOCKTRX-CANCEL-row-structure finding (§17.15) added 2026-07-14, the `mfit.dyndns.biz` LAN/WAN DNS-routing investigation (§18) added 2026-07-20, and an SSMS SSL/TLS client-connection troubleshooting note (§16.5) merged in from a parallel same-day session as v5.6, and the lot-numbering suffix-overflow incident + natural-grow fix (§19) added 2026-07-20 as v5.7, the `UpdatePickUpCutlot` root-cause fix + system-wide negative-balance sweep (§17.16) added 2026-07-20 as v5.8, the CostVatHD/CostVatDetail impact investigation (§20) added 2026-07-20 as v5.9, and the §17.16.1 v2→v3 unbound-join production outage + fix added 2026-07-20 as v5.10, and the §21 MOVE-category root-cause investigation (`StockCls.GetStockAllQR` non-deterministic TOP 1 bug, fix delivered to owner) added 2026-07-21 as v5.11, and the §22 sweep-completion writeup (items #29-44 closed, new ADJUST STOCK Remark convention, 5 new sub-patterns, `RootLotProduct` trigger enforcement confirmed live) added 2026-07-21 as v5.12. This markdown file is the single source of truth (v5.12) — the parallel `.docx` version was discontinued 2026-07-04. Treat any fact not explicitly tagged `[VERIFIED]` or `[VERIFIED W/ OWNER]` with appropriate caution.*
## 23. SESSION 2026-07-21 — SECOND SWEEP PASS, COST-FIELD AUDIT, `Create_MOVEINOUT` PERMANENT FIX `[NEW 2026-07-21]`

> เพิ่มต่อจาก §22. เนื้อหานี้เป็นงานของ session 2026-07-21 (รอบที่สอง) หลัง §22 ปิด sweep เดิมไปแล้ว
> ADJUST counter (`InvoiceNo='CLAUDE AI'`, `Remark LIKE 'ADJ%CL-%'`) เดินถึง **`ADJ2607CL-00060`** ณ สิ้น session นี้

### 23.1 หลักการที่ยืนยันซ้ำ: sweep เป็น "ต่อเนื่อง" ไม่ใช่ "ครั้งเดียวจบ"

การ re-run sweep (`StockLotBalanceUnit<0 AND Active=1`) และ `GetIncompletedProduction` วันนี้ **เจอของใหม่เพิ่ม** แม้ §22 เพิ่งปิด 44 รายการไปเมื่อ 20/07 นี่คือพฤติกรรมปกติของ live system: ตราบใดที่ root cause ยังไม่ถูกปิดที่ต้นตอ 100% หรือมี pattern ที่ยังไม่เคยตรวจ ธุรกรรมใหม่ทุกวันจะสร้างของใหม่ได้เรื่อย ๆ — **ไม่ได้แปลว่าที่แก้ไปแล้วผิด** ควร re-run sweep เป็นระยะ ไม่ใช่ถือว่าปิดถาวรหลังรอบเดียว

### 23.2 `GetIncompletedProduction` full sweep — 3 เคสใหม่, พบ 2 sub-pattern เพิ่ม

Re-run query เต็ม (§17.6) เจอ 3 `ProductionID` ค้าง:

| ProductionID | pi/st | ชนิด | rootlotno | วิธีแก้ |
|---|---|---|---|---|
| `PR081025-0001` | 8/7 | **Sub-pattern D (ใหม่)** | `040825-0440` | un-cancel แถวเดิม + §17.4 |
| `PR150726-0050` | 2/1 | A (cancel, both draws cancelled) | `050626-0091` | insert active OUT กลับ lot ที่ produceitem ระบุ |
| `PR150726-0053` | 2/**3** | **st>pi (ใหม่)** | `050626-0093` | cancel duplicate ที่ Claude เคย insert ผิด lot |

#### 23.2.1 Sub-pattern D (ใหม่) — "cancel line ทีหลังนาน ๆ โดยไม่มี redo, ของไม่ถูกใช้ที่อื่น"

`PR081025-0001` (ต.ค. 2025) ถูกกด cancel ที่ STOCKTRX line `00882270` (input `PI081025-0006`, LOTNO `040825-1252`) **เมื่อไม่นานมานี้** (reversal row `01147379` มี StockRunning สูงเทียบเท่ากิจกรรม ก.ค. 2026 แค่ backdate `StockTrxDate` ให้ตรงของเดิม) — production ปิดจบไป 9 เดือน ไม่มี redo และ trace แล้วไม่มีใครเอา unit นี้ไปใช้ต่อ = คนกด cancel line เก่าโดยไม่ได้ตั้งใจ (อาการสด ๆ ของ cancel-sync bug §17.10)

**วิธีแก้ (owner-directed): un-cancel — คืนสถานะแถวเดิม ไม่ใช่ ADJUST STOCK.** เมื่อสาเหตุคือการกด cancel พลาด วิธีที่ถูกคือย้อน cancel กลับ ไม่ใช่ยัด phantom stock ทับ history ที่ควรสมบูรณ์อยู่แล้ว:
```sql
UPDATE stocktrx SET Active=1, Remark='<PLOTNO เดิม>', InvoiceNo='CLAUDE AI'
WHERE StockRunning='00882270'   -- Remark กลับเป็น 'C081025-1' (ค่าเดิมก่อนโดน cancel)
```
การ un-cancel นี้ทำให้ `GetIncompletedProduction` ตรง (8=8) ทันที **แต่เผยให้เห็น negative balance เดิมที่ซ่อนอยู่** (double-draw ยุคก่อน guard 07-2025) จึงตามด้วย §17.4 มาตรฐาน: insert ADJUST STOCK IN +1 (`01147447`, `ADJ2607CL-00059`, backdate 1 วันก่อนธุรกรรมแรกใน partition) + override negative row เป็น 0

#### 23.2.2 กรณี `st_cnt > pi_cnt` (กลับด้าน, ไม่เคยเจอใน §17.8) — เกิดจาก Claude เอง

`PR150726-0053`: st มากกว่า pi (3 vs 2) ตรวจแล้ว **ไม่ใช่ app bug** — แถวเกิน (`01146384`, lot `250626-0192`) มี `InvoiceNo='CLAUDE AI'`, `ID1='Manual'` = **Claude session ก่อนหน้า insert ผิด lot** (produceitem ระบุ `050626-0093` แต่เผลอเบิกจาก `250626-0192`) วิธีแก้: cancel แถวเกินด้วยกลไก §17.15 (reversal `01147449`)
**บทเรียน:** ก่อน insert active PRODUCTION OUT ต้องยึด `produceitem` (master) เป็นตัวระบุ lot เสมอ ห้ามเลือก lot อื่นที่แค่ share ProductID

### 23.3 Cost-field consistency audit — anti-pattern "fake CostVatDetail placeholder"

Owner ตั้งเกณฑ์: ภายใน rootlotno เดียวกัน field `Cost/smcost/cycletime/CostID/CostDetailID/InventoryCost/DLThbKg/LGCost/LossKG` **ต้องเหมือนกัน** (ยกเว้นข้อยกเว้นที่ระบบทำเองตาม §17.15: MOVE ปล่อย `InventoryCost/CostID/CostDetailID` เป็น null, SELL ปล่อย `LGCost` null — ปกติ ไม่ใช่ error)

Scan Claude-authored rows ทั้งหมดเทียบกับแถวจริงใน 각 rootlotno เจอ **2 error จริง** (ที่เหลือเป็น false positive จาก null-by-design ข้างต้น + PO batch ที่ lot ละแถวเดียวไม่มีตัวเทียบ):

| StockRunning | rootlotno | ปัญหา | แก้เป็น |
|---|---|---|---|
| `01136585` | `100126-0201` | `InventoryCost=null`, ผูก `CostID=4242` | `InventoryCost=410`, `CostID/CostDetailID=null` |
| `01146735` | `100622-0134` | `InventoryCost=null`, ผูก `CostID=4240` | `InventoryCost=160`, `CostID/CostDetailID=null` |

**สาเหตุ (anti-pattern จาก Claude session ยุคแรก, counter #00005/#00047):** สร้าง `CostVatDetail` record ปลอมขึ้นใหม่โดยใส่ `RMCost=0`/`InventoryCost=0` แล้วผูก `CostID`/`CostDetailID` ไปยัง record ปลอมนั้น แทนที่จะทำตามมาตรฐานปัจจุบัน (§20): **ADJUST STOCK ไม่ผูก CostVatDetail เลย — ปล่อย `CostID/CostDetailID=null` แล้ว copy `InventoryCost` จากแถว BUY/PRODUCTION จริงใน rootlotno เดียวกัน** record ปลอมที่เหลือ (`CostVatID 19310`, `19323`) กลายเป็น orphan ไม่ต้องลบก็ได้ ไม่กระทบอะไร

**มาตรฐานที่ยืนยัน:** เมื่อ Claude insert/แก้ ADJUST STOCK ห้ามสร้าง CostVatDetail ใหม่ ให้ `CostID/CostDetailID=NULL` และ copy cost fields ที่เหลือจากแถวจริง (non-cancel, non-ADJUST) ใน rootlotno เดียวกัน

### 23.4 พฤติกรรมปกติ (ไม่ใช่ error): `Cost=0` ของ production ที่เพิ่ง approve

Scan rootlotno ทั้งระบบที่ `Cost` ไม่ตรงกันภายในตัวเอง เจอ ~44 lot แต่ **ทุกแถวที่ `Cost=0` มี StockDate อยู่ในช่วง 18–21/07/2026 เท่านั้น** (แถวเก่ากว่านั้นมี Cost ครบ) = production ที่เพิ่ง approve จะได้ `Cost=0` ชั่วคราว รอ nightly recalc (`UpdateRecalCost`/`CalculateNewCost`, §4/§12) มา backfill — **ไม่ใช่ error ไม่ต้องแก้** อาจ lag 1–3 วันสำหรับ lot ที่ยัง active ต่อเนื่อง

### 23.5 Negative-balance sweep — 2 เคสใหม่, 1 pattern ใหม่

| StockID | rootlotno | ชนิด | สาเหตุ | วิธีแก้ |
|---|---|---|---|---|
| `200726-0201` | `200726-0124` | MOVE duplicate | double-scan (§23.6) | cancel คู่ซ้ำ §17.15 |
| `180726-0212` | `180726-0212` | SELL | **late-invoice (ใหม่, §23.7)** | ADJUST STOCK IN + override |

#### 23.6 MOVE double-scan — root cause ของ `200726-0124`

Move batch `MV210726-0004` (DSH116) ทุก rootlotno มี MOVEOUT+MOVEIN คู่เดียว **ยกเว้น** `200726-0124` ที่มีสองคู่ซ้อน insert จริง (`StockDate`/`StockTime`) ต่างกัน **8 นาที** (14:33 และ 14:41) ทั้งคู่ผ่าน `ID1='Scanner'` อ้าง `StockRef=01146831` (BUY เดียวกัน) — ล็อตมีของ 1 หน่วย ถูกสแกนย้ายซ้ำ → คู่ที่สองไม่มีของจริง → balance -1

**สำคัญ: ไม่ใช่ bug เดียวกับ StockID-generation fix ที่ deploy 20/07.** fix นั้น (`Create_MOVEINOUT`, comment ในตัว SP ระบุ "แก้เฉพาะ block สร้าง @newStockID") แก้เรื่อง suffix โต/string-sort คนละเรื่องกับ double-scan โดยสิ้นเชิง `Create_MOVEINOUT` เดิม **ไม่มี** applock / idempotency re-check / post-insert guard เลย จึงกันสแกนซ้ำไม่ได้ (ต่างจาก `UpdateApproveProduction` §17.1 ที่มีครบ) — ดู fix ถาวร §23.8

#### 23.7 SELL late-invoice pattern (ใหม่) — root cause ของ `180726-0212`

invoice `INV2026-07-D0757` ลงวันที่ 13/07/2026 แต่ AUTOSELL cut จริงเมื่อ 18/07/2026 (ช้า 5 วัน) ตอน cut ไปเจอ lot ที่เพิ่งผลิตสด (`180726-0212`, produced 18/07) มาเติม แต่ระบบ stamp `StockTrxDate` = วันที่ในใบ invoice (13/07) **ไม่ใช่วันที่ cut จริง (18/07)** เมื่อ sort ตาม `CONVERT(date,StockTrxDate,103)` จึงดูเหมือนขายก่อนผลิต (ทั้งที่ StockRunning จริงคือผลิต `01147293` ก่อนขาย `01147304`) — **ของไม่ได้ขาดจริง แค่ timestamp สลับ**

ต่างจาก §17.16 (AUTOSELL wrong-lot / oversell) และ §21 (MOVE non-deterministic subquery) — เป็น code path ใหม่ที่ยังไม่เคยตรวจ: การขายที่ fulfill ช้ากว่าวันที่ในใบสั่ง
**วิธีแก้ (มาตรฐาน §17.4):** ADJUST STOCK IN +1 (`01147452`, `ADJ2607CL-00060`, backdate 12/07 = ก่อนวันที่ invoice) + override negative row เป็น 0 เพื่อปิด sweep ให้ตรงมาตรฐานเดียวกัน
**Follow-up (ยังไม่แก้):** ควรพิจารณาให้ AUTOSELL stamp `StockTrxDate` = วันที่ cut จริง ไม่ใช่วันที่ invoice — เพื่อไม่ให้ chronology เพี้ยนตั้งแต่ต้นทาง

### 23.8 `Create_MOVEINOUT` GUARD v2 — permanent fix สำหรับ double-scan `[DELIVERED, REQUIRES SSMS]`

แก้ที่ต้นตอโดยเพิ่ม guard 3 ชั้นใน `Create_MOVEINOUT` **ไม่แตะ logic การย้ายเดิม** (ไฟล์ `Create_MOVEINOUT_guard_v2.sql`):

1. **`sp_getapplock` (Exclusive บน rootlotno) + `BEGIN TRAN`** — serialize คำสั่งย้ายบน lot เดียวกัน กัน race/call ซ้อน (`@LockTimeout=15000`)
2. **Idempotency pre-check** — หลังได้ lock อ่านสถานะต้นทางสด ๆ ถ้า `active<>1` หรือ `balance<=0` (เคยย้ายไปแล้ว) → `THROW 51012` **ชั้นนี้จับ double-scan ห่าง 8 นาทีได้ตรง ๆ** เพราะรอบสองเห็นว่าต้นทางถูก consume แล้ว
3. **Post-insert negative-balance guard** — หลัง recompute ถ้ามีแถวติดลบใน lot → `ROLLBACK` + `THROW 51014` (safety net สุดท้าย ครอบทุกกรณี)

ครอบด้วย `TRY/CATCH` (ปิด cursor + rollback + `THROW` เดิมกลับให้ VB.NET) และแก้ `drop table #tmp` เดิมเป็น `if object_id(...) is not null drop` (เดิม error รอบแรก)

**Deploy:** SSMS เท่านั้น (`CREATE OR ALTER`+`THROW` ถูก block ผ่าน MCP tool) — มี rollback script `Create_MOVEINOUT_ROLLBACK_to_20250720.sql` (คืนเวอร์ชัน 07-20 ที่ยังเก็บ StockID fix; **ไม่ใช่** `Create_MOVEINOUT_backup` 07-08 ที่เก่ากว่าและจะทำให้เสีย StockID fix)

**Latent bug ที่สังเกตเห็น (ยังไม่แก้):** ในตัว SP เดิม `select * into #mvOutTmp ... where StockRunning = @stockrunningTrx` อ้าง `@stockrunningTrx` ที่ถูก declare แต่ไม่เคย set (= NULL) — น่าจะควรเป็น `@stockrunning` ยังไม่แตะเพราะ move ทำงานได้จริงในการใช้งาน (อาจมี path อื่นที่ set ค่า) ควรตรวจกับ owner ก่อนแก้

### 23.9 Open items ที่เกิดจาก session นี้

- **`Create_MOVEINOUT_guard_v2.sql` ยังไม่ deploy** — รอ SSMS + test (สแกนย้ายปกติ 1–2 รายการ แล้วลองสแกนซ้ำ lot เดิมว่าโดน block พร้อม Thai error)
- **AUTOSELL late-invoice** (§23.7) — พิจารณาแก้ให้ stamp `StockTrxDate` = วันที่ cut จริง
- **`@stockrunningTrx` NULL latent bug** (§23.8) — ตรวจกับ owner
- **KB_History ตกรุ่น** — SQL copy (`kb_id=1`) sync ครั้งสุดท้าย 2026-07-07 ขณะที่ไฟล์ `.md` เป็น v5.12+ ควร re-sync ให้ตรง
