# Claude Code Brief — Trading Report Bakery v6 (Demo Build)

**Goal:** Add UOM-aware entry and month-over-month variance flagging to an existing interim stock-count app, for a demo tomorrow.

**This is a demo build against seeded data.** Not production. Do not migrate live data. Do not touch Firebase security rules.

---

## 1. Repo context

Static single-page app, no build step. Files:

```
index.html    ~100 lines, mostly base64 logo assets — barely needs touching
style.css     472 lines, CSS custom properties (--txt, --border, --r12, etc.)
app.js        2,223 lines, vanilla JS, all logic — THIS IS THE MAIN FILE
firebase.js   Firebase config + LOGO_URI constant
data.json     { items: [330], stores: [185], admin: {} }
```

Conventions in `app.js` to follow: template-literal HTML injected into `#content`;
`esc()` for all user/data strings; `fNum(n, dec)` for number formatting;
`toast(msg, type)` for feedback; `showModal(html)` / `closeModal()`;
Thai UI text throughout. Helper `dbGet/dbSet/dbUpdate/dbPush` wrap Firebase.

Relevant existing functions:
- `renderEntry()` → `loadEntryForMonth()` → `buildEntryView()` → `buildEntryRows()`
- `refreshEntryBody()` — re-renders tbody on filter/search change
- `onQty(code, val)` — writes to `ENTRY_DATA`, sets `DIRTY`
- `saveEntry()` — multi-path update to `entries/{storeNo}/{YYYY-MM}/{itemCode}`

---

## 2. FIRST: fork to a demo namespace

Before any other change. In `app.js`, introduce a single constant and route every
Firebase path through it:

```js
const DB_ROOT = 'demo';   // all reads/writes go under demo/...
```

Apply to `entries`, `monthControl`, `logs`, `masterData`, `presence`.
Easiest safe approach: change `dbGet`/`dbSet`/`dbUpdate`/`dbRemove`/`dbPush` to
prefix `DB_ROOT + '/'` internally, and leave every call site unchanged.

Verify no write can land outside `demo/` before proceeding.

---

## 3. Schema change

**Current:**
```
entries/{storeNo}/{YYYY-MM}/{itemCode} = 12.5          // bare number
```

**New:**
```
entries/{storeNo}/{YYYY-MM}/{itemCode} = {
  qty:       12.5,          // number counted
  uom:       "ลัง",          // unit the count was taken in
  pack_size: 120,           // sub-units per one `uom`
  counted_at: 1721740000000
}
```

**Backward compatibility is required.** Write a normaliser used on every read:

```js
function normalizeEntry(v) {
  if (v === null || v === undefined || v === '') return null;
  if (typeof v === 'number' || typeof v === 'string') {
    return { qty: parseFloat(v) || 0, uom: null, pack_size: null,
             legacy: true, counted_at: null };
  }
  return { qty: ..., uom: ..., pack_size: ..., legacy: false, ... };
}
```

Legacy rows render with the UOM cell showing `หน่วยไม่ระบุ` in warning colour.
**Never silently coerce a legacy value into a UOM.** Do not run any migration.

---

## 4. Item master reference (new file)

Create `master_uom.json` from the supplied FBK3 extract — 82 items, keyed by item code:

```json
{
  "133352": { "packtype": "ลัง",   "pack_size": 120, "sub_uom": "ชิ้น" },
  "859997": { "packtype": "ลัง",   "pack_size": 6000, "sub_uom": "ชิ้น" },
  "198368": { "packtype": "ชิ้น",  "pack_size": 30,  "sub_uom": "ฟอง" }
}
```

Load alongside `data.json` in `loadData()` into a global `MASTER_UOM = {}`.

**Source-file quirks to handle when generating this JSON:**
- Header labels are swapped vs. content — the column headed `Sub Unit` holds the
  *number*, `Sub UnitPack` holds the *unit name*. Trust position, not header.
- Two adjacent columns both named `Packtype`: abbreviated (`ลง`) and full (`ลัง`).
  **Use the full form.**
