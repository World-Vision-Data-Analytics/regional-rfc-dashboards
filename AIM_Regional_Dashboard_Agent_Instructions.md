# AIM / AIM+ FY26 — Regional Indicator Dashboard
## Agent Build Playbook (region-agnostic)

**Purpose.** Reproduce, for any World Vision region, the self-contained HTML regional
indicator dashboard first built for East Asia (EASO). One workbook feeds three layers —
**Descriptive** (current status), **Time-series** (year-over-year change), and **RC/sponsorship**
(registered-child gap). The output is a single `index.html` with no external dependencies.

**Companion documents.** `AIM_OIOS_Agent_Instructions.docx` (indicator computation from
microdata), `AIM_Reader_Workbook_Agent_Instructions.md` (reader-workbook rollout),
`AIM_AIM+ FY26 Analytics Architecture Guidelines` (the three-pillar method this dashboard renders).

**Governing ethos.** Audit before build. Derive everything from the workbook every run —
never hardcode a region's columns, sections, or reverse-codes. Be faithful to the source and
*flag* its anomalies rather than silently absorbing them. Do not be misleading: stamp low-n and
non-comparable results; never present two conflicting "current" numbers for one indicator.

---

## 0. Inputs and parameters

The agent is given, or must establish, the following before building:

1. **Authoritative workbook** — `FY26_AIM_AIM_Plus_Executive_Dashboard_vShare.xlsx` (or the
   current equivalent). This is the single source of truth. Three sheets are used:
   - `Descriptive global dashboard` — current-status values, all regions side by side.
   - `Time-series global dashboard` — FY25→FY26 change cells (rich text + fill colour).
   - `RC only descriptive dashboard` — registered-child values (sponsorship regions only).
2. **Template HTML** — an existing regional dashboard (the EASO file is the reference). Supplies
   the CSS, the JavaScript render engine, and the body markup. The agent reuses this verbatim and
   injects new data + a handful of region-specific patches. Do not rebuild the front-end from scratch.
3. **Region parameters** (supply these at the top of the build script):

   | Parameter | Example (EASO) | How to obtain |
   |---|---|---|
   | `REGION_NAME` | `East Asia (EASO)` | GC / launch deck |
   | `COUNTRIES` (ordered) | Mongolia, Cambodia, China, Vietnam, Thailand, Myanmar, Laos | **Detected** from the sheet header row (see §2); this list is the expected roster to match against |
   | `SHORT` codes | MNG, KHM, CHN, VNM, THA, MMR, LAO | ISO-3 or the workbook's own short codes |
   | `SPONSORSHIP` | Yes (RC layer present) | Regions with sponsorship programming have RC data; grant-only regions (e.g. MEER) do not |
   | `SUBOFFICE_RULE` | None (7 clean countries) | Some regions split a country into sub-offices (e.g. MEER's Syria hubs) and must combine them |

   Country rosters for the FY26 AIM+ regions are in the launch deck. Do **not** trust the roster
   blindly — detect the actual columns present (§2) and reconcile against the expected list.

---

## 1. Sequence (never deviate from this order)

```
AUDIT  →  BUILD  →  VERIFY
```

- **Audit.** Open the workbook. Detect the region's country columns in each sheet. Detect section
  boundaries. Read the NOTE row. Reconcile the example/template against the current workbook and
  decide it is a *style reference only* (examples are routinely stale — see §9).
- **Build.** Extract the three layers, assemble one `DATA` array, derive `INVERSE`, inject into the
  template, apply the trend-endpoint suppression and footnotes.
- **Verify.** Run the full suite in §8. Do not present until every check is zero-defect.

---

## 2. Region-agnostic column detection (critical)

The "global dashboard" sheets lay **all regions side by side**. You must locate *this* region's
country columns; never assume a fixed range like `K:Q`.

**Method.** Scan the first ~12 rows of each sheet for the header row — the row whose cells contain
country names. Match cells against the region's expected roster (country names are globally unique,
so name-matching is unambiguous). Record the matched column indices **in sheet order** and the
country each maps to. Confirm the block is contiguous and complete.

