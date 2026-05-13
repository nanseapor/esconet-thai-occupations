# TSCO ↔ ISCO-08 Bridge — via ILO ISCO-88 Crosswalk

**Built:** 2026-05-13 for CHANCEDEE V3 (CD3-3 CR session)

**Purpose:** Structural bridge from Thailand's TSCO 2544 classification (ISCO-88-based) to ISCO-08 — without LLM. Enables joining TSCO codes to existing ESCONET pipeline (ISCO-08 → ESCO → SOC → WASK).

---

## Source

- **ILO XLS:** `corrtab88-08.xls` (115 KB, 672 rows, 6 cols) — official ISCO-88 → ISCO-08 correspondence table from ILO Bureau of Statistics
- **Acquisition:** ILO direct URLs (`ilo.org`, `webapps.ilo.org`) return 403 for automated fetch. Used `Guidowe/occupationcross` R package on GitHub which mirrors the source file in `data-raw/cross/`.
- **TSCO source:** existing `THAI/tsco.parquet` (TSCO 2544, B.E. 2544 = A.D. 2001, กรมการจัดหางาน)
- **ISCO-08 Thai labels:** existing `THAI/isco08.parquet` (National Statistical Office Thailand)

---

## Files

| File | Purpose | Rows | Cols |
|---|---|---|---|
| `isco88_to_isco08.parquet` | Raw ILO crosswalk — ISCO-88 ↔ ISCO-08 at Unit level | 671 | 6 |
| `tsco_unit_to_isco08.parquet` | TSCO Unit → ISCO-08 Unit with Major filter status flag | 673 | 11 |
| `tsco_occupation_to_isco08_clean.parquet` | TSCO Occupation → ISCO-08 Unit, **clean 1-to-1 subset only** | 1,185 | 10 |

---

## Schema details

### `isco88_to_isco08.parquet` — ILO source crosswalk

| Column | Type | Notes |
|---|---|---|
| `isco88_title` | string | English title from ISCO-88 |
| `isco88_code` | string | 4-digit code (zero-padded) |
| `isco08_code` | string | 4-digit code |
| `partial` | string | `'p'` if the ISCO-88 code splits into multiple ISCO-08 codes; empty otherwise |
| `isco08_title` | string | English title from ISCO-08 |
| `comments` | string | ILO comments (mostly null) |

Stats: 389 unique ISCO-88 codes · 436 unique ISCO-08 codes · 221 are 1-to-1 · 168 are 1-to-many (max fan-out 11) · 356 rows flagged `'p'`

### `tsco_unit_to_isco08.parquet` — TSCO Unit normalized bridge

| Column | Type | Notes |
|---|---|---|
| `tsco_code` | string | TSCO Unit 4-digit code |
| `tsco_name_th` | string | TSCO Thai name |
| `tsco_major` | string | TSCO Major code (1 digit, from `SUBSTR(tsco_code,1,1)`) |
| `isco08_code` | string | Bridged ISCO-08 code (null if no bridge) |
| `isco08_name_th` | string | ISCO-08 Thai label (null if no bridge) |
| `isco08_major` | string | ISCO-08 Major code |
| `partial` | string | From ILO crosswalk |
| `isco08_title_en` | string | ISCO-08 English |
| `isco88_title_en` | string | ISCO-88 English (anchor) |
| `major_match` | boolean | True if `tsco_major == isco08_major` |
| `status` | string | `'kept'` / `'filtered'` / `'no_bridge'` |

**`status` semantics:**
- `kept` (521 rows · 78%) — bridge passes Major filter, safe to use
- `filtered` (147 rows · 22%) — bridge exists but TSCO Major ≠ ISCO-08 Major (drop to prevent dilution)
- `no_bridge` (5 rows) — TSCO Unit has no ISCO-88 equivalent in ILO (Thailand-specific)

### `tsco_occupation_to_isco08_clean.parquet` — TSCO Occupation clean mapping

Only includes TSCO Occupations whose parent TSCO Unit bridges 1-to-1 to a single ISCO-08 Unit (after Major filter).

