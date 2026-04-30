# Wyze AI Proactive Messages — Agent Design Guide

> Goal: turn the AI from a passive answer-bot into an **opinionated, evidence-backed assistant** that speaks first only when it has something the user actually wants to hear.
>
> This file describes the message types, anatomy, and copy patterns the agent should use when composing a proactive message from real user data (`window.__rawGroups`, `window.__rawDevices`, `__SUBJECT_INDEX`, `__camNameMap`, `__SUBJECT_FEEDBACK`, `__chatHistory`).

---

## 1. The Five Message Types

Each type maps to a distinct user expectation, time-horizon, and primary CTA. Pick exactly **one** type per message. If two compete, choose the one with the **higher confidence and shorter time-horizon**.

| # | Type | Purpose | Time horizon | Primary CTA |
|---|---|---|---|---|
| **A** | **Recommendation** | "I noticed something — here's a setting that would help." | One-time decision | Accept · Dismiss · Not now |
| **B** | **Routine Reminder** | "Based on your pattern, a nudge at the right moment." | Recurring, time-anchored | Confirm cadence · Snooze |
| **C** | **Insight** | "Here's what I observed — no action needed, just awareness." | Read-only | Tap to expand · Share |
| **D** | **Feedback Request** | "Help me get smarter — confirm this for me." | Quick interaction | Tag · Confirm · Skip |
| **E** | **Anomaly Flag** | "Something deviated from your baseline — worth a glance." | Time-sensitive | Review event · Mark normal |

### Sample one-liners (copy tone)

- **A** Recommendation — *"You usually leave at 7:45 AM. Want vacation mode while you're away?"*
- **B** Routine Reminder — *"Trash truck tomorrow at 6:50 AM — bins out tonight?"*
- **C** Insight — *"176 dog-walkers passed by this week. Up 12% over last week."*
- **D** Feedback Request — *"Same Tesla I've seen 72 times — name this driver?"*
- **E** Anomaly Flag — *"First-time pedestrian at 3:14 AM on the side yard. Worth a look."*

---

## 2. Universal Message Anatomy

Every proactive message produced by the agent is a record with these fields:

```yaml
type:        A | B | C | D | E
title:       <= 50 chars, plain English, no clickbait
body:        <= 240 chars, two short sentences max
evidence:    string — the raw observation behind the claim
            ("Tesla detected 72× in the last 30 days")
confidence:  Low | Medium | High         # drives tone
surface:     home_banner | chat | inbox | push | event_detail
primary_cta: { label, action }            # single, clear
secondary:   [ "dismiss", "snooze", "tell me more", "not for me" ]
cooldown:    duration                     # min wait before re-surfacing
expiry:      timestamp                    # auto-remove after
data_min:    observation window required (e.g. "14 days")
```

### Field rules

- **evidence** — always present. Without it the agent feels mystical. Tapping it shows the raw events / numbers behind the claim.
- **confidence** — drives copy tone:
  - High: declarative — *"Your trash comes Thursday."*
  - Medium: hedged — *"Looks like trash comes Thursday — does that match?"*
  - Low: questioning — *"I'm not sure, but trash might come Thursday. Confirm?"*
- **cooldown** — *minimum* hours/days before a dismissed message of the same kind is shown again:
  - Recommendation (A): **7 days** after "Not now"
  - Reminder (B): use the cadence interval
  - Insight (C): never re-fire identical content; new week = new insight
  - Feedback (D): per-subject cooldown 24 h
  - Anomaly (E): 1 h, but only if same anomaly reoccurs
- **expiry** — Anomalies expire fastest (24 h). Insights live ~7 days. Recommendations live until accepted or dismissed.
- **data_min** — never fire below threshold:
  - Routine claims need ≥ 14 days
  - Anomaly comparisons need ≥ 7 days of baseline
  - Recommendation/Vacation need ≥ 5 days off-pattern

### Confidence calibration cheat-sheet

| Signal pattern | Confidence | Example |
|---|---|---|
| ≥ 90% match across ≥ 14 days | High | "Trash truck Thursday" |
| 60–90% match or < 14 days | Medium | "Looks like Thursday is trash day?" |
| < 60% match or one-off | Low | Don't fire — not worth the user's tap |

---

## 3. Insight Catalog (data → message)

Use this when picking *what* to surface. Each row pairs a derivable observation with the message-type it usually maps to.