Header rows observed in the FY26 workbook (verify, don't assume):

| Sheet | Header row | EASO columns |
|---|---|---|
| `Descriptive global dashboard` | 5 | K–Q (11–17) |
| `Time-series global dashboard` | 8 | K–Q (11–17) |
| `RC only descriptive dashboard` | 5 | D–J (4–10) |

```python
def detect_country_cols(ws, expected_countries, header_scan=14, max_col=200):
    """Return (header_row, [(col_index, country_full), ...]) in sheet order."""
    want = {c.lower(): c for c in expected_countries}
    for r in range(1, header_scan + 1):
        hits = []
        for c in range(1, max_col + 1):
            v = norm(ws.cell(r, c).value).lower()
            if v in want:
                hits.append((c, want[v]))
        if len(hits) >= max(3, len(expected_countries) // 2):
            return r, hits
    raise RuntimeError("Country header row not found — check region roster / sheet")
```

Notes:
- The **RC sheet may list fewer countries** than Descriptive (only sponsorship offices). That is
  expected. Detect independently per sheet.
- If the detected set differs from the expected roster, **stop and report** — do not guess.

---

## 3. Section / domain detection

Column A carries the child-results section headers and the additional-indicators header. Detect
boundaries by regex; assign each indicator row to the section that precedes it.

| Column-A pattern | Domain key | Result | Sub-label |
|---|---|---|---|
| `1.` | `1.` | Result 1 | Hope |
| `2.1` | `2.1` | Result 2 | Health |
| `2.2` | `2.2` | Result 2 | Nutrition |
| `3.` | `3.` | Result 3 | Protection |
| `4.` | `4.` | Result 4 | Learning |
| `5.` | `5.` | Result 5 | Resilience |
| `6.` | `6.` | Result 6 | WASH |
| `ADDITIONAL INDICATORS…` | `ADD` | Additional | (outside CRF) |

```python
def find_sections(ws, col=1):
    secs = []
    for r in range(1, ws.max_row + 1):
        a = norm(ws.cell(r, col).value)
        if not a: continue
        if re.match(r'^\d+\.\d+\s', a):
            secs.append((r, re.match(r'^(\d+\.\d+)\s', a).group(1)))
        elif re.match(r'^\d+\.\s', a):
            secs.append((r, re.match(r'^(\d+)\.', a).group(1) + '.'))
        elif a.upper().startswith('ADDITIONAL INDICATORS'):
            secs.append((r, 'ADD'))
    return secs
```

- The **six child-results domains are fixed**; the domain *count* is always 6. `ADD` is everything
  after the ADDITIONAL header.
- Ship `ADD` as a **separate opt-in filter**, excluded from the default "All Results" view. It
  surfaces indicators the workbook files outside the CRF (and, in sponsorship regions, RC-specific
  indicators) without silently dropping anything. **Flag this to the requester** and offer to remove
  it for strict example-parity.

---

## 4. Shared helpers, indicator parsing, values

```python
import re
from decimal import Decimal, ROUND_HALF_EVEN
from openpyxl import load_workbook

def norm(s):
    if s is None: return ''
    return str(s).replace('\u200b', '').replace('\xa0', ' ').replace('\u2019', "'").strip()

def round1(x):                                   # round-half-to-even, 1 decimal
    return float(Decimal(str(x)).quantize(Decimal('0.1'), rounding=ROUND_HALF_EVEN))

def norm_code(raw):                              # C-codes get an IND prefix
    raw = raw.strip()
    if raw.startswith('IND'): return raw
    if re.match(r'^C\d', raw): return 'IND' + raw
    return raw

def parse_ind(a):                                # "<code> - <desc>" or "<code>  <desc>"
    m = re.match(r'^([A-Za-z0-9_.]+(?:\s+(?:CG|AD|adol))?)\s*[-\u2013\u2014]\s*(.*)$', a) \
        or re.match(r'^([A-Za-z0-9_.]+)\s{2,}(.*)$', a)
    if not m: return None
    code = m.group(1).strip()
    return (code, m.group(2).strip()) if re.match(r'^(IND|C\d)', code) else None

SKIP = ('FY26', 'NOTE', 'LEGEND', 'ADDITIONAL', 'INDICATOR (IND#)')
def is_skip(a):
    au = a.upper()
    return (au.startswith(SKIP)
            or bool(re.match(r'^\d+(\.\d+)?\.?$', a))
            or bool(re.match(r'^\d+\.\d+\s', a)) or bool(re.match(r'^\d+\.\s', a)))
```

**Rules that must hold across all three layers:**
- **Missing markers** `''`, `'-'`, `'.'` → null (no data). Never coerce to 0.
- **Rounding** is round-half-to-even to 1 decimal, applied uniformly. (This produces occasional
  ±0.1 differences from a stale example that used round-half-up — the spec value is the even one.)
- **Code normalisation**: C-codes get `IND` prefixed so everything is an `IND…` key.
- **IND41 duplicate**: the workbook lists IND41 twice (adults; adolescents). Split by description →
  `IND41` (adults) and `IND41_adol` (adolescents).
- **Duplicate normalised codes**: e.g. `INDC1C.23176` (in WASH) and `C1C.23176` (in Additional) both
  normalise to `INDC1C.23176` but are different rows/offices. Keep **both**; disambiguate the second
  under its **raw workbook code**. Never silently merge — that hides a source inconsistency.

---

## 5. Layer 1 — Descriptive (status)

For each indicator row in the region's columns: collect per-office values (rounded, nulls kept in
`all_vals`, non-nulls also in `countries`), then `n = reporting offices`, `avg = unweighted mean of
reporting offices`, `min`, `max`, `spread`, and an **ascending** `countries` list.