- `Sub UnitPack` is text in 81 of 82 rows — coerce to number, drop non-numeric.
- Skip rows where packtype is `#N/A` or class is `0` (1 row: "Bakery น้ำเย็น").
- Normalise `ชน.` → `ชิ้น`.
- Only 67 of 330 app items will have a master record. **Missing is the normal case.**

---

## 5. UOM vocabulary (dropdown)

Derived from the FBK3 extract. Single flat list, this exact order:

```js
const UOM_LIST = [
  "ลัง", "กล่อง", "ถุง", "แพ็ค", "กระสอบ",
  "ชิ้น", "ฟอง", "ซอง", "ชุด",
  "กรัม", "กิโลกรัม"
];
```

Default selection: the item's `packtype` from `MASTER_UOM` if present, else blank
(`— เลือกหน่วย —`). Blank is allowed while editing; flagged on save.

---

## 6. Entry screen changes

Replace the single `Pack Size` column. New table columns:

| No. | Class | รหัส | ชื่อสินค้า | จำนวน | หน่วยนับ | ขนาดบรรจุ | อ้างอิง | สถานะ |
|---|---|---|---|---|---|---|---|---|

- **จำนวน** — numeric input, `step=0.01`, min 0. Reuse existing `qty-inp` class.
- **หน่วยนับ** — `<select>` from `UOM_LIST`.
- **ขนาดบรรจุ** — numeric input, prefilled from `MASTER_UOM[code].pack_size`
  when the row is first rendered empty. User can override.
- **อ้างอิง** — read-only. Shows `120 ชิ้น/ลัง` from master, or `—` if no master record.
- **สถานะ** — flag chips (§7).

Keep existing keyboard navigation working: Enter / ArrowDown / ArrowUp should move
between **จำนวน** inputs, skipping the select and pack-size fields.

### Replace the meaningless total

The current footer sums `Pack Size` across all items — that number has never meant
anything. Replace with **per-UOM subtotals**:

```
รวม: 45 ลัง · 1,200 ชิ้น · 8,500 กรัม   (กรอกแล้ว 62 / 330 รายการ)
```

Same change in the KPI tile on the store dashboard (`📦 Pack Size รวมเดือนนี้`).

---

## 7. Flag logic

Compute per item on load, against the previous calendar month
(`entries/{storeNo}/{prevYM}/{itemCode}`, normalised).

Evaluate in this order and show **at most one** chip, first match wins:

**A. UOM changed** — `prev.uom` and `cur.uom` both present and different.
```
⚠️ หน่วยนับเปลี่ยน: ชิ้น → แพ็ค
```
Amber. **Do not display a percentage.** A delta across different units is
meaningless and showing one is the exact error being guarded against.
Requires acknowledgment before save (§8).

**B. Pack size mismatch vs. master** — `cur.pack_size` present, master record exists,
and they differ.
```
⚠️ ขนาดบรรจุไม่ตรงกับ master (master: 6000)
```
Amber. Informational — does not block save. This is the master-data validation output.

**C. Quantity variance** — same UOM in both months, `prev.qty > 0`,
`|cur.qty − prev.qty| / prev.qty > 0.30`.
```
🔺 +85%   (เดือนก่อน 12)      /      🔻 −60%  (เดือนก่อน 40)
```
Red if increase, blue if decrease. Informational.

**D. New item** — no prior-month record, current has a value. `🆕 รายการใหม่`. Neutral grey.

**E. Missing UOM** — qty entered but `uom` blank. `⚠️ ยังไม่ระบุหน่วยนับ`. Amber. Blocks save.

Threshold: hardcode `const VARIANCE_THRESHOLD = 0.30;` as a module-level constant
with a comment that it becomes admin-configurable later. **Do not build a settings screen.**

Add a filter control next to the existing Class filter:
`แสดง: ทั้งหมด / เฉพาะรายการที่ต้องตรวจสอบ` — the latter shows only rows with a flag.

---

## 8. Save behaviour

Extend `saveEntry()`:

- Build the new object shape per item; write `null` for cleared rows as today.
- Preserve the existing month-active recheck, `_saveInProgress` guard,
  `FB_ONLINE` check, and 3× retry with backoff. **Do not refactor these.**
