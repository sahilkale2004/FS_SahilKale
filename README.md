# FS_SahilKale
Student commute optimizer (full Stack)

# 🚗 Student Commute Optimizer — Creative Design & Presentation

> **Elevator pitch (1 sentence):**
> A privacy-first, student-focused carpooling app that finds ride partners by matching real routes and times — fast, friendly, and built to keep students safe while cutting travel cost and congestion.

---

## ✨ What I improved (creative highlights)

* Polished executive summary and elevator pitch for judges/interviewers.
* Reworked architecture into visually clear diagrams (textual + sequence flow) so reviewers can quickly understand components and data flow.
* Added a polished UI mockup (ASCII + descriptions) and a step-by-step storyboard showing a student finding a match.
* Included a compact PostGIS schema and example queries for the matching core (copy-paste ready).
* Added a scoring explainer with trade-offs, and a short one-page checklist you can use in interviews.

---

## 1. Executive summary (judge-friendly)

**Student Commute Optimizer** is a mobile-first app that helps students discover and coordinate shared rides by matching overlapping routes and compatible times. It is designed to be: **private** (pseudonyms + coarse location), **accurate** (route geometry matching with PostGIS), and **fast** (caching + corridor search). The MVP focuses on onboarding via college email, seamless route creation, real-time match discovery, and in-app anonymous chat.

**Key value proposition for reviewers:** reduces vehicle count, cuts student commute costs, and demonstrates practical geospatial engineering (PostGIS) and real-time features (WebSocket). Good technical depth while respecting privacy.

---

## 2. Architecture — visual (ASCII)

<img width="577" height="463" alt="image" src="https://github.com/user-attachments/assets/ef717001-5ca1-400e-807a-59dded405ca6" />


**Notes:** the Match Service uses PostGIS spatial indices for corridor and overlap checks; it queries Redis for active, in-memory trip info to keep latency low.

---

## 3. Sequence flow — find & chat (short)

1. Student A creates a trip (origin, destination, depart window). Frontend POSTs to `/trips/create`.
2. API stores trip in Postgres (route geometry returned from Routing API) and pushes job to `matching` queue.
3. Match worker reads job → queries PostGIS for ST\_DWithin candidates → computes overlap scores → writes top matches to Redis and notifies Student A via push/WebSocket.
4. Student A opens a match → WebSocket channel created (chat room tied to match id) → anonymous chat begins.

---

## 4. UI mockups & storyboard

**Map Screen — compact mockup (text)**

```
[Top bar]  SEARCH: From [Home]  To [College]

[Map canvas]
  ┌────────────────────────────────────┐
  │  --- polyline showing route ---->  │
  │  ○ (Pseudonym_21)  ○ (Pseudonym_87) │
  └────────────────────────────────────┘

[Matches list]
 ─ Pseudo: "BlueFalcon"   Overlap: 72%  Time: 8:00 - 8:15  [Chat]
 ─ Pseudo: "RouteRaja"     Overlap: 64%  Time: 7:45 - 8:05  [Chat]
```

**Storyboard (3 panels)**

* Panel 1: Onboard with college email → choose pseudonym `BlueFalcon` (system suggests alternatives if taken).
* Panel 2: Enter route & time → app shows matches and overlap %; tap match to preview pickup suggestion.
* Panel 3: Start in-app chat → confirm pickup spot & recurring schedule.

**Design cues to highlight in interview:** rounded-card UI, clear overlap badge, colorless icons (privacy), subtle safety badge (college-verified), and microcopy to reassure users ("Your identity stays private until you reveal it").

---

## 5. PostGIS schema (compact, copy-ready)

```sql
-- Enable PostGIS (run once)
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  college_id TEXT,
  pseudonym TEXT UNIQUE NOT NULL,
  real_name_hash BYTEA,
  profile_meta JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE trips (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  origin GEOGRAPHY(POINT, 4326),
  destination GEOGRAPHY(POINT, 4326),
  route_geom GEOMETRY(LINESTRING, 4326),
  depart_start TIMESTAMPTZ,
  depart_end TIMESTAMPTZ,
  seats_available INT DEFAULT 1,
  status TEXT DEFAULT 'planned',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Spatial index for fast corridor queries
CREATE INDEX idx_trips_route_geom ON trips USING GIST (route_geom);
CREATE INDEX idx_trips_origin ON trips USING GIST (origin);
```