| Column | Type | Notes |
|---|---|---|
| `tsco_occ_code` | string | TSCO Occupation code (NNNN.NN format) |
| `tsco_occ_name_th` | string | TSCO Occupation Thai name |
| `tsco_occ_name_en` | string | TSCO Occupation English name (may be empty) |
| `tsco_occ_description` | string | TSCO source description (Thai, ~427 char avg) |
| `tsco_unit_code` | string | Parent TSCO Unit code |
| `tsco_unit_name_th` | string | Parent TSCO Unit Thai name |
| `tsco_major` | string | TSCO Major (1 digit) |
| `isco08_code` | string | Inherited ISCO-08 code (single, from parent's 1-to-1 bridge) |
| `isco08_name_th` | string | ISCO-08 Thai label |
| `isco08_title_en` | string | ISCO-08 English title |

**Coverage:** 1,185 of 1,708 TSCO Occupations (69.4%) — the subset where structural inheritance gives unambiguous mapping. Remaining 523 Occupations either inherit 2+ ISCO-08 candidates (need LLM narrowing) or have no parent bridge.

---

## Method

```
TSCO Unit code (4-digit)
  ≡ ISCO-88 Unit code  (TSCO 2544 IS ISCO-88-based at Unit level)
  → ILO crosswalk (671 pairs)
  → ISCO-08 Unit code(s)
  → filter where SUBSTR(tsco_code,1,1) == SUBSTR(isco08_code,1,1)
  → kept / filtered / no_bridge
```

**Major filter rationale:** ISCO restructured 1988→2008 with meaningful Major reassignments
(e.g., agricultural "general managers" ISCO-88 1311 → ISCO-08 6xxx workers; food vendors
ISCO-88 9111 → ISCO-08 5212 services). Without filter, bridging TSCO Major 1 manager
could leak into TSCO Major 6 worker WASK profiles. With filter, these zero out as
genuine semantic loss.

**Inheritance for Occupations:** TSCO Occupation 2131.60 (วิศวกรซอฟต์แวร์)
inherits parent TSCO Unit 2131's bridge. Clean subset uses only parents that resolved
to a single ISCO-08 — guarantees unambiguous Thai→ISCO-08 mapping.

---

## Coverage

### `tsco_unit_to_isco08.parquet`

| Metric | Value |
|---|---|
| TSCO Units total | 393 |
| With bridge in ILO | 388 (98.7%) |
| **Kept after Major filter** | **365 (93%)** |
| Zeroed by filter (Major reassigned) | 23 |
| No bridge (Thailand-specific) | 5 |
| 1-to-1 after filter | 276 / 365 (76%) |
| Distinct ISCO-08 reached after filter | 400 / 436 |

### `tsco_occupation_to_isco08_clean.parquet`

| Metric | Value |
|---|---|
| TSCO Occupations in clean subset | 1,185 / 1,708 (69.4%) |
| TSCO Units parents (1-to-1) | 269 |
| ISCO-08 Units gaining Thai labels | 232 / 436 |
| Avg Thai labels per ISCO-08 Unit gained | 5.1 |
| ISCO-08 Units gaining 16+ Thai labels | 7 (chemistry, glass, metals, textiles, jewelry plants) |

---

## Known gaps / what's deferred

1. **523 TSCO Occupations (30.6%) not in clean subset** — under parents that bridge 1-to-many. Resolving requires LLM narrowing of TSCO Thai name → specific ISCO-08 Unit. Deferred to Phase 2.

2. **5 Thailand-specific TSCO Units** with no ILO bridge:
   - `1144` ผู้บริหารองค์กรที่ทำหน้าที่ตรวจสอบตามรัฐธรรมนูญ (constitutional commissioners)
   - `1150` เจ้าหน้าที่ของรัฐ (government officials, generic)
   - `1238` ผู้จัดการฝ่ายเทคโนโลยีสารสนเทศ (IT manager — TSCO had this in 2001 before ISCO-88 had a code; ISCO-08 added 1330 in 2008)
   - `3139` เจ้าหน้าที่อุปกรณ์ด้านทัศนศาสตร์
   - `4145` เจ้าหน้าที่พิพิธภัณฑ์

3. **23 TSCO Units zeroed by Major filter** — concept moved across Major in ISCO-08 restructuring. Includes street food vendors, agricultural managers, photographers, mining workers, teaching assistants (3xx → 2xxx upgrade), etc.

4. **TSCO vocabulary is 2001-era** — no entries for DevOps, blockchain, IoT, data scientist, ML engineer. Modern ICT/data roles must be accessed via ESCO (English) since TSCO has no Thai vocabulary for them.

5. **`isco08` codes new in 2008 unreachable via this bridge** — some ISCO-08 Units (e.g., 2529 Database/network professionals n.e.c. = where IT security specialists fit) have no ISCO-88 predecessor in the ILO crosswalk and therefore can't be reached from any TSCO code, even when semantically correct.

---

## Query patterns

### Load all bridge tables

```python
import duckdb
con = duckdb.connect()
for tbl in ["isco88_to_isco08", "tsco_unit_to_isco08", "tsco_occupation_to_isco08_clean"]:
    con.execute(f"CREATE VIEW {tbl} AS SELECT * FROM read_parquet('THAI/{tbl}.parquet')")
```

### Q1. Thai TSCO Occupation → WASK profile (full chain)

```sql
-- Thai job title "วิศวกรเครื่องกล" → WASK ratings
SELECT w.dimension, w.element_label, w.importance_im, w.level_lv
FROM tsco_occupation_to_isco08_clean t
JOIN crosswalk cw ON cw.esco_uri IN (
    SELECT esco_uri FROM occupations_esco WHERE isco_code = t.isco08_code
)
JOIN occupation_wask w ON w.onet_soc = cw.onet_soc
WHERE t.tsco_occ_name_th LIKE '%วิศวกรเครื่องกล%'
  AND cw.match_type IN ('exactMatch', 'closeMatch')
  AND NOT w.recommend_suppress
ORDER BY w.importance_im DESC
LIMIT 30;
```

### Q2. Get all Thai labels for a given ISCO-08 Unit

```sql
-- Thai vocabulary under ISCO-08 2144 (Mechanical engineers)
SELECT tsco_occ_code, tsco_occ_name_th, tsco_occ_description
FROM tsco_occupation_to_isco08_clean
WHERE isco08_code = '2144'
ORDER BY tsco_occ_code;
```

### Q3. Find Units zeroed by Major filter (audit dilution cases)

```sql
SELECT tsco_code, tsco_name_th, isco08_code, isco08_name_th, isco08_major
FROM tsco_unit_to_isco08
WHERE status = 'filtered'
  AND tsco_code NOT IN (
    SELECT tsco_code FROM tsco_unit_to_isco08 WHERE status='kept'
  )
ORDER BY tsco_code;
```

---

## Provenance

- **ILO `corrtab88-08.xls`** retrieved via `git clone https://github.com/Guidowe/occupationcross.git` → `data-raw/cross/corrtab88-08.xls`. R package mirrors original ILO file unchanged.
- **Build script trace:** session transcripts in CHANCEDEE V3 / CD3-3 CR (2026-05-13).
- **Generated by:** Claude (Anthropic) under Ford direction.
- **No LLM mapping** — all bridges are deterministic from ILO source + structural filter.