### 3a. Routine & Rhythm (Type C / A)
- **Departures & returns** — "You leave 7:30–8:00 AM on weekdays. Friday you tend to leave 30 min later."
- **Home occupancy windows** — "Weekdays 8 AM–5 PM your home is typically empty."
- **Weekend shift** — "Saturdays start ~3 hours later than weekdays."
- **Routine stability score** — "Your weekday routine is 92% consistent over 30 days." (a single shareable number)

### 3b. Recurring services & people (Type C / D)
- **Service identification** — "Cleaning crew arrives every other Wed, ~10:45 AM, in a Nissan Rogue."
- **New regular** — "Someone in a silver SUV has appeared 5× in 2 weeks — add to known visitors?"
- **Schedule shift** — "Yard crew shifted from Saturday to Sunday this week."

### 3c. Vehicles & faces (Type D / C)
- **Top vehicles by frequency** — "Tesla (72), Silver SUV (57), Nissan Rogue (4 — cleaning)."
- **First-time vehicles** — "3 vehicles seen for the first time this week."
- **Unidentified faces** — "12 face detections we couldn't place — confirm any?" (D-type hook)

### 3d. Deliveries & packages (Type C)
- **Delivery cadence** — "Avg 4 packages/week. Peak day: Tuesday."
- **Carrier mix** — "Amazon 60%, UPS 20%, FedEx 15%, USPS 5%."
- **Linger time** — "Packages sit on porch avg 2.3 hours before retrieval."

### 3e. Neighborhood & foot traffic (Type C)
- **Walkability index** — "176 dog-walkers, 220 pedestrians this week."
- **Peak hours** — "Foot traffic peaks 7–9 AM and 3–5 PM."
- **Quiet windows** — "Lowest activity 1–4 AM."

### 3f. Wildlife & environment (Type C / E)
- **Animal sightings** — "Coyotes 3 nights this week, 2–5 AM."
- **Weather-tied events** — "Motion events spiked during Tuesday's storm."

### 3g. Coverage & system health (Type A / E)
- **Offline minutes** — "Garage cam was offline 4.2 hours this week."
- **Detection misses** — "3 events with low-confidence classification — review?"
- **Battery / signal trends** — "Front Door cam battery dropped 18% — ~5 days remaining."

### 3h. Comparison & change (Type A / E)
- **Week-over-week deltas** — "Activity ↓ 40% vs last week — vacation?" (vacation-mode trigger)
- **First-time-this-month events** — surface novelty without alarming
- **Streaks** — "12 days without a missed delivery alert."

---

## 4. Decision tree (which type to send)

```
1. Is the data anomalous vs the user's own baseline?
     yes → Type E (Anomaly Flag)
     no  → continue
2. Is there a recurring pattern with a stable cadence?
     yes & next occurrence is < 12h away → Type B (Routine Reminder)
     yes & not imminent                  → Type C (Insight)
3. Is there a setting / agent that would clearly benefit the user?
     yes & confidence ≥ Medium → Type A (Recommendation)
4. Is there a low-cost identification the user can confirm?
     yes → Type D (Feedback Request)
5. Otherwise → don't speak.
```

The agent should err on the side of silence. **One message per surface per day** is the cap unless an Anomaly justifies an interrupt.

---

## 5. Copy patterns

### Title (≤ 50 chars)
- Lead with the **verb or noun the user cares about**, not the type.
  - ✅ "Trash truck tomorrow at 6:50 AM"
  - ❌ "Routine Reminder: garbage pickup"
- Use numbers and proper nouns when you have them — they signal evidence.
  - ✅ "Tesla hasn't left the driveway in 6 days"
  - ❌ "Vehicle inactive for several days"

### Body (≤ 240 chars, max 2 sentences)
- Sentence 1 = the observation. Sentence 2 = the offer or question.
  - "Activity is down 40% vs last week and your Tesla hasn't departed. Want me to switch to vacation mode?"

### Evidence (≤ 80 chars)
- Always quantitative. Always tap-able.
  - "47 events over 14 days · tap to inspect"

### Primary CTA (≤ 16 chars)
- A verb. Match the type:
  - A → "Turn on", "Try it", "Set up"
  - B → "Remind me", "Confirm"
  - C → "See details", "Share"
  - D → "Yes, that's …", "Tag"
  - E → "Review event", "Mark normal"