- **Before writing**, if any row has flag **A** or **E**, open a modal listing them:
  ```
  พบ N รายการที่ต้องยืนยัน
  [list: code — name — flag]
  ☐ ยืนยันว่าตรวจสอบแล้ว
  [ยกเลิก] [ยืนยันและบันทึก]
  ```
  Checkbox must be ticked to enable the confirm button.
- On success, `dbPush('logs', {...})` as today, plus `flagged_count`.

---

## 9. Seed script

Create `seed.js` — a standalone script (or an admin-only button, whichever is faster)
that writes demo data under `demo/`. This is what the demo runs on.

Seed **20 items** drawn from codes present in both `data.json` and `master_uom.json`,
across classes 68 and 78, for **two months: `2026-06` and `2026-07`**.

Also set `demo/monthControl/2026-06/active = false` and `2026-07/active = true`.

Planted cases — the demo depends on these:

| # | Case | June | July | Expected flag |
|---|---|---|---|---|
| 1 | **UOM change** | `qty 240, uom ชิ้น, pack 1` | `qty 2, uom ลัง, pack 120` | **A** — no % shown |
| 2 | **Pack size mismatch** (item `859997`) | `qty 1, uom ลัง, pack 6000` | `qty 1, uom ลัง, pack 16000` | **B** — master says 6000 |
| 3 | Large increase | `qty 12, uom ลัง` | `qty 22, uom ลัง` | **C** +83% |
| 4 | Large decrease | `qty 40, uom กล่อง` | `qty 15, uom กล่อง` | **C** −63% |
| 5 | Legacy row | bare number `35` | *(empty)* | renders `หน่วยไม่ระบุ` |
| 6 | New item | *(none)* | `qty 5, uom ถุง` | **D** |
| 7 | Missing UOM | `qty 8, uom ถุง` | `qty 9, uom null` | **E** — blocks save |
| 8–20 | Stable | small ±5% drift, same UOM | | none |

Case 2 is a **real discrepancy in the supplied master data** — the item name reads
`1X16000ชน` while `Sub UnitPack` says `6000`. Keep it; it demonstrates the flag
catching a genuine error rather than a contrived one.

---

## 10. Out of scope tonight

Do not build, and do not refactor toward:

- Admin all-stores rollup, admin dashboard KPIs, `renderStoreStatus`, `renderPresence`
- Excel export/import changes (leave exports emitting the legacy column; they will
  break gracefully on new-shape rows — acceptable for demo, note it in the README)
- Firebase Auth, RTDB security rules, password changes
- Any migration of live `entries/` data
- Approval workflow / draft-vs-submitted states
- Admin-configurable thresholds
- Cost, valuation, or any arithmetic involving `Cost` / `Cost Vat` from the FBK3 file.
  **The app must never multiply a quantity by a rate.** Ignore those columns entirely.

---

## 11. Priority order

Build in this sequence and stop wherever time runs out:

1. `DB_ROOT` fork + seed script *(nothing else is safe or demonstrable without these)*
2. Schema + `normalizeEntry()` + `master_uom.json` loading
3. Entry screen: three fields + reference column
4. Flags A and C *(the core demo)*
5. Per-UOM subtotals replacing the meaningless sum
6. Flags B, D, E + save-confirmation modal
7. Flag filter control

---

## 12. Acceptance checks

- [ ] No Firebase write occurs outside `demo/` — verify in console before demoing
- [ ] Legacy bare-number row renders without crashing, shows `หน่วยไม่ระบุ`
- [ ] Case 1 shows a UOM-change chip and **no percentage anywhere on that row**
- [ ] Case 2 shows the master mismatch with `6000` cited
- [ ] Save is blocked until the confirmation checkbox is ticked
- [ ] Footer shows per-UOM subtotals, not a single cross-unit sum
- [ ] Class filter, search, and Enter/arrow navigation still work
- [ ] Renders usably at 380px width
- [ ] An item with no `master_uom.json` record renders with `—` in อ้างอิง and no error
