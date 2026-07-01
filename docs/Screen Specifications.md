# Gold Cup — Screen Specifications

## Flight submission (2-step)

---

### Step 1 — File upload

**Purpose:** Accept an IGC file and detect duplicates before doing any
expensive parsing.

**Page content:**
- Title: GBSC Gold Cup Contest
- Subtitle: Flight submission form
- Brief instruction text: "Select an IGC flight log to upload."
- File picker — accepts `.igc` files only (enforce via `accept=".igc"` and
  server-side MIME/extension check).
- Submit button: "Upload and continue →"

**Server processing on submit:**

1. Reject if file extension is not `.igc` or content does not parse as a
   valid IGC file (return a user-friendly error on step 1, not step 2).
2. Compute SHA-256 hash of the uploaded file.
   - If the hash matches an existing submission by **any** pilot: show error
     "This exact file has already been submitted." Provide the name of the user who currently owns that flight along with the flight date and the submission date.  Also provide a link to the existing flight details (all users can view all flights).
3. Parse `HFDTE` from the IGC header to extract the flight date.
   - If the flight date falls outside the current contest season (1 Apr –
     30 Nov): show error "This flight date is outside the contest season."
   - If the logged-in pilot already has a submission for the same flight
     date: show warning (not a hard block) —
     "You already have a submission for [date]. Do you want to
     [update that submission] instead?" with a link. Pilot may dismiss
     and continue to submit a second flight for the same date (two Gold Cup
     attempts in one day is rare but permitted).
4. Store the uploaded file temporarily (associated with the session) and
   redirect to Step 2.

---

### Step 2 — Review, confirm, and submit

**Purpose:** Show the pilot what the app detected from the IGC file, let
them correct anything, display preliminary scoring results, and confirm
the submission.

The page is divided into two panels: **Flight details** (left/top) and
**Preliminary score** (right/bottom). The preliminary score updates
automatically when the pilot changes the glider or variant.

---

#### Panel A — Flight details (editable)

| Field | Source | Editable? | Notes |
|-------|--------|-----------|-------|
| Pilot name | Logged-in account | No | Display only |
| Flight date | `HFDTE` from IGC header | Yes | Allow correction if logger clock was wrong |
| Glider | Club fleet table (via `HFGID`) or fuzzy match on `HFGTY`; shown as a searchable dropdown of all handicap-table entries | Yes — required | Each entry in the dropdown is a unique glider/variant combination exactly as it appears in the SSA handicap table, e.g. "LS8 - 15m", "LS8 - 18m", "ASK 21 - solo", "ASK 21 - dual". Wing span variant and solo/dual occupancy are implicit in the selection. |
| Co-pilot name | Blank | Yes — required when a dual-occupancy entry is selected | Shown only when the selected handicap-table entry is a dual type. Free text. |
| Handicap index | Derived from selected glider entry | No | Read-only; updates automatically when the glider selection changes. |

**Gold Distance bonus:**
- Shown only if **both** conditions are met:
  1. The flight's scored distance ≥ 300 km (186.4 mi), **and**
  2. The pilot has not previously claimed the bonus (no other flight by
     this pilot currently has the bonus active, and the pilot's account
     profile does not have the "already earned" flag set — see Account
     profile screen).
- Checkbox: "I am claiming the 10% Gold Distance bonus for this flight."
  Unchecked by default.
- Helper text: "Check this box if this flight will be submitted to the SSA
  for your Gold Badge distance leg. Each pilot may claim this bonus at most
  once. The bonus is on the honor system."
- A pilot may un-claim the bonus by editing the flight and unchecking this
  box, which restores their ability to claim it on a different flight.

---

#### Panel B — Preliminary scoring results

Displayed after the IGC file is parsed. Labeled
**"Preliminary score — subject to change as other flights are submitted."**

