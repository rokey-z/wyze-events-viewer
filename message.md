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

The chat drawer has a hidden **Test** button that calls `_testProactiveMessage()`. The button rotates through types A → E (and an SMS preview as a 6th step), generating a message from real loaded data each time and pushing the result into the chat history. This is a UX preview, not a runtime engine.

---

## 11. SMS / MMS Alerts (out-of-app surface)

When something time-sensitive happens and the user isn't in the app, the agent escalates via SMS (or MMS when an image is available). SMS has no buttons, no formatting, no swipe-actions — so the message must be **self-contained, image-led, and answerable with a single digit**.

### Anatomy

Every SMS alert has exactly three parts, in this order:

```
1. [image]       — one MMS attachment of the relevant moment (or a contact-sheet
                   of up to 3 images if the alert involves multiple cameras)
2. message       — one or two short sentences. Must include WHAT happened,
                   WHERE, and HOW LONG. No marketing tone. No links.
3. reply menu    — numbered options the user can text back. Always 2–3 choices.
                   First option is the action the user is most likely to want.
                   Always include a "Show me" option if a richer view exists.
```

### Voice — write like a watchful housemate, not an app

The SMS should read as if someone who lives at home is texting you, not as an automated alert. Same data — warmer voice. Compare:

- ❌ App-notification: *"Garage door open for 47 min."*
- ✅ Housemate: *"Heads up — your garage door has been open since 10:14 PM, almost 50 minutes now. Just flagging in case you forgot."*

Six rules:

1. **Open with a connector, not a label.** *"Hey,"*, *"Heads up —"*, *"Quick heads-up —"*, *"Saw a …"*. Never `Wyze · Front Door:` inside the body — the sender header already says that.
2. **Anchor on real time AND duration.** *"since 10:14 PM, almost 50 minutes now"*. Readers latch onto either; both feel grounded.
3. **Use possessives.** *"your Tesla"*, *"your front porch"*, *"your Backyard 2 cam"*. Sounds like a roommate, not an alarm system.
4. **End with intent, not a command.** *"Just flagging."*, *"Thought you'd want to see."*, *"Worth a swap while it's still light out."*
5. **Reply labels are voice replies, not buttons.** *"got it, grabbing now"*, *"on it, closing now"*, *"all good, ignore"* — what you'd actually text back to a friend.
6. **No caps, no marketing, no exclamations.** A friend doesn't say "ALERT!".

### Style rules

- The sender header (e.g. `Wyze · Front Door:`) handles attribution. Don't repeat it inside the body.
- Soft length cap: ≤ 220 characters works well for a chatty MMS body. Hard cap on standard SMS length only matters if there's no MMS.
- Use a single emoji at the start to convey type at a glance: 📦 package, 🚪 garage, ⚠️ anomaly, 🚗 vehicle, 🦝 wildlife, 🔋 battery, 📷 offline.
- Do **not** include URLs in the SMS body. The "Show me" reply triggers a deep-link push back to the user.
- Quiet hours: an SMS only fires during quiet hours if confidence is High AND impact is irreversible (intruder, water leak, smoke). Everything else queues to morning.

### Reply protocol

Replies are single-digit responses (1, 2, 3). The agent service should accept any of:
- Just the digit (`2`)
- Digit + text (`2 not now`)
- Lowercase keyword that matches the action (`snooze`, `show`, `done`)

If the user replies anything else, the agent texts back a short re-prompt:
> *"Reply 1, 2, or 3. (1 = mark picked up, 2 = snooze 1h, 3 = show me)"*

### Templates (filled from real data)

Each template's three parts map to the bullets above.

#### Package on porch too long

```
[mms] front_porch_pkg_2026-04-29_14:02.jpg
Wyze · Front Door:
📦 Hey — UPS dropped a package on the front porch around 11:48 AM.
It's been sitting there for 3 hours now. Want me to ping you again
later?
Reply 1 got it, grabbing now · 2 remind me at 5 PM · 3 show me
```

Trigger: a delivery group with no later "picked up / carried in" event from the same camera within `cooldown` (default 90 min).
Confidence: High. Cadence cap: ≥ 2 h between repeat texts on the same package.

#### Garage door left open

```
[mms] garage_door_open_2026-04-29_22:47.jpg
Wyze · Garage:
🚪 Heads up — your garage door has been open since 10:14 PM,
almost 50 minutes now. Just flagging in case you forgot.
Reply 1 on it, closing now · 2 ping me again in 30 · 3 show me
```

Trigger: a garage open event with no later close event after the user-set threshold (default 30 min). Re-fire after another 30 min if no reply.
Confidence: High. Quiet-hours: allowed (impact = potentially overnight exposure).

#### First-time visitor at off-hours

```
[mms] front_door_anom_2026-04-29_02:14.jpg
Wyze · Front Door:
⚠️ Someone just walked up to your front door at 2:14 AM. First time
I've seen anyone there that late in the past 30 days — thought you'd
want to see.
Reply 1 all good, ignore · 2 tell me if it happens again · 3 show me
```

Trigger: a person event during the user's quiet window (00:00–05:00 by default) when baseline = 0 over the last 14 days at that hour.
Confidence: High. Impact: irreversible. Allowed in quiet hours.

#### Vehicle parked unusually long

```
[mms] driveway_tesla_2026-04-29_07:30.jpg
Wyze · Driveway:
🚗 Your Tesla's been in the driveway since 7:30 PM yesterday — 12.5
hours, about 4× longer than usual. Wanted to flag in case you forgot
to plug it in.
Reply 1 all good, plugged in · 2 flag me next time · 3 show me
```

Trigger: a vehicle has been parked in the same camera's view ≥ 4× the user's median dwell time. Daytime only.
Confidence: Medium. Cadence cap: 1 SMS per vehicle dwell event.

#### Camera offline

```
[no image — offline]
Wyze · Backyard 2:
📷 Quick heads-up — your Backyard 2 cam dropped off at 1:42 PM.
Battery was at 4%. Worth a swap while it's still light out.
Reply 1 on it, swapping battery · 2 ignore for now · 3 show me
```

Trigger: heartbeat missed > 10 min, OR battery <= 5% (whichever earlier).
Confidence: High. No image attachment (camera is offline). Allowed in quiet hours.

#### Wildlife: late-night raccoon

```
[mms] backyard_raccoon_2026-04-29_03:12.jpg
Wyze · Backyard:
🦝 Saw a raccoon out back at 3:12 AM, near the trash bins again.
Third time this week — looks like the same one.
Reply 1 noted, thanks · 2 tell me each time · 3 show me
```

Trigger: same wildlife species detected ≥ 3 times in 7 days.
Confidence: Medium. Cadence cap: 1 / day.

### Anti-spam ledger

Every SMS the agent sends is recorded under `localStorage["wyze:sms-log"]` with `{ type, ts, replied }`. Rules:

- **Hard cap**: 3 SMS / 24 h.
- **Per-type cap**: 1 SMS / type / 6 h, regardless of confidence.
- **No reply → escalate, then back off**: if the user doesn't reply within `cooldown`, send the same alert once more, then go silent on that type for 24 h.
- **Reply = success**: a reply (any digit) resets the cooldown and counts the message as "delivered intent met".

