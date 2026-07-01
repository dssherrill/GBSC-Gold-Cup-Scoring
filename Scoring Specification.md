# Gold Cup Scoring Specification

Source of truth: [2022 GBSC Gold Cup Rules](https://www.soargbsc.net/gold_cup_contest)
(rules dated 8/27/2022 — re-verify before implementation if the page has been updated)

> **Terminology note:** The rules page calls this a "Turn Area Task" (TAT), which is
> the US soaring contest term for what the FAI calls an "Assigned Area Task" (AAT).
> Both terms mean the same thing: the pilot must enter **every** designated area to be
> scored as a completion. This is distinct from a Modified Assigned Task (MAT), where
> areas may be skipped. The Gold Cup is **not** a MAT — skipping either turn area
> results in a non-completion.
>
> **Note on WinScore:** WinScore's point-calculation formulas (CompletionRatio,
> ShortTaskFactor, ContestantFactor, MSP/MDP parameters) are designed for
> multi-pilot, single-day contest scoring. They do not apply here. The Gold Cup
> uses its own simpler season-relative formula, stated explicitly in Rule 7 and
> documented below. The WinScore source code is only relevant for the TAT/AAT
> optimal-distance algorithm and the handicap index lookup.

---

## 1. Contest season

- Opens: **1 April** each year
- Closes: **30 November** each year
- Season identifier: the calendar year (e.g. 2026)

---

## 2. Task definition (fixed for all seasons)

| Turnpoint | Type | Radius | Altitude constraint | Coordinates |
|-----------|------|--------|--------------------|-----------------------------------------|
| Sterling | Start cylinder | 4 mi | ≤ 5,500 ft MSL at start time | N42°25.570' W071°47.630' |
| Springfield (VSF) | Turn area | 45 mi | — | N43°20.460' W072°31.160' |
| Southbridge (3B0) | Turn area | 5 mi | — | N42°06.050' W072°02.330' |
| Sterling | Finish cylinder | 2 mi | ≥ 2,500 ft MSL at finish time | N42°25.570' W071°47.630' |

- **Direction:** either clockwise (Springfield first) or counter-clockwise
  (Southbridge first). Both are valid.
- **Minimum task time:** 2 hours 30 minutes (150 minutes).
- **Task distance range:** 82 mi (minimum) to 296 mi (maximum).

---

## 3. Handicap index