```python
def extract_descriptive(ws, cols, COUNTRIES, SHORT):
    secs = find_sections(ws)
    def sec_for(r):
        key = None
        for sr, k in secs:
            if sr < r: key = k
            else: break
        return key
    out, seen = [], {}
    for r in range(1, ws.max_row + 1):
        a = norm(ws.cell(r, 1).value)
        if not a or is_skip(a): continue
        pi = parse_ind(a)
        if not pi: continue
        raw_code, desc = pi
        key = sec_for(r)
        if key is None: continue
        code = norm_code(raw_code)
        allv, rep = [], []
        for i, c in enumerate(cols):
            v = ws.cell(r, c).value; vs = norm(v); val = None
            if vs not in ('', '-', '.'):
                try: val = round1(float(v))
                except: val = None
            allv.append({'short': SHORT[COUNTRIES[i]], 'full': COUNTRIES[i], 'value': val})
            if val is not None:
                rep.append({'short': SHORT[COUNTRIES[i]], 'full': COUNTRIES[i], 'value': val})
        if not rep: continue
        disp = uniq = code
        if code == 'IND41' and 'adolescent' in desc.lower():
            disp = uniq = 'IND41_adol'
        if uniq in seen:                                  # duplicate-code disambiguation
            disp = uniq = raw_code
            if uniq in seen: uniq = disp = f"{raw_code}__r{r}"
        seen[uniq] = r
        vals = [x['value'] for x in rep]
        out.append({'row': r, 'domain_key': key, 'ind_code': disp, 'ind_desc': desc,
                    'n': len(rep), 'avg': round1(sum(vals)/len(vals)),
                    'min': min(vals), 'max': max(vals), 'spread': round1(max(vals)-min(vals)),
                    'countries': sorted(rep, key=lambda x: x['value']), 'all_vals': allv,
                    'trend': [], 'rc_gap': None, 'rc_rows': [], 'ts_only': False})
    # attach result labels from a fixed RESULT_META map keyed by domain_key
    return out
```

`RESULT_META` (fixed; supplies `result_num`, `result_short`, `sub_label`, `result_label`) —
1.→R1 Hope, 2.1→R2 Health, 2.2→R2 Nutrition, 3.→R3 Protection, 4.→R4 Learning,
5.→R5 Resilience, 6.→R6 WASH, ADD→Additional. Confirm each domain maps to exactly one label set.

---

## 6. Reverse-coding — derive from the NOTE row (do not hardcode)

The Descriptive sheet has a NOTE row: *"Reverse-coded (lower-is-better) indicators: …"*. Parse the
codes from that row every run. **This is the single source of truth for direction.**

```python
def derive_reverse(ws):
    note = None
    for r in range(1, ws.max_row + 1):
        a = norm(ws.cell(r, 1).value)
        if a.upper().startswith('NOTE') and 'REVERSE' in a.upper():
            note = a; break
    if not note: raise RuntimeError("Reverse-code NOTE row not found")
    body = note.split(':', 1)[1] if ':' in note else note
    # MUST match IND-prefixed C-codes (INDC1A.0008) as well as IND123 and bare C2A.x
    codes = re.findall(r'\bIND\d+\b|\bINDC\d[A-Za-z]?\.\w+|\bC\d[A-Za-z]?\.\w+', body)
    return set(norm_code(c) for c in codes)
```

