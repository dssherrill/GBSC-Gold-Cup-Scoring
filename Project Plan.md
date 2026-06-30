# Gold Cup Automated Scoring & Leaderboard — Project Plan

## Status
Design/planning phase. No code written yet. This doc captures everything decided in
discussion so far, so an AI assistant (or a human) picking this up cold has full context.

## Background

The Greater Boston Soaring Club (GBSC) runs a season-long "Gold Cup" contest:
https://www.soargbsc.net/gold_cup_contest

Currently, scoring is done manually using WinScore (a Windows desktop GUI app from the
Soaring Society of America): https://www.gfbyars.com/winscore/

Goal: let a club member upload an IGC flight log via a web page and have that flight
automatically scored and added to a running leaderboard, without manual intervention.

## Key decision: do NOT use WinScore

WinScore was investigated as a potential scoring engine, including its "Autoscore" /
automated-results documentation
(https://www.gfbyars.com/winscore/QT_Auto_Scoring_Guide.pdf). Conclusion: **not viable**
for this project.

- WinScore is a Windows-only desktop GUI app. No API, no CLI, no headless mode.
- Its "automation" features are: (a) Gmail filters to route incoming pilot emails, and
  (b) a folder-watcher inside the GUI that auto-scores new IGC files dropped into a
  synced folder and uploads results — but only to the SSA's own contest website. It does
  not expose a generic webhook/API and has no concept of an arbitrary destination site.
- A human contest director is still required throughout: setup, resolving warnings,
  applying penalties, final review/re-upload.
- It is built for sanctioned multi-day regionals (start-gate penalties, airspace
  penalties, weather devaluation), which is much more machinery than the Gold Cup
  actually needs.

**Decision: reimplement the Gold Cup scoring rules directly in our own code.** The rules
are genuinely simpler than a sanctioned contest (see below), and this gives full control
over the submission pipeline and a clean web-native leaderboard.

## Gold Cup contest rules (from soargbsc.net)

- Format: Turn Area Task (TAT) — start cylinder, two named turn areas (Springfield,
  Southbridge), finish cylinder.
- Handicapped scoring ("Herold Handicapping" — see open question below on whether this
  is just the standard SSA index list).
- Each year starts a new season.
- Each pilot may submit any number of flights through the season.
- **best 3 flights of the season** count toward the pilot's total score for the season.
- Scoring is **season-relative**, not fixed-scale:
  - The fastest handicapped speed of the season so far = 1,000 points. All other
    completed-task flights are scored as a percentage of that.
  - Non-completions (didn't finish, but flew > 50 handicapped miles) are scored relative
    to the longest qualifying non-completion of the season — 500 points or 80% of the
    lowest speed points (whichever rule applies; re-check exact wording against the rules
    page when implementing).
  - **Important implication:** because every score is relative to the current
    season-best, submitting a new flight can change the "anchor" and require
    recalculating points for every other flight already in the system, not just the new
    one. The scoring engine must treat this as the normal case, not an edge case.
- Results for past seasons will be viewable in the same way as the current season.
- GPS traces must be in IGC format (or converted to it). This is the only data input
  format we need to support.
- Digital signatures in IGC files will not be verified since Gold Cup rules state that
  IGC files "need not be secure".  That said, validation would be easy using the web 
  server at http://vali.fai-civl.org/webservice.html
- A pilot need not fly the same glider every flight, and need not own the glider flown
  (club gliders are encouraged). This means **glider/handicap selection happens per
  flight, not per pilot** — there's no fixed "pilot's glider" to default to.

Action item: re-read the actual rules page closely when implementing the points formula
— summarized from memory of the discussion above, not re-verified line by line here.

## Data management
- The original IGC files are retained.
- The DB schema includes a column to identify the contest season, which is always 
  the year in which the flight occurred.

## Handicap data

Source: SSA/Soaring Society handicap index list (2025), same list WinScore itself ships
with:
https://www.soaringspot.com/uploads/048/4848/files/Handicap_list_2025.pdf

- This is a glider **model** → index lookup table (not per-pilot), with separate
  "without ballast" and "with ballast" index numbers per glider, plus wingspan and MTOW.
- Some airframes (e.g. ASK-21) have **separate dual vs. solo index rows**.
- Some airframes can be flown with different wing lengths. By far the most common case
  is 15-meter or 18-meter wingtip extensions.  The handicap depends on the wing length.
  Each variant must have a separate entry in the handicap table.  When submitting an IGC
  file, the pilot must confirm both the glider and the variant flown, both of which are typically evident in
  the IGC file.

### Working assumption (NOT YET CONFIRMED — verify with current Gold Cup scorer)

Based on the pilot's (David's) personal experience submitting flights as a contestant:
he has never been asked about ballast state or solo/dual occupancy when submitting a
log. Inference: **the club's actual practice does not adjust handicap for ballast or
occupancy** — it likely just uses a single canonical index per glider type (probably the
"without ballast" number).

**Design consequence:** build the handicap table with a single index per glider type by
default. For split-row gliders like the ASK-21, flag them explicitly and get a direct
answer from the club's contest director on which number (dual or solo) is actually used
in practice, rather than guessing. Don't build ballast-state or crew-count prompts into
the submission form unless/until this assumption is contradicted.

### Glider identification from IGC files (confirmed approach)

Real sample IGC header (from David's own flight, XCSoar logger):

```
HFDTE280626
HFFXA050
HFPLTPILOTINCHARGE:David Sherrill
HFCM2CREW2:
HFGTYGLIDERTYPE:ASK-21 (PIL)
HFGIDGLIDERID:N421GB
HFCIDCOMPETITIONID:GB
HFFTYFRTYPE:XCSOAR,XCSOAR Android 7.44
HFGPS:LX
```

Relevant fields:
- `HFGTY` (GLIDERTYPE) — free-text glider type string. Reliably present on modern
  loggers, but it's free text (e.g. has trailing tags like `(PIL)` that need stripping)
  and won't exactly match the handicap list's spelling/spacing conventions.
- `HFGID` (GLIDERID) — tail number. Useful for matching against a small **club fleet
  table** (tail → canonical type), which is more reliable than fuzzy-matching free text
  for the gliders the club actually owns.
- `HFCM2CREW2` — present-but-empty in solo flights. *Originally considered using this to
  auto-detect solo vs. dual, but per the ballast/occupancy assumption above, this is
  probably unnecessary — keep as a possible future signal only if that assumption turns
  out to be wrong.*

**Resolution strategy for glider → handicap index, in priority order:**
1. Tail number (`HFGID`) matches a known entry in a club fleet table → use that glider's
   canonical index directly.
2. Otherwise, fuzzy-match `HFGTY` (after stripping parenthetical suffixes, normalizing
   case/dashes/spacing) against the handicap list.
3. **Always show the pilot the detected glider + index as an editable field at
   submission time**, regardless of which tier matched, so they can correct it if it's
   wrong or missing. This is the single human-in-the-loop checkpoint for the whole
   pipeline — no separate manual review step needed after the fact.

## User authentication

A login account is required to submit flights. Authenticated pilots can submit and
modify their own flights only; admins can add, modify, delete, and void all flights.

**Registration flow:**
- The app maintains a pre-loaded table of known club member email addresses.
- Self-service registration: pilot enters their email address on the registration page.
- A time-limited magic link is emailed to that address to complete registration
  (set display name and password). No temporary passwords — the magic link is the
  one-time credential.
- Email with an address **not** in the known-members table is allowed to register,
  but the app immediately sends a notification email to all admins flagging the
  unknown address. The account is active and usable right away; the admin
  notification is informational, not a gate. This minimizes friction for legitimate
  members whose email wasn't pre-loaded, while keeping admins aware.

**Email infrastructure:**
- Outbound email (magic links, admin notifications) is sent via a transactional
  email service (SendGrid, Mailgun, or Postmark — to be chosen at build time)
  configured via Railway environment variables. No SMTP credentials in source code.

**Admin tasks related to the member email list:**
- Add new members' email addresses when they join the club.
- Update an address when a member's email changes.
- Remove departed members (prevents future registrations from those addresses, but
  does not affect existing accounts).
- Review admin-notification emails for unknown-address registrations and deactivate
  any accounts that appear illegitimate.

## Architecture decision

**Standalone Python web app, deployed on Railway, NOT a Drupal module.**

Rationale:
- David does not have full access to soargbsc.net (Drupal). Building standalone 
  means the leaderboard can launch and be useful independent of ever getting Drupal 
  access.
- David already runs other Python projects on Railway — matches existing operational
  pattern, no new platform to learn.
- Python's ecosystem is well-suited to the actual hard part (IGC parsing, geodesic /
  turn-area geometry).
- Drupal can't run Python natively anyway, so "integrate into Drupal later" was never
  going to mean "becomes a Drupal module." The realistic later-integration path is:
  Drupal embeds this via iframe, or a Drupal block/view fetches JSON from this app's API
  and renders it in Drupal's own theme. Keeping a clean HTTP API (not just
  server-rendered pages) preserves that option without adding meaningful cost now.