Source: [SSA Handicap List 2025](https://www.soaringspot.com/uploads/048/4848/files/Handicap_list_2025.pdf)

The handicap index `H` is a dimensionless number (close to 1.0) assigned per
glider model. A higher index indicates a faster glider.

Handicapped values are obtained by dividing raw values by the index:

$$\text{HDistance} = \frac{\text{ScoredDistance}}{H}$$

$$\text{HSpeed} = \frac{\text{ScoredSpeed}}{H}$$

This equalizes performance across glider types: a faster glider (higher H)
has its speed divided by a larger number.

Per club practice, the "without ballast" index is used for all flights, and no
adjustment is made for crew count or ballast state.

---

## 4. TAT/AAT optimal scoring distance

The pilot **must enter both turn areas** — failing to enter either area results in a
non-completion regardless of total distance flown. (The Gold Cup is not a MAT; no
areas may be skipped.)

Distance is awarded based on the **optimal scored path** through each turn area —
the path that maximizes total task distance while using only GPS fixes the pilot
actually reached.

**Algorithm:**

1. Parse the IGC B-records (GPS fixes) into a time-ordered sequence of
   lat/lon positions.
2. Identify the valid start fix: the first fix at or after the pilot's
   declared start time that is within the start cylinder and at or below
   the 5,500 ft MSL ceiling.
3. For each turn area in task order:
   a. Collect all GPS fixes that fall within that turn area's radius.
   b. For each candidate fix `P` in the turn area, compute the total
      task distance: `d(Start → P) + d(P → NextTurnArea_or_Finish)`.
      (Use great-circle / geodesic distances throughout.)
   c. The **scored turnpoint** for this area is the candidate fix that
      maximises this sum.
4. Identify the valid finish fix: the first fix that is within the finish
   cylinder and at or above the 2,500 ft MSL floor, occurring after the
   last scored turnpoint.
5. **Scored distance** = great-circle distance along the path:
   `Start → ScoredTP1 → ScoredTP2 → Finish`

**Notes:**
- If the pilot did not reach a turn area at all, the flight is a
  non-completion (see Section 6).
- The pilot may fly either direction; the scoring algorithm must try both
  orderings (Springfield→Southbridge and Southbridge→Springfield) and use
  whichever yields a valid path.
- All distance calculations must use geodesic (WGS-84 ellipsoid) distances,
  not flat-earth approximations, given the scale of the task (~100–300 mi).

---

## 5. Scored speed and the minimum-time rule

If the pilot completes the task in less than the minimum task time, they are
scored as if they flew for exactly the minimum time (Rule 7A elaboration):

$$\text{ScoredTime} = \max(\text{ActualFlightTime},\ 2.5\ \text{hours})$$

$$\text{ScoredSpeed} = \frac{\text{ScoredDistance}}{\text{ScoredTime}}$$

$$\text{HSpeed} = \frac{\text{ScoredSpeed}}{H}$$

---

## 6. Flight classification

| Classification | Criteria |
|---------------|----------|
| **Completion** | Pilot exited start cylinder, reached both turn areas (in either order), and entered the finish cylinder meeting the altitude requirement. |
| **Qualifying non-completion** | Did not complete the task, but flew > 50 **handicapped** miles (`HDistance > 50`). |
| **Non-qualifying non-completion** | Did not complete the task and flew ≤ 50 handicapped miles. Not scored (0 points). |

---

## 7. Points formula

### 7A. Speed points (completions)

Let `BestHSpeed` be the highest handicapped speed among all completions in
the current season.

$$\boxed{P_{\text{speed}} = 1000 \times \frac{\text{HSpeed}}{\text{BestHSpeed}}}$$

The pilot with `BestHSpeed` scores exactly 1,000 points. All other finishers
score proportionally less.

### 7B. Distance points (qualifying non-completions)

Let:
- `LowestSpeedPoints` = the lowest speed-point score among all completions
  this season = $1000 \times \dfrac{\text{LowestFinisherHSpeed}}{\text{BestHSpeed}}$
- `BestNonCompletionHDist` = the longest `HDistance` among all qualifying
  non-completions this season

**Maximum distance points anchor:**

$$\text{MaxDistPoints} = \max\!\left(500,\ 0.80 \times \text{LowestSpeedPoints}\right)$$

**Individual non-completion score:**

$$\boxed{P_{\text{dist}} = \text{MaxDistPoints} \times \frac{\text{HDistance}}{\text{BestNonCompletionHDist}}}$$

The pilot with the longest qualifying non-completion scores exactly
`MaxDistPoints`; all others score proportionally less.

**Edge case — no non-completion exceeds 50 handicapped miles (Rule 7B):**

$$\text{MaxDistPoints} = 500 \times \frac{\text{BestNonCompletionHDist}}{50}$$

$$P_{\text{dist}} = \text{MaxDistPoints} \times \frac{\text{HDistance}}{\text{BestNonCompletionHDist}}$$

which simplifies to:

$$\boxed{P_{\text{dist}} = 500 \times \frac{\text{HDistance}}{50} = 10 \times \text{HDistance}}$$

**Edge case — no completions yet in the season:**
`LowestSpeedPoints` is undefined; default `MaxDistPoints` to 500 until the
first completion is recorded.

### 7C. Gold Distance bonus (Rule 7C / Overview)

The FAI Gold Badge has three independently earnable legs: a 5-hour duration
flight, a 3,000 m altitude gain, and a **300 km distance flight**. A Gold Cup
flight that also qualifies as the pilot's Gold Distance leg earns a **10% bonus**
on that flight's points:

$$P_{\text{final}} = P \times 1.10$$

This applies to both speed-point and distance-point flights.

**What counts as a "valid Gold Distance leg":**
The FAI/SSA Gold Distance requirement is a flight of ≥ 300 km (186.4 mi).
For a closed-course Gold Cup flight, the qualifying distance is most likely the
total scored task distance (not a single leg). However, formal badge validity
is determined by the SSA, not this app — the pilot must submit separately to
the SSA for badge certification.

**Implementation approach:**
- The app checks whether the flight's scored distance ≥ 300 km (186.4 mi)
  and flags it as *potentially eligible* for the Gold Distance bonus.
- A checkbox on the submission form lets the pilot confirm they are claiming
  the bonus for this flight. The 10% is applied when checked.
- No automatic verification; the honor system applies, consistent with the
  rest of the Gold Cup rules.

---

## 8. Season standings

- Each pilot's **season score** = sum of their best 3 flight scores
  (highest 3 `P_final` values, across any number of submitted flights).
- A pilot with fewer than 3 flights scores the sum of all flights submitted.
- Scores are **season-scoped**: past seasons are archived and read-only.

---

## 9. Score storage and display strategy

The database stores only **raw flight values** — `ScoredHandicappedSpeed`,
`ScoredHandicappedDistance`, `ScoredDistance`, `ScoredTime`, `HandicapIndex`,
and flight metadata. No season-relative point values are persisted.

Season-relative points (the 1,000-point scale) are computed on the fly at
display/API time by:
1. Querying all flights for the season.
2. Deriving `BestHSpeed`, `LowestSpeedPoints`, and `BestNonCompletionHDist`
   from the stored raw values.
3. Applying the formulas in Sections 7A and 7B to each flight.
4. Selecting each pilot's best 3 results and summing for the season standings.

This eliminates any need to recompute stored values when a new flight changes
the season anchor. At single-club scale (tens of flights per season), computing
points at read time is trivially fast.

---

## 10. Open scoring questions (resolve before implementing points engine)

1. **"or 80% of the lowest speed points"** (Rule 7B): interpreted here as
   `max(500, 0.80 × LowestSpeedPoints)`. Confirm with contest director that
   this is the intended reading.