> **Regex gotcha (cost a real bug).** A naive pattern `\b(?:IND\d+|C\d…)\b` **silently drops**
> `INDC1A.0008` / `INDC1A.0018` / `INDC1A.21053` because there is no word boundary between the `D`
> and the `C`, and `IND\d+` needs a digit after `IND`. You must include the `\bINDC\d…` alternative.
> Always verify stunting (`INDC1A.0008`) and wasting (`INDC1A.0018`) end up reverse-coded.

- `INVERSE` (shipped to the front-end) = derived reverse set ∩ codes actually present in the region.
- **`INDC2A.027994` (sense of safety) must NOT be reverse-coded** — higher is better. Deriving from
  the NOTE handles this automatically (it isn't listed). This is the documented trap; confirm it.

**Direction-aware bands** (green always = good):

| | Good (green #1D9E75) | Moderate (amber #EF9F27) | Low (red #E24B4A) |
|---|---|---|---|
| Normal (higher = better) | ≥ 70 | 40–69 | < 40 |
| Reverse-coded (lower = better) | ≤ 15 | 16–35 | > 35 |

Comparability threshold: an indicator is regionally comparable only at **n ≥ 3** reporting offices.
Show low-n indicators but exclude them from regional comparison and flag them.

---

## 7. Layer 2 — Time-series (change), and the two rules that matter most

Each time-series cell is rich text, e.g. `'78.45% → 74.80%\n▼ -3.66 pp\np=0.0378 **'`.

```python
def extract_trend(ws, cols, COUNTRIES, SHORT):
    trend = {}
    for r in range(1, ws.max_row + 1):
        a = norm(ws.cell(r, 1).value)
        if not a or is_skip(a): continue
        pi = parse_ind(a)
        if not pi: continue
        code = norm_code(pi[0]); desc = pi[1]
        key = 'IND41_adol' if (code == 'IND41' and 'adolescent' in desc.lower()) else code
        for i, c in enumerate(cols):
            v = ws.cell(r, c).value
            if v is None or '→' not in str(v): continue
            s = str(v)
            m = re.search(r'([\d.]+)%\s*→\s*([\d.]+)%', s)
            if not m: continue
            frm, to = float(m.group(1)), float(m.group(2))
            # (1) SIGN COMES FROM THE ARROW, not the text sign
            am = re.search(r'([▲▼])\s*[+-]?\s*([\d.]+)\s*pp', s)
            mag = float(am.group(2)); pp = mag if am.group(1) == '▲' else -mag
            # (2) SIGNIFICANCE COMES FROM THE p-VALUE, not the fill colour
            pm = re.search(r'p\s*([<=>])\s*([\d.]+)', s)
            if pm:
                op, val = pm.group(1), float(pm.group(2))
                sig = (val < 0.05) or (op == '<' and val <= 0.05)   # harden the p<0.05 edge
            else:
                sig = False
            trend.setdefault(key, []).append(
                {'short': SHORT[COUNTRIES[i]], 'full': COUNTRIES[i],
                 'from': round(frm, 2), 'to': round(to, 2), 'pp': round(pp, 2), 'p': (val if pm else 1.0), 'sig': sig})
    return trend
```

> **Rule 1 — pp sign from the arrow (▲=+, ▼=−).** Cells vary: some are unsigned (`▲ 1.39 pp`), some
> have a spaced minus (`▼ - 2.64 pp`). The arrow glyph is the reliable sign; trust it over the text.
>
> **Rule 2 — significance from the p-value, never the fill colour.** The workbook's colour scheme is
> green = significant-favourable, red = significant-unfavourable, amber = marginal (0.05 ≤ p < 0.10),
> grey = neutral — **but it is not reliable.** In the EASO data a genuine −18.5 pp, p<0.001 decline
> (Myanmar school attendance) was accidentally left **grey**. Reading the p-value flags it correctly;
> reading the colour would have dropped a major finding. Asterisks (`*`/`**`) are inconsistent — ignore.

> **Rule 3 — DO NOT display the from→to endpoints.** The time-series "to" (current-year) is computed
> on a **different basis** than the Descriptive status and frequently diverges: in EASO, 41 of 98
> office-cells differed by >2 pp (13 by >5 pp), concentrated in specific offices, and the status
> matched *neither* endpoint in 86 of 98 cells (it is an independent figure). Showing both would put
> two conflicting "current" numbers on one page. **Render only the tested change: Δpp + direction +
> significance.** Keep `from`/`to` in the data but not on screen, and add the footnote (§9.4).

---

## 8. Layer 3 — RC / sponsorship (conditional)

Only sponsorship regions have RC data. Detect it; **do not assume presence or absence.** Grant-only
offices (and grant-only regions such as MEER) legitimately have no RC values — that is correct, not
a gap to fill.

For each indicator+office where **both** an RC value and a Descriptive value exist:
`all = round1(descriptive)`, `rc = round1(rc)`, `gap = round1(rc − descriptive)`.
`rc_rows` sorted by `gap` ascending; `rc_gap = round1(mean of the rounded gaps)`.

```python
def extract_rc(ws, cols, COUNTRIES):
    rc = {}
    for r in range(1, ws.max_row + 1):
        a = norm(ws.cell(r, 1).value)
        if not a or is_skip(a): continue
        pi = parse_ind(a)
        if not pi: continue
        code = norm_code(pi[0]); desc = pi[1]
        key = 'IND41_adol' if (code == 'IND41' and 'adolescent' in desc.lower()) else code
        for i, c in enumerate(cols):
            v = ws.cell(r, c).value; vs = norm(v)
            if vs in ('', '-', '.'): continue
            try: rc.setdefault(key, {})[COUNTRIES[i]] = float(v)
            except: pass
    return rc     # combine with raw descriptive values in assemble() to compute all/rc/gap
```

If the region has **no RC data at all**, omit the RC filter/panel entirely (or state its absence);
do not ship an empty equity layer.

---

## 9. Assemble, patch the template, add footnotes

### 9.1 Assemble `DATA`
Merge the three layers by key (`ind_code`). `trend` and `rc_rows` attach by the same key; the
disambiguated Additional duplicate (raw-code key) correctly receives no trend/RC. Header tiles:

```python
def favourable(d, t, INVERSE):     # a rise is good normally; a fall is good if reverse-coded
    return (t['pp'] > 0) != (d['ind_code'] in INVERSE)
```
- `CRF indicators` = count where `domain_key != 'ADD'`
- `offices` = region country count
- `comparable` = CRF indicators with `n ≥ 3`
- `favourable / total` = over significant CRF trend office-cells, how many are favourable.

### 9.2 Reuse the template; patch only these points
Reuse the template's CSS + JS render engine verbatim. Replace/patch:
1. `const DATA = [...]` → new JSON.
2. `const INVERSE = [...]` → derived list; fix its comment to reference this region.
3. `ord` map → add `ADD:7`.
4. `rLabels` array in `setResult` → append `'Additional'`.
5. Filter bar → add the `Additional` button (after Result 6).
6. CSS → add `.dom-ADD`.
7. `filteredData` → exclude `ADD` from the `'all'` view.
8. Header → region eyebrow, sub-line (CRF vs additional split, office count, layer note), the four
   count tiles, `<title>`.

### 9.3 Apply the trend-endpoint suppression
In `trendBlock`, drop the `from% → to%` cell from each row (realign the CSS grid to 4 columns:
country · change-bar · Δpp *with `pp` unit* · significance). Keep `from`/`to` in the data only.

### 9.4 Add the footnotes (required)
- **In the trend panel note**: state that absolute FY25/FY26 levels are not shown because the
  time-series is computed separately from the status figures and the two can differ; point to the
  status panel for current levels. End with a `†`.
- **Global page footnote** (add a `.page-footnote` block after `.main`): status vs. change come from
  two separate workbook sheets and can diverge (name the worst-affected offices for this region);
  significance is read from the p-value, not cell shading; reverse-coding follows the NOTE row;
  round-half-to-even to 1 decimal; n ≥ 3 comparability rule.

---

## 10. Region-specific variation (check every time)

| Concern | What to do |
|---|---|
| **Country columns** | Detect per sheet (§2). Never reuse another region's range. |
| **Sponsorship (RC)** | Present → ship RC layer. Grant-only region → omit it. Detect, don't assume. |
| **Sub-offices** | If the region splits a country into sub-offices (e.g. MEER Syria hubs), combine them per the region's rule (HH-weighted or as the workbook dictates) before computing office-level figures. EASO had 7 clean countries — N/A. |
| **Which offices diverge** | Recompute the trend-vs-status divergence for *this* region and name the worst-affected offices in the footnote. |
| **Reverse-code roster** | Same NOTE row, but only codes present in the region enter `INVERSE`. |
| **Additional indicators** | The `ADD` set differs by region. Always ship it as an opt-in filter and flag it. |

---

## 11. Verification suite (mandatory — do not present until all pass)

Run every check. All must be zero-defect. Re-read the workbook **independently** (don't reuse the
build's own parsed objects) so the check can catch build bugs.

1. **Cell-by-cell (Descriptive)** — independent re-read vs the embedded `DATA` (tolerance 0.0501).
   Make the re-read **collision-aware** (mirror the duplicate-code disambiguation) or it will raise
   false mismatches on `INDC1C.23176` / `C1C.23176`.
2. **Derived stats** — `n`, `min`, `max`, `spread`, `avg`, ascending sort, `all_vals ↔ countries`.
3. **Trend reconciliation** — independent re-parse with **sign taken from the arrow**; compare
   `pp`, `from`, `to`, `sig` for every office-cell.
4. **RC reconciliation** — independent `all` / `rc` / `gap` and the `rc_gap` mean.
5. **Trend-to vs Descriptive divergence report** — quantify (buckets: 0–0.5 / 0.5–2 / 2–5 / >5 pp),
   confirm endpoints are suppressed in the render, and feed the worst offices into the footnote.
6. **Reverse-coding** — derived set == NOTE row; stunting/wasting included; `INDC2A.027994` excluded.
7. **Headless render (Node, DOM stubbed)** — exercise every filter, sort, search, and **every**
   detail view; expect **0 exceptions**.
8. **Self-containment** — no external `src`/`href`, no `fetch`/`XHR`, no `localStorage`/`sessionStorage`.
9. **Range / NaN** — all displayed values in [0, 100]; no NaN.
10. **Structure** — all `ind_code`s unique (so `selectInd` is safe); each domain maps to one label set.

```python
# Headless render harness (Node v22). Stub document/window, then:
#   eval(<script from the HTML>);  run setResult over all filters; setSort over all modes;
#   setQuery over a few strings; selectInd(d.ind_code) for EVERY d in DATA; backToList().
# Any thrown exception is a failure. (See render_test.js from the EASO build.)
```

---

## 12. Output and governance

- Name the file **`index.html`** for hosting.
- **SharePoint / OneDrive force-download raw HTML** — they will not render it. Host on a static web
  server (e.g. Azure Static Web Apps) or an intranet server. State this to the requester.
- The file embeds **every office's values**; treat the hosting location as a data-governance
  decision, not just a technical one.
- Keep the build script, the injection script, and the verification scripts alongside the output so
  the next region is a parameter change, not a rewrite.

---

## 13. One-screen checklist

```
[ ] Detect country columns per sheet (never hardcode a range)
[ ] Detect sections; ship ADD as an opt-in filter (flag it)
[ ] Missing → null; round-half-to-even to 1 dp
[ ] Split IND41 vs IND41_adol; disambiguate duplicate codes by raw code
[ ] Derive INVERSE from the NOTE row (regex must catch INDC1A.xxxx)
[ ] Direction-aware bands; n>=3 comparability
[ ] Trend pp sign FROM ARROW; significance FROM p-VALUE (not colour)
[ ] Suppress from→to endpoints; keep Δpp + significance; add footnotes
[ ] RC layer only if sponsorship present; detect, don't assume
[ ] Combine sub-offices if the region requires it
[ ] Header tiles: CRF count / offices / comparable / favourable-of-significant
[ ] Reuse template CSS+JS; patch DATA/INVERSE/ord/rLabels/ADD button/dom-ADD/filter/header
[ ] Run the full verification suite — zero defects
[ ] Name index.html; note SharePoint/governance caveat
```

*Prepared as a companion to the EASO FY26 AIM Regional Dashboard build. Faithful to the FY26
AIM/AIM+ Executive Dashboard workbook and the FY26 Analytics Architecture. Audit → Build → Verify.*