| Item | Value |
|------|-------|
| Flight classification | Completion / Qualifying non-completion / Non-qualifying non-completion |
| Start time | HH:MM local (derived from IGC B-records and declared start) |
| Finish time | HH:MM local — shown only for completions |
| Time on course | H:MM — shown only for completions |
| Scored distance | X.X mi (raw) |
| Handicapped distance | X.X mi |
| Scored speed | X.X mph (raw) — shown only for completions |
| Handicapped speed | X.X mph — shown only for completions |
| Preliminary points | NNN (based on current season data; will change if a faster flight is later submitted) |

The start and finish times, and the scored distance, serve as a sanity check
that the pilot uploaded the correct file.

**Takeoff and landing:** The actual wheels-off and wheels-on times from the
IGC B-records may be shown as additional context ("Takeoff: HH:MM, Landing:
HH:MM") to help the pilot confirm they have the right file. These are
informational only and are not part of the scored result.

**Map display:** Plotting the flight path on a map with the task cylinders
overlaid would be the most effective sanity check of all. Deferred as a
future enhancement — not in initial build scope.

---

#### Submission

- "Submit this flight" button — disabled until all required fields are
  filled (glider resolved, variant confirmed if applicable, occupancy
  confirmed if two-seat, co-pilot name entered if dual).
- On success: redirect to the pilot's flight history page with the new
  flight highlighted and a confirmation banner.
- On failure (e.g. race condition where a duplicate was submitted between
  steps 1 and 2): show a clear error message and do not lose the pilot's
  step 2 entries.

---

## Pilot screens

### Leaderboard (public — no login required)

- Season selector (dropdown, defaults to current season).
- Table: one row per pilot, sorted by total season score descending.
  Columns: Rank, Pilot name, Season score, Flight dates contributing to score.
- Clicking a pilot name navigates to that pilot's detail page.

### Pilot flight detail page (public — no login required)

- Pilot name and season selector at top.
- Sort toggle: by score (descending) or by date (ascending).
- One row per submitted flight. Columns:
  - Flight date
  - Glider type
  - Handicap index
  - Classification (Completion / Non-completion)
  - Points (current season-relative value)
  - Raw distance (mi)
  - Handicapped distance (mi)
  - Raw speed (mph) — blank for non-completions
  - Handicapped speed (mph) — blank for non-completions
  - Time on course — blank for non-completions
  - Start time
  - Finish time — blank for non-completions
  - ★ marker on the up-to-3 flights counting toward the season score
- Pilot can edit or delete their own flights when logged in.
  - **Edit** goes directly to step 2 of the submission flow with all
    existing values pre-populated. The original IGC file is reused —
    no re-upload required. Only metadata (flight date, glider selection,
    Gold Distance bonus) can be changed.
  - **Delete** removes the flight permanently after a confirmation prompt.

---

## Account profile screen

Accessible to any logged-in pilot from a nav link. Editable fields:
- Display name
- Email address (triggers a new magic-link confirmation to the new address
  before the change takes effect)
- **"I have previously earned the FAI Gold Badge distance leg"** checkbox.
  - When checked, the Gold Distance bonus checkbox is permanently hidden
    from all future flight submissions for this pilot.
  - This covers pilots who earned the distance leg before this app existed.
  - An admin can also set or clear this flag on any pilot's behalf from the
    user management screen.

---

## Admin screens

### User management

- Table of all registered accounts. Columns: Name, Email, Role
  (Pilot / Admin), Registered date, Account status (Active / Deactivated).
- Actions per row: Deactivate / Reactivate, Promote to Admin / Demote.
- Separate tab or section: **Member email list** — the pre-loaded table of
  known club email addresses. Columns: Email, Date added, Added by.
  Actions: Add email, Remove email.

### Flight management

- Table of all submitted flights across all pilots and seasons.
  Columns: Date, Pilot, Glider, Classification, Points, Submitted at.
- Filters: season, pilot, classification.
- Actions per row: View detail (same as pilot detail page), Void flight,
  Delete flight.
- **Void** marks a flight as excluded from scoring without deleting the
  record. A voided flight remains visible to the pilot with a "Voided"
  label and a reason field (admin enters a brief note).
- **Delete** permanently removes the flight and its IGC file. Requires a
  confirmation step.