### Secondary actions
- Always include **Dismiss** and **Not now** for A.
- Include **Snooze 1h / 1d / 1w** for B and E.
- C usually only needs **Tap to expand** — no negative path.

---

## 6. Surfacing rules

| Surface | Allowed types | Cap |
|---|---|---|
| Home banner | A, B, E | 1 at a time |
| Chat (proactive bubble) | A, B, C, D, E | 1 unread per session |
| Inbox | All | unbounded |
| Push notification | E only (with quiet hours respected) | 3 / day max |
| Event detail | D | 1 per event |

**Quiet hours** — pull `wyze:settings.quietStart` / `quietEnd` from `localStorage`. During quiet hours: **only Type E with High confidence may push**; everything else queues to morning.

---

## 7. Reusable templates (filled from real data)

The following templates are what the in-app `_testProactiveMessage()` button uses. Each picks data from `window.__rawGroups` + `window.__rawDevices` + `__SUBJECT_INDEX`.

### A — Recommendation: vacation mode
```
title:   "Activity ↓ {pct}% — try vacation mode?"
body:    "{n_events} events vs {n_last} last week, and your driveway has been still for {days} days. Want me to dial back alerts and watch for anomalies?"
evidence:"{n_events} events / 7 days · {top_camera} quietest"
confidence: High if pct ≥ 40 and days ≥ 5, else Medium
primary_cta: "Turn on"
secondary: ["Not now", "Dismiss"]
cooldown: 7d
```

### B — Routine Reminder: trash day
```
title:   "Trash truck tomorrow at {hh:mm}"
body:    "Looks like collection runs every {weekday} ~{hh:mm}. Want a heads-up the night before?"
evidence:"Detected on {n} of last {N} {weekday}s"
confidence: High if n/N ≥ 0.9
primary_cta: "Remind me"
secondary: ["Already do", "Stop"]
cooldown: cadence interval
```

### C — Insight: weekly recap
```
title:   "Last 7 days · {n_events} events"
body:    "Top subject: {top_subject} ({pct}%). Most active camera: {top_camera}. Quietest hour: {quiet_hour}."
evidence:"{breakdown}"
confidence: High
primary_cta: "See details"
secondary: ["Share"]
cooldown: 7d
```

### D — Feedback Request: identify a face/vehicle
```
title:   "Same {kind} I've seen {n}× — name them?"
body:    "Want to tag this {kind} so I can group future sightings?"
evidence:"{n} sightings across {span_days} days · last seen {when}"
confidence: Medium
primary_cta: "Yes, tag"
secondary: ["Skip", "Not the same"]
cooldown: 24h per subject
```

### E — Anomaly Flag: off-baseline activity
```
title:   "Unusual: {actor} at {hh:mm}"
body:    "First time anyone's been seen at the {place} between {window} in the last {N} days."
evidence:"Baseline: 0 events at this hour · 1 today"
confidence: High if z-score ≥ 3
primary_cta: "Review event"
secondary: ["Mark normal", "Not interested"]
cooldown: 1h
```

---

## 8. Don'ts

- **Don't** speak when confidence is Low and the action is non-trivial.
- **Don't** repeat dismissed Recommendations within their cooldown.
- **Don't** fire E (Anomaly) for a known recurring event you yourself surfaced as B.
- **Don't** pad messages with hedges ("It seems that maybe…"). Either you have a claim or you don't.
- **Don't** send during quiet hours unless type=E with High confidence.

---

## 9. Where to find data inside the app

| Field | Location |
|---|---|
| Groups (events) | `window.__rawGroups` |
| Devices | `window.__rawDevices` |
| Camera name map | `window.__camNameMap` |
| Subject index | `window.__SUBJECT_INDEX` |
| Per-subject feedback | `window.__SUBJECT_FEEDBACK` (right/wrong marks) |
| Quiet hours / settings | `localStorage["wyze:settings"]` |
| Existing chat history | `window.__chatHistory` |

---

## 10. Test hook

The chat drawer has a hidden **Test** button that calls `_testProactiveMessage()`. The button rotates through types A → E, generating a message from real loaded data each time, and pushes the result into the chat history with rich formatting (type badge · title · body · evidence chip · primary CTA · secondary actions · confidence dot). This is a UX preview, not a runtime engine.