---

## 6. Key PostGIS queries (core matching)

**Find candidates within corridor (\~500 meters)**

```sql
SELECT id, user_id, ST_AsText(route_geom) as route_wkt
FROM trips
WHERE status = 'planned'
  AND ST_DWithin(route_geom::geography, ST_GeomFromText($1,4326)::geography, 500)
  AND id != $2
LIMIT 200;
```

**Compute overlap length with a candidate (SQL snippet)**

```sql
SELECT ST_Length(ST_Intersection(a.route_geom::geography, b.route_geom::geography)) AS overlap_m
FROM trips a, trips b
WHERE a.id = $1 AND b.id = $2;
```

Use these to compute `overlap_ratio = overlap_m / LEAST(a_len, b_len)` where `a_len = ST_Length(a.route_geom::geography)`.

---

## 7. Compact matching pseudocode (interview-ready)

```
def findMatches(trip_id):
    new_trip = db.load(trip_id)
    candidates = postgis_find_candidates(new_trip.route_geom, corridor_meters=500)
    matches = []
    for c in candidates:
        overlap_m = compute_overlap_meters(new_trip.id, c.id)
        overlap_ratio = overlap_m / min(new_trip.length_m, c.length_m)
        time_score = time_window_overlap(new_trip.start, new_trip.end, c.start, c.end)
        prox_score = pickup_proximity_score(new_trip.origin, c.origin)
        score = 0.6*overlap_ratio + 0.3*time_score + 0.1*prox_score
        if score > 0.35:
            matches.append({c.id, score})
    return sorted(matches, key=score, reverse=True)[:10]
```

**Tip for interviewers:** explain why weights (0.6/0.3/0.1) — overlap is most important for ride-sharing success.

---

## 8. Scoring — quick math explanation (showing trade-offs)

* If routes overlap 50% (0.5), time window overlap 0.8, proximity 0.5 → `score = 0.6*0.5 + 0.3*0.8 + 0.1*0.5 = 0.3 + 0.24 + 0.05 = 0.59` — a strong match.
* Judges will appreciate a clear threshold rationale (e.g. `>0.35` accepts reasonable detours but avoids weak matches).

---

## 9. Safety & privacy one-pager (for judges)

* **Pseudonym + College verification:** ensures each account is linked to a real student but identity is hidden.
* **Minimal location sharing:** discovery coordinates rounded to 50m to avoid precise tracking.
* **Mutual reveal:** identity only revealed when both parties explicitly consent.
* **Moderation:** one-click reporting, automated filters, manual review pipeline.

---

## 10. Interview cheat-sheet (use this in 2-min demo)

1. Elevator pitch (15s) — use the sentence at the top.
2. Architecture snapshot (30s) — point to PostGIS + MatchSvc + ChatSvc.
3. Core algorithm (45s) — explain corridor search + ST\_Intersection + scoring weights.
4. Privacy & safety (20s) — pseudonyms, rounding, college verification.
5. Next steps / scale plan (10s) — caching, worker queues, sharding.

---

## 11. Optional polished deliverables I can produce next (pick one)

* **Detailed Figma-style wireframes** (I will produce annotated, step-by-step UI specs).
* **Full SQL + migration scripts** (complete DDL and indexes for Postgres/PostGIS).
* **Node.js Match Worker** (production-ready snippet showing queue, PostGIS calls, scoring logic).
* **Polished architecture diagram (PNG or SVG)** you can include in slides (I will generate a downloadable SVG).

---

### Final note

I updated this canvas to be presentation-ready and interviewer-friendly. If you want, I can now **generate a downloadable SVG architecture diagram** or **scaffold the Node.js match worker** — tell me which and I’ll produce it next.
000000000000000000000000000000000