**Proposed stack:**
- Backend: Python, FastAPI
- Database: Postgres (Railway has first-class support)
- Frontend: plain HTML/CSS/JS or simple server-rendered templates — kept lightweight and
  portable so it could later be iframed into Drupal or restyled to match its theme
- Expose both: (a) a JSON API (submit flight, fetch leaderboard) and (b) a normal
  server-rendered leaderboard page — so it's a complete usable product standalone, AND
  embeddable/consumable from Drupal later with reasonable effort.

## Open questions / action items before or during implementation

1. **Confirm with the club's current Gold Cup scorer:**
   - Is "Herold Handicapping" actually just the SSA index list above, or a distinct
     published table?
   - Confirmed: no ballast adjustment, no dual/solo adjustment? (Working assumption above
     — verify.)
   - For split-row gliders (ASK-21 dual vs. solo, and check the list for any others),
     which number does the club actually use today?
2. **Re-verify the exact points formula** against the live rules page when implementing
   — all rules should be quoted/checked precisely, not relied on from this summary.
3. **Club fleet table** — need tail-number → canonical glider type mapping for GBSC's own
   gliders, to support Tier 1 of the glider resolution strategy. Does not yet exist;
   needs to be built (probably a short hand-maintained list).
4. **Recalculation strategy when the season-best "anchor" flight changes** — discussed
   but not finally decided. Leaning toward "recompute all stored flights live on every
   new submission" since contest scale (single club, one season) makes this cheap; batch
   recompute is the fallback if that's ever untrue.  Alternatively, keep raw scores for
   individual flights and only use 1000-point relative scoring for each pilot's net 
   score.

## Build order (proposed, not started)

1. Stand up the Railway app skeleton + DB schema
2. Handicap lookup table — extract the PDF into clean CSV/JSON; normalize glider-type
   strings for fuzzy matching; flag dual/solo-split rows for club confirmation.
3. IGC parser — header records (`HFGTY`, `HFGID`, `HFDTE`, pilot) + B-records (lat/lon/
   alt/time fixes).
4. Turn-area task scoring geometry — start cylinder, two turn areas, finish cylinder,
   min time/altitude rules, optimal-distance-within-area logic.
5. Points engine — handicapped speed/distance → season-relative points; best-3-of-N per
   pilot 
6. Minimal web UI to exercise the above (upload IGC, see computed score) before building
   out full accounts/leaderboard/Railway deployment.

## Source documents referenced

- Gold Cup rules: https://www.soargbsc.net/gold_cup_contest
- WinScore overview: https://www.gfbyars.com/winscore/
- WinScore Autoscore guide (why we're not using it):
  https://www.gfbyars.com/winscore/QT_Auto_Scoring_Guide.pdf
- Handicap list (2025): https://www.soaringspot.com/uploads/048/4848/files/Handicap_list_2025.pdf
