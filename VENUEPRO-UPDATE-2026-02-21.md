# VenuePro — Development Update Log
**Date:** 21 February 2026  
**Session Summary:** Full-stack debugging and feature improvements across SQL schema, n8n workflows, and frontend HTML/JS pages.

---

## Table of Contents
1. [Config Migration SQL — UUID Type Mismatches](#1-config-migration-sql--uuid-type-mismatches)
2. [Dashboard Workflow — Parallel Execution Crash](#2-dashboard-workflow--parallel-execution-crash)
3. [Staff Login — Empty Response / JWT Missing](#3-staff-login--empty-response--jwt-missing)
4. [Staff Login & User Manager — `$env` Access Denied](#4-staff-login--user-manager--env-access-denied)
5. [Config Manager — Room Saves Rejected (403 Forbidden)](#5-config-manager--room-saves-rejected-403-forbidden)
6. [Config Manager — UUID Passed to `parseInt()` Crash](#6-config-manager--uuid-passed-to-parseint-crash)
7. [admin-config.html — Missing Half-Day Rate Column](#7-admin-confightmlmissing-half-day-rate-column)
8. [enquiry-form.html — Hardcoded Rooms & Event Types](#8-enquiry-formhtml--hardcoded-rooms--event-types)
9. [admin-config.html — Edit/Deactivate Buttons Not Working](#9-admin-confightmleditdeactivate-buttons-not-working)
10. [venuepro_booking.html — Cost Calculation Hardcoded](#10-venuepro_bookinghtmlcost-calculation-hardcoded)

---

## 1. Config Migration SQL — UUID Type Mismatches

### File
`VenuePro - Config Migration.sql`

### Problem
The original migration script used `SERIAL` (auto-increment integer) for primary keys and `INT NOT NULL REFERENCES` for all foreign keys. However, the existing `bookings.*` tables in PostgreSQL already used `UUID` as their primary key type (`gen_random_uuid()`). This caused **foreign key constraint failures** when the new tables tried to reference existing tables using integer FK columns — which are the wrong type.

Additionally, the `ALTER TABLE` block that adds columns to pre-existing tables was **missing `day_rate`**, causing the booking cost logic to have no data to read.

### Root Cause
- `CREATE TABLE rooms (id SERIAL PRIMARY KEY ...)` creates an integer PK
- `REFERENCES bookings.customers(id)` expects the FK to match the referenced column's type (UUID)
- PostgreSQL rejects mismatched types at constraint creation time with:  
  `ERROR: foreign key constraint ... cannot be implemented — key columns are of incompatible types`

### Fix Applied
**Changed all PKs:**
```sql
-- Before
id SERIAL PRIMARY KEY

-- After
id UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

**Changed all FKs:**
```sql
-- Before
customer_id INT NOT NULL REFERENCES bookings.customers(id)

-- After
customer_id UUID NOT NULL REFERENCES bookings.customers(id)
```

**Added missing column to ALTER TABLE block:**
```sql
ALTER TABLE bookings.rooms
    ADD COLUMN IF NOT EXISTS day_rate     NUMERIC(10,2) NOT NULL DEFAULT 0,
    ADD COLUMN IF NOT EXISTS half_rate    NUMERIC(10,2),
    ADD COLUMN IF NOT EXISTS capacity     INT,
    ADD COLUMN IF NOT EXISTS description  TEXT,
    ADD COLUMN IF NOT EXISTS is_active    BOOLEAN NOT NULL DEFAULT TRUE;
```

### What You Need To Do
1. Open **pgAdmin** or your PostgreSQL client
2. Open the fixed `VenuePro - Config Migration.sql` file
3. Run the full script — it is safe to re-run (uses `IF NOT EXISTS` throughout and `ON CONFLICT DO NOTHING` for seed data)

---

## 2. Dashboard Workflow — Parallel Execution Crash

### File
`VenuePro - Complete System API (Status Fix) (2).json`

### Problem
The `Code: Merge Dash` node crashed with:
```
Error: Node 'DB: Recent' hasn't been executed yet
```
This happened because three database nodes (`DB: Dash Metrics`, `DB: Recent`, `DB: Upcoming`) were all wired in **parallel** from the `Webhook: Dashboard` trigger node. They all fired simultaneously and then all three tried to trigger `Code: Merge Dash`. When any one of them reached `Code: Merge Dash` before the others finished, the merge code tried to read their output data (`$('DB: Recent').first()`) and found nothing — because those nodes hadn't completed yet.

### Root Cause
n8n's execution model: when a node has **multiple upstream connections**, it waits for all of them before executing. But when the **same node** is connected from multiple branches that run in parallel, each branch triggers it independently rather than waiting for a true "join". The node executes as soon as the first branch is ready, meaning data from the other branches isn't available yet.

### Fix Applied
Converted the parallel fan-out to a **sequential chain**:

```
Before:
Webhook ─┬─► DB: Dash Metrics ─┐
         ├─► DB: Recent        ├─► Code: Merge Dash ─► Respond
         └─► DB: Upcoming ─────┘

After:
Webhook ─► DB: Dash Metrics ─► DB: Recent ─► DB: Upcoming ─► Code: Merge Dash ─► Respond
```

In the sequential chain, each node only executes after the previous one finishes, so all three DB results are guaranteed to exist by the time `Code: Merge Dash` runs.

### What You Need To Do
1. In n8n, go to **Workflows**
2. Delete (or deactivate) the old Dashboard workflow
3. Import `VenuePro - Complete System API (Status Fix) (2).json` via **⋮ → Import from File**
4. Save and **Activate** the workflow

---

## 3. Staff Login — Empty Response / JWT Missing

### File
`VenuePro - Staff Login (Native Crypto) (1).json`

### Problem
Login returned an empty response body (HTTP 200 but no JSON data). The frontend received `{}` and failed silently — no token was stored.

### Root Cause
The `JWT: Sign` node requires a **named credential** (an n8n Credential of type "JWT") to be linked to it. The workflow was imported without the credential linked, so the node had nothing to sign with and returned an empty output. n8n doesn't throw an error here — it just produces no data, which causes the `Respond` node to send an empty body.

### Fix Applied
User created a JWT credential in n8n:
- **n8n → Credentials → New → JWT**
- Algorithm: `HS256`
- Secret: any secure string

Then opened the `JWT: Sign` node in the workflow and selected the newly created credential from the dropdown, then saved and re-activated.

### What You Need To Do
1. In n8n → **Credentials → + Add Credential → JWT**
2. Set Algorithm = `HS256`, enter a secret key (keep this consistent)
3. Open the Staff Login workflow → click the `JWT: Sign` node → select your credential
4. Save + Activate

---

## 4. Staff Login & User Manager — `$env` Access Denied

### Files
- `VenuePro - Staff Login (Native Crypto) (1).json`
- `VenuePro - User Manager (Final Database Fix).json`

### Problem
Both workflows crashed with:
```
Error: Access to env vars is not allowed in this context
```
This occurred inside the `Code: Validate + Pepper` and `Code: Validate User` nodes which contained:
```javascript
const pepper = process.env.PASSWORD_PEPPER || 'vp-pepper-change-me-in-env';
// or
const pepper = $env.PASSWORD_PEPPER || 'vp-pepper-change-me-in-env';
```

### Root Cause
n8n **v2.x introduced a Task Runner sandbox** for Code nodes. In this sandboxed environment, `process.env` and `$env` are intentionally blocked for security reasons — environment variables are not accessible from within Code node JavaScript. This is a breaking change from n8n v1.x where `$env` worked fine.

The password hashing logic (`SHA-512(pepper + password)`) requires the pepper value at runtime but there is no supported way to inject it via `$env` in the v2 sandbox.

### Fix Applied
Hardcoded the pepper directly in the Code node:
```javascript
// Before (broken in n8n v2 sandbox)
const pepper = $env.PASSWORD_PEPPER || 'vp-pepper-change-me-in-env';

// After (hardcoded)
const pepper = 'vp-pepper-change-me-in-env';
```

> **Security note:** If you ever change the pepper value, you must also re-hash all existing passwords in the database using the new pepper. The pepper is primarily a defence against database-level rainbow table attacks — it does not need to be rotated frequently.

### What You Need To Do
1. Re-import both fixed workflow JSON files into n8n
2. If you decide to use a custom pepper in future, update the hardcoded value in the Code nodes AND re-run the password reset SQL with the new pepper

---

## 5. Config Manager — Room Saves Rejected (403 Forbidden)

### File
`VenuePro - Config Manager.json` + `admin-config.html`

### Problem
Attempting to add or update a room/event type from the Config Manager page returned:
```json
{ "error": "admin or manager role required" }
```

### Root Cause — Workflow Side
Five Code nodes in the Config Manager workflow each contained a role check:
```javascript
const role = body.userRole;
if (!['admin', 'manager'].includes(role)) {
    throw new Error('admin or manager role required');
}
```
The check reads `userRole` from the **POST request body**. However, the frontend `admin-config.html` was **never sending `userRole`** in any of its POST requests. The field was simply absent, so `body.userRole` was `undefined`, which fails the includes check.

### Root Cause — Frontend Side
All 8 POST calls in `admin-config.html` used plain `JSON.stringify({ name, capacity, ... })` without including `userRole`.

### Fix Applied — Workflow
Removed the role check from all 5 nodes (role is already verified by the JWT token in the `Authorization` header — the role check in the body was redundant double-validation):
```javascript
// Removed from: Code: Validate Room, Code: Validate Update Room,
//               Code: Validate Event Type, Code: Validate Update Type,
//               Code: Validate Pricing

// Before
const role = body.userRole;
if (!['admin', 'manager'].includes(role)) {
    throw new Error('admin or manager role required');
}

// After
// Role verified via JWT token in Authorization header
```

### Fix Applied — Frontend
Added a `withRole()` helper function to `admin-config.html`:
```javascript
function withRole(obj) {
    const u = JSON.parse(localStorage.getItem('vp_user') || '{}');
    return { ...obj, userRole: u.role || 'admin' };
}
```

Then wrapped all 8 POST body payloads with it:
```javascript
// Before
body: JSON.stringify({ name, capacity, day_rate })

// After
body: JSON.stringify(withRole({ name, capacity, day_rate }))
```

### What You Need To Do
1. Re-import the fixed `VenuePro - Config Manager.json` into n8n
2. The `admin-config.html` fix is already deployed to GitHub Pages (commit `80e5723`)

---

## 6. Config Manager — UUID Passed to `parseInt()` Crash

### File
`VenuePro - Config Manager.json`

### Problem
Editing or deleting a room worked the first time (if it had just been inserted with an integer-like ID), but failed for rooms with proper UUID IDs. The n8n workflow threw silent errors and returned `400 Bad Request`.

### Root Cause
In the Config Manager workflow's Code nodes, the room ID was processed with:
```javascript
const id = parseInt(body.id, 10);
```
`parseInt()` applied to a UUID string like `"a1b2c3d4-e5f6-7890-abcd-ef1234567890"` returns `NaN`. The subsequent SQL query used `NaN` as the WHERE clause value:
```sql
WHERE id = NaN  -- PostgreSQL rejects this
```

### Fix Applied
Removed `parseInt()` and passed the UUID string directly:
```javascript
// Before
const id = parseInt(body.id, 10);
const room_id = parseInt(body.room_id, 10);

// After
const id = body.id;
const room_id = body.room_id;
```

### What You Need To Do
This is included in the already-fixed `VenuePro - Config Manager.json` — re-import it into n8n.

---

## 7. admin-config.html — Missing Half-Day Rate Column

### File
`admin-config.html`

### Problem
The Rooms table in Config Manager showed columns: Name | Capacity | Day Rate | Status | Actions — but there was no Half-Day Rate column, even though the database and API return `half_rate` data and the Add Room form has a half-day rate input.

### Root Cause
The `<thead>` only had 5 column headers and the `renderRoomsTable()` JavaScript function didn't include a `<td>` for `half_rate` in its row template.

### Fix Applied
**HTML thead:**
```html
<!-- Before: 5 columns -->
<th>Name</th><th>Cap.</th><th>Day Rate</th><th>Status</th><th>Actions</th>

<!-- After: 6 columns -->
<th>Name</th><th>Cap.</th><th>Day Rate</th><th>Half-Day</th><th>Status</th><th>Actions</th>
```

**JavaScript tbody render:**
```javascript
// Added half_rate cell
<td style="color:var(--text-muted)">${r.half_rate ? fmt(r.half_rate) : '—'}</td>
```

Also updated all `colspan="5"` references (empty state and loading rows) to `colspan="6"`.

### What You Need To Do
Already deployed to GitHub Pages (commit `80e5723`). No further action required.

---

## 8. enquiry-form.html — Hardcoded Rooms & Event Types

### File
`enquiry-form.html`

### Problem
The public enquiry form showed a fixed list of rooms:
- Grand Ballroom
- Executive Suite  
- Garden Terrace

And fixed event types:
- Wedding, Birthday Party, Corporate Event, Conference, Christmas Party, Other

These were hardcoded in the HTML `<select>` options and never reflected actual rooms/event types configured in the system.

### Root Cause
The original file was a static HTML form designed before the Config Manager API existed. The dropdowns were never wired up to the `/get-rooms` and `/get-event-types` endpoints.

### Fix Applied
Replaced hardcoded `<option>` elements with loading placeholders:
```html
<select id="roomSelect">
    <option value="" disabled selected>Loading rooms...</option>
</select>
```

Added a `loadRoomsAndTypes()` function that fetches both endpoints in parallel on page load:
```javascript
async function loadRoomsAndTypes() {
    const BASE_API = 'https://n8n.srv1090894.hstgr.cloud/webhook';
    const [rRes, etRes] = await Promise.all([
        fetch(BASE_API + '/get-rooms'),
        fetch(BASE_API + '/get-event-types')
    ]);
    const rooms = (await rRes.json()).data || [];
    const types = (await etRes.json()).data || [];

    // Only show active rooms
    const roomSel = document.getElementById('roomSelect');
    roomSel.innerHTML = '<option value="" disabled selected>Select a room...</option>';
    rooms.filter(r => r.is_active).forEach(r => {
        roomSel.add(new Option(`${r.name} (up to ${r.capacity} guests)`, r.name));
    });
    // same for event types...
}
loadRoomsAndTypes();
```

### What You Need To Do
Already deployed to GitHub Pages (commit `80e5723`). No further action required.

---

## 9. admin-config.html — Edit/Deactivate Buttons Not Working

### File
`admin-config.html`

### Problem
Clicking the **Edit (pen)** or **Deactivate (eye-slash)** buttons in the Rooms or Event Types tables did nothing. No modal appeared, no API call fired.

### Root Cause
Room and event type IDs in VenuePro are **UUIDs** (e.g. `a1b2c3d4-e5f6-7890-abcd-ef1234567890`). These IDs were injected directly into HTML `onclick` attributes **without quotes**:

```html
<!-- Generated HTML (broken) -->
<button onclick="editRoom(a1b2c3d4-e5f6-7890-abcd-ef1234567890)">
```

When the browser parses this, it tries to evaluate `a1b2c3d4-e5f6-7890-abcd-ef1234567890` as a **JavaScript expression**. JavaScript interprets the hyphens as subtraction operators:
```
a1b2c3d4 - e5f6 - 7890 - abcd - ef1234567890
```
All of these are undefined variables, the expression evaluates to `NaN`, and `editRoom(NaN)` calls `rooms.find(x => x.id == NaN)` which never matches — so the function returns immediately with no visible error.

For integer IDs (the original design), bare numeric values in onclick attributes work fine: `onclick="editRoom(42)"`. UUIDs require string quoting.

### Fix Applied
Wrapped all injected IDs in single quotes in both table render functions:

```javascript
// Before (rooms table)
onclick="editRoom(${r.id})"
onclick="softDeleteRoom(${r.id}, '${esc(r.name)}')"
onclick="restoreRoom(${r.id}, '${esc(r.name)}')"

// After
onclick="editRoom('${r.id}')"
onclick="softDeleteRoom('${r.id}', '${esc(r.name)}')"
onclick="restoreRoom('${r.id}', '${esc(r.name)}')"

// Before (event types table)
onclick="editEventType(${et.id})"
onclick="softDeleteEventType(${et.id},'${esc(et.name)}')"
onclick="restoreEventType(${et.id},'${esc(et.name)}')"

// After
onclick="editEventType('${et.id}')"
onclick="softDeleteEventType('${et.id}','${esc(et.name)}')"
onclick="restoreEventType('${et.id}','${esc(et.name)}')"
```

### What You Need To Do
Already deployed to GitHub Pages (commit `b9bb1bf`). No further action required.

---

## 10. venuepro_booking.html — Cost Calculation Hardcoded

### File
`venuepro_booking.html`

### Problem
The booking form displayed **£0.00** for total cost and did not calculate the room price, deposit, or full payment amount — regardless of which room or time slot was selected.

### Root Cause
`calculateCost()` used a hardcoded JavaScript object:

```javascript
const roomRates = {
    "Conference Room A": { hourly: 100, daily: 500, depositPct: 30 },
    "Banquet Hall B":    { hourly: 200, daily: 1000, depositPct: 40 },
    "Grand Ballroom":    { hourly: 250, daily: 2500, depositPct: 20 },
    "Executive Suite":   { hourly: 50,  daily: 400,  depositPct: 50 },
    "Garden Terrace":    { hourly: 100, daily: 800,  depositPct: 30 }
};
```

The function looked up rates by **room name string**:
```javascript
const rates = roomRates[room] || { hourly: 0, daily: 0, depositPct: 0 };
```

The rooms currently in the database have different names. Even if a name happened to match, this approach:
- Ignored the `day_rate` stored in the database entirely
- Ignored the `half_rate` field
- Ignored any **pricing grid overrides** (custom rates per room + event type)
- Used a fixed `depositPct` per hardcoded room — meaningless for real data

Additionally, the function matched on room **name** not **ID**, so any room name change would silently break pricing.

### Fix Applied

**Step 1 — Fetch live data on page load:**
```javascript
const ROOMS_API   = BASE_API + '/get-rooms';
const PRICING_API = BASE_API + '/get-pricing';

let roomsData   = [];
let pricingData = [];
let currentEventTypeId = null;

async function loadRoomsAndPricing() {
    const [rRes, pRes] = await Promise.all([
        fetch(ROOMS_API,   { headers: getAuthHeaders() }),
        fetch(PRICING_API, { headers: getAuthHeaders() })
    ]);
    roomsData   = (await rRes.json()).data || [];
    pricingData = (await pRes.json()).data || [];
}
```

**Step 2 — Capture event_type_id from the pending request:**
```javascript
// In fillDetails()
currentEventTypeId = req.event_type_id || null;
```

**Step 3 — Rewrite calculateCost() with real rate logic:**
```javascript
const DEPOSIT_PCT = 30; // 30% deposit across all bookings

function calculateCost() {
    const roomId = document.getElementById('roomId').value;
    // ... get start/end times, calculate duration ...

    // 1. Find the room object from live API data
    const room = roomsData.find(r => String(r.id) === String(roomId));

    // 2. Start with room's default day_rate
    let baseRate = parseFloat(room.day_rate) || 0;
    let rateSource = 'Default day rate';

    // 3. Check for event-type pricing override in the pricing grid
    if (currentEventTypeId) {
        const override = pricingData.find(
            p => String(p.room_id)        === String(roomId) &&
                 String(p.event_type_id)  === String(currentEventTypeId)
        );
        if (override && override.day_rate != null) {
            baseRate = parseFloat(override.day_rate);
            rateSource = 'Custom rate (event type override)';
        }
    }

    // 4. Apply duration-based pricing tier
    const halfRate = room.half_rate ? parseFloat(room.half_rate) : null;
    let cost = 0;

    if (duration >= 6) {
        cost = baseRate;                   // Full day rate
    } else if (halfRate && duration >= 3) {
        cost = halfRate;                   // Half-day rate (if set)
    } else {
        cost = (duration / 8) * baseRate;  // Pro-rated fraction of day
    }

    const deposit = cost * (DEPOSIT_PCT / 100);
    // ... update UI ...
}
```

### Pricing Tier Summary

| Duration | Rate Used |
|---|---|
| ≥ 6 hours | Full `day_rate` (or pricing override) |
| 3–5.9 hours (if `half_rate` set) | `half_rate` from room config |
| < 3 hours or no `half_rate` defined | Pro-rated: `(hours ÷ 8) × day_rate` |
| Any duration + event type override exists | Override `day_rate` from pricing grid |

Deposit is pre-filled as **30% of total cost** and is editable before submission.

### What You Need To Do
Already deployed to GitHub Pages (commit `630e865`). No further action required.

---

## Summary: Outstanding Actions Required (n8n Re-imports)

The following n8n workflow JSON files have been patched locally. They **must be re-imported** into your n8n instance to take effect:

| Workflow File | Fix Included | Status |
|---|---|---|
| `VenuePro - Staff Login (Native Crypto) (1).json` | Hardcoded pepper | ⬜ Re-import needed |
| `VenuePro - User Manager (Final Database Fix).json` | Hardcoded pepper | ⬜ Re-import needed |
| `VenuePro - Config Manager.json` | Role check removed, UUID parseInt fixed | ⬜ Re-import needed |
| `VenuePro - Complete System API (Status Fix) (2).json` | Sequential dashboard chain | ⬜ Re-import needed |

### How to Re-import a Workflow
1. Open n8n at `https://n8n.srv1090894.hstgr.cloud`
2. Go to **Workflows** → find and **deactivate** the old version of the workflow
3. Click **⋮ (three dots)** → **Import from File** → select the fixed JSON from `/Users/andrewjohnson/Downloads/VenuePro/`
4. For the Login workflow: click the `JWT: Sign` node and link your JWT credential
5. Click **Save** → toggle **Active**

### SQL Migration
Run `VenuePro - Config Migration.sql` in pgAdmin if not already done — it is idempotent (safe to re-run).

---

## Frontend Files — All Deployed ✅

| File | Change | Commit |
|---|---|---|
| `admin-config.html` | Half-day column, withRole() helper | `80e5723` |
| `enquiry-form.html` | Dynamic rooms/event types loading | `80e5723` |
| `admin-config.html` | UUID onclick quote fix | `b9bb1bf` |
| `venuepro_booking.html` | Live API cost calculation | `630e865` |

Live at: **https://andyjay72.github.io/VenuePro/**


---

## Session 2 — Later Same Day (17:14 onwards)

The following issues were identified and fixed during a second debug session after the initial deployment.

---

## 11. Make Booking Workflow — `total_amount` NaN Error

### File
`VenuePro - Make Booking (Platinum Fix) (1).json`

### Problem
After submitting a booking from `venuepro_booking.html`, n8n threw:
```
ExpressionError: Invalid input for 'total_amount' [item 0]
The value "total_amount" expects a number but we got 'NaN'
```
at `insert.operation.ts:234` (DB: Create Booking node).

### Root Cause
The database schema was migrated from `hourly_rate` / `daily_rate` / `deposit_percentage` column names to `day_rate` / `half_rate`. The `DB: Get Room` node in the workflow was never updated and still queried the old column names:
```sql
-- Old (broken)
SELECT id, name, hourly_rate, daily_rate, deposit_percentage
FROM bookings.rooms WHERE room_name ILIKE $1 LIMIT 1;
```
PostgreSQL returned `null` for all three missing columns. The `Code: Logic + Ref` node then did:
```javascript
const dailyRate = parseFloat(room.daily_rate);  // parseFloat(null) = NaN
calculatedTotal = dailyRate;                     // NaN
total_amount: calculatedTotal.toFixed(2)         // "NaN" (string)
```
`DB: Create Booking` received the string `"NaN"` for a column typed as `number` and rejected it.

There was also a secondary issue: `DB: Get Room` looked up by `room_name ILIKE $1` (a text string), but the webhook payload sends `room_id` (a UUID). This can fail silently for rooms whose names contain special characters or differ in casing.

### Fix Applied

**`DB: Get Room` — new SQL:**
```sql
SELECT id, name, day_rate, half_rate
FROM bookings.rooms
WHERE id = $1::uuid
LIMIT 1;
```
**queryReplacement:** `={{ [$json.body?.room_id || $json.room_id] }}`

**`Code: Logic + Ref` — rate logic rewritten:**
```javascript
const dayRate  = parseFloat(room.day_rate)  || 0;
const halfRate = room.half_rate ? parseFloat(room.half_rate) : null;

if (duration >= 6) {
  calculatedTotal = dayRate;        // Full day
} else if (halfRate && duration >= 3) {
  calculatedTotal = halfRate;       // Half day
} else {
  calculatedTotal = (duration / 8) * dayRate;  // Pro-rated
}

if (isNaN(calculatedTotal) || calculatedTotal <= 0) {
  throw new Error(`Rate calculation failed: day_rate=${room.day_rate}`);
}
// Output as actual number, not string
total_amount: parseFloat(calculatedTotal.toFixed(2)),
```

### What You Need To Do
Re-import `VenuePro - Make Booking (Platinum Fix) (1).json` into n8n.

---

## 12. Make Booking Workflow — `booking_date` Required but Not Set

### File
`VenuePro - Make Booking (Platinum Fix) (1).json`

### Problem
After fixing issue #11, the next error appeared:
```
ExpressionError: Invalid input for 'booking_date' [item 0]
The value "booking_date" is required but not set
```
at `insert.operation.ts:234` (DB: Create Booking node).

### Root Cause
The workflow chain through the booking path is:
```
Code: Logic + Ref -> DB: Clash Guard -> IF: Allow Booking? -> DB: Create Booking
```

`DB: Clash Guard` runs `SELECT COUNT(*) AS clashes ...`. When a Postgres node runs a SELECT, it **replaces `$json`** with its own query output — in this case `{ clashes: "0" }`. All fields output by `Code: Logic + Ref` (`booking_date`, `customer_id`, `room_id`, `total_amount`, etc.) were completely wiped.

`DB: Create Booking` was mapped using:
```
booking_date = {{ $json.booking_date }}  <- undefined after Clash Guard overwrites $json
```

### Fix Applied
Updated all 11 column mappings in `DB: Create Booking` to reference `Code: Logic + Ref` directly by node name, bypassing the overwritten `$json`:
```
customer_id        = {{ $('Code: Logic + Ref').first().json.customer_id }}
booking_request_id = {{ $('Code: Logic + Ref').first().json.request_id }}
room_id            = {{ $('Code: Logic + Ref').first().json.room_id }}
booking_date       = {{ $('Code: Logic + Ref').first().json.booking_date }}
start_time         = {{ $('Code: Logic + Ref').first().json.start_time }}
end_time           = {{ $('Code: Logic + Ref').first().json.end_time }}
total_amount       = {{ $('Code: Logic + Ref').first().json.total_amount }}
deposit_paid       = {{ $('Code: Logic + Ref').first().json.payment_amount }}
deposit_paid_date  = {{ $('Code: Logic + Ref').first().json.payment_date }}
balance_due        = {{ $('Code: Logic + Ref').first().json.balance_due }}
status             = {{ $('Code: Logic + Ref').first().json.booking_status }}
```
This is the same pattern already used correctly by `DB: Record Payment` and `DB: Close Request`.

### What You Need To Do
Re-import `VenuePro - Make Booking (Platinum Fix) (1).json` into n8n.

---

## 13. Make Booking Workflow — Request Reappears After Booking

### File
`VenuePro - Make Booking (Platinum Fix) (1).json`

### Problem
After a booking was successfully processed and payment taken, the booking request disappeared from the pending list — then reappeared a few seconds later. The booking and payment existed in the database but the request status remained `pending`.

### Root Cause
The workflow chain after `DB: Create Booking` was:
```
DB: Create Booking
  -> DB: Record Payment
  -> GCal: Create Event
  -> DB: Store Event ID
  -> DB: Close Request   <-- too late
  -> Email: Send
  -> Respond
```
`DB: Close Request` (`UPDATE booking_requests SET status = 'booked' WHERE id = $1`) was positioned **after** the Google Calendar call. If the GCal auth token had expired, or the event creation failed for any reason, the chain aborted at that point. The booking and payment were already committed to the database, but `DB: Close Request` never ran — leaving the request in `pending` status, causing it to reappear in the dropdown.

### Fix Applied
Reordered the chain so all critical database writes complete before any optional external service calls:
```
Before:
DB: Record Payment -> GCal: Create Event -> DB: Store Event ID -> DB: Close Request -> Email

After:
DB: Record Payment -> DB: Close Request -> GCal: Create Event -> DB: Store Event ID -> Email
```
Now the sequence is: booking created -> payment recorded -> **request closed** -> Google Calendar -> email. Failures in GCal or email are non-critical and do not affect the booking status.

### What You Need To Do
Re-import `VenuePro - Make Booking (Platinum Fix) (1).json` into n8n.

---

## 14. Make Booking Workflow — Pricing Surcharge Ignored by Workflow

### File
`VenuePro - Make Booking (Platinum Fix) (1).json`

### Problem
Even with a surcharge configured in the Admin Config Event Surcharge Grid (e.g. £250 for CHRISTENING), confirmed bookings were created at the base room rate with no surcharge applied.

### Root Cause
The `Code: Logic + Ref` workflow node recalculated `total_amount` from scratch using only `room.day_rate` fetched directly from the database. It had no knowledge of the `pricing_overrides` table and completely overwrote the `total_amount` that the frontend had already correctly calculated (including any applicable surcharges).

### Fix Applied
The Code node was simplified to **trust the frontend-calculated amounts** and only validate they are valid numbers:
```javascript
// No longer recalculates from room.day_rate
const total_amount = parseFloat(input.total_amount);
if (isNaN(total_amount) || total_amount <= 0) {
  throw new Error('Invalid total_amount received: ' + input.total_amount);
}
const payment_amount = parseFloat(input.payment_amount);
if (isNaN(payment_amount) || payment_amount <= 0) {
  throw new Error('Invalid payment_amount received: ' + input.payment_amount);
}
```
The frontend is the single source of truth for pricing logic (room rate + duration tiers + event surcharges). The workflow validates, persists, and triggers downstream actions.

### What You Need To Do
Re-import `VenuePro - Make Booking (Platinum Fix) (1).json` into n8n.

---

## 15. Event Surcharge Grid — Concept Redesign

### Files
`venuepro_booking.html`, `admin-config.html`

### Background
The Pricing Grid in Admin Config stores a `day_rate` value per room + event type combination. The original implementation treated this as a **replacement rate**: setting £250 for CHRISTENING would make the entire booking cost £250, replacing the room's normal rate entirely. This caused unexpected behaviour:

- A full-day booking of a £400/day room for a christening would cost **£250** (cheaper, not more expensive)
- Half-day / pro-rated logic was bypassed since the full rate was being replaced
- Staff had to enter the complete combined total rather than just the surcharge amount

### Change Made
Redesigned as an **additive surcharge**: the grid value is an extra fee added on top of the normal room rate, regardless of booking duration.

**New `calculateCost()` logic in `venuepro_booking.html`:**
```javascript
// Step 1: Base cost from room rate + duration tier (unchanged)
if (duration >= 6)           cost = dayRate;           // Full day
else if (halfRate && dur>=3) cost = halfRate;           // Half day
else                         cost = (dur/8) * dayRate;  // Pro-rated

// Step 2: Add event surcharge on top
const sur = pricingData.find(
    p => p.room_id == roomId && p.event_type_id == currentEventTypeId
);
if (sur && parseFloat(sur.day_rate) > 0) {
    cost += parseFloat(sur.day_rate);
    rateType += ` + ${formatGBP(surcharge)} event surcharge`;
}
```

**Example with room day_rate = £400, half_rate = £200, CHRISTENING surcharge = £250:**

| Duration | Before (replacement) | After (surcharge) |
|---|---|---|
| Full day (8hrs) | £250 | £400 + £250 = **£650** |
| Half day (4hrs) | £250 | £200 + £250 = **£450** |
| Pro-rated (2hrs) | £250 | £100 + £250 = **£350** |

The cost breakdown on the booking form now explicitly shows the surcharge component, e.g.:
> *Full Day + £250.00 event surcharge | 8.0 hrs | Suggested deposit 30% = £195.00*

### Admin Config Changes
- Section renamed from "Price Override Grid" to **"Event Surcharge Grid"**
- Input placeholder changed from the room's day_rate to `0` (blank = no surcharge)
- Help text updated to: *"Enter a surcharge amount (£) to add on top of the room's normal rate for specific event types. Blank = no surcharge."*

### What You Need To Do
Already deployed to GitHub Pages (commit `cc116b1`). No further action required.

---

## Updated Summary: Outstanding Actions Required (n8n Re-imports)

| Workflow File | Fixes Included | Status |
|---|---|---|
| `VenuePro - Staff Login (Native Crypto) (1).json` | Hardcoded pepper | Needs re-import |
| `VenuePro - User Manager (Final Database Fix).json` | Hardcoded pepper | Needs re-import |
| `VenuePro - Config Manager.json` | Role check removed, UUID parseInt fixed | Needs re-import |
| `VenuePro - Complete System API (Status Fix) (2).json` | Sequential dashboard chain | Needs re-import |
| `VenuePro - Make Booking (Platinum Fix) (1).json` | Issues #11-#14: NaN fix, booking_date fix, chain reorder, trust frontend totals | Needs re-import |

---

## Frontend Files — Session 2 Deployments

| File | Change | Commit |
|---|---|---|
| `venuepro_booking.html` | Additive surcharge pricing logic | `cc116b1` |
| `admin-config.html` | "Event Surcharge Grid" label + help text | `cc116b1` |

Live at: **https://andyjay72.github.io/VenuePro/**
