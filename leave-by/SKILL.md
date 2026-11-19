---
name: leave-by
description: 'Adds a leave-by block before every calendar event that has an address: travel time, when to leave, alternatives. Use for commute planning or "when do I need to leave for X".'
---

# Leave-By - Automatic commute-time calendar blocks

A tightly scoped job: keep the calendar so that in front of **every** event with a real
location there is a travel block, with honest travel time, a departure time, and
alternatives. It runs **quietly at night in the background**, so a companion morning
briefing (if you use one) only has to *read* the blocks instead of computing them.

No messages are sent to the user. At the end it prints a short run summary
(created / updated / skipped / cleaned up) as a log for you, not as a notification.

---

## First-time setup (run this before the first real run)

Trigger this flow when **any** of these is true:
- No configuration exists yet (see "Configuration" below), or
- The user says "set up leave-by", "configure the commute planner", "change my commute settings".

Run the setup as a **short form**, not an interview. If the assistant has a structured
multiple-choice question tool (for example `AskUserQuestion`), use it: tappable options,
the recommended default first and marked "(Recommended)", at most 4 questions per round.
**Every question must also offer an "Other" option** where the user can type their own
value. If the tool adds "Other" automatically, rely on that; otherwise add an explicit
"Other (type your own)" option. If no such tool exists, ask the same questions in plain
text, with the default named in each question.

**Step 0 - Home address (plain text, required).** Ask for the full address every commute
is measured from. This is free text, so it is not a multiple-choice question. There is no
default. Derive two things from it instead of asking:
- the **city / region**, for routing searches
- the **timezone**, as an IANA name (e.g. a Madrid address → `Europe/Madrid`, a Canary
  Islands address → `Atlantic/Canary`). Do not take it from the calendar's settings: those
  are often left on an old location. Never ask the user for an IANA name.

Show both in the final confirmation in plain words (e.g. "Madrid time, UTC+1"), so the user
can correct them if they are wrong.

**Round 1 (form, 3 questions):**
1. **Which calendar** - options: `primary` (Recommended) / Other (type a calendar name).
2. **Travel modes you use** - multi-select: `walking`, `transit`, `taxi/rideshare`,
   `driving (own car)` / Other. Default selection: walking, transit, taxi.
3. **Arrival buffer** - ask it in plain words: "How many minutes early do you want to
   arrive?" Explain in the option text that this is added on top of the travel time
   (example: event at 18:00, 20 min travel, 10 min early → leave at 17:30).
   Options: `5 min early` / `10 min early` (Recommended) / `15 min early` / Other.

**Round 2 (form, 4 questions):**
4. **How far ahead to plan** - "How many days ahead should blocks be added?"
   Options: `30 days` (Recommended) / `90 days` / `180 days` / Other.
5. **Walking threshold** (prefer walking if the walk is at most this long) - `15` / `30`
   (Recommended) / `45` min / Other.
6. **Color of the commute blocks** - ask: "Which color should your travel blocks have in
   your calendar?" Explain in one line that they get their own color so they stand out from
   real events at a glance, and so the skill can recognize its own blocks. Tell the user to
   pick a color they do not use for anything else. Options: `Sage / soft green` (Recommended,
   colorId `2`) / `Graphite / grey` (colorId `8`) / `Lavender / light purple` (colorId `1`) /
   Other (name a Google Calendar color). If the chosen color is already common in the user's
   calendar, warn once and suggest another.
7. **When should it run?** - ask: "How often should your travel blocks be updated?"
   Options: `Every night` (Recommended, e.g. 03:00) / `Every night + midday top-up` (catches
   events added during the day) / `Once a week` (Sunday night) / `Only when I ask`. Other
   (type a time or rhythm).

**Setting up the schedule.** After the form, if the answer is anything but "Only when I
ask", create the scheduled task(s) yourself with the scheduling tool this assistant has
(look for one with `tool_search("schedule")` or similar). Use the user's timezone and a
prompt such as `Run the leave-by commute planner.` For the midday top-up, add a second task
around 12:00 with `Run the leave-by commute planner for the next 2 days.` Save the chosen
rhythm as `schedule` in the config. If no scheduling tool exists, say so in one line and
show the user the exact prompt and time to set up themselves. Never create a duplicate
schedule: if one already exists for leave-by, update it instead.

Validate "Other" answers before saving: a number for minutes and days, an existing
calendar for the calendar. If one is invalid, re-ask only
that question.

After the user confirms, **write the configuration** (see next section) and run one full
pass immediately so they see it working. Confirm in one line: the home address, the derived
city and timezone in plain words, the calendar, the schedule, and how many blocks were created.

---

## Configuration

Store these settings where this assistant persists user settings - a memory entry named
`leave-by-config`, or a `config.json` in this skill's folder (see `config.example.json`).
On every run, **load the config first**. If none exists, run First-time setup instead of
guessing.

Settings and defaults:

| Key | Meaning | Default |
|-----|---------|---------|
| `home_address` | Start address for all commutes | (required) |
| `timezone` | IANA timezone | from `home_address` |
| `city` | City/region for routing searches | from `home_address` |
| `calendar` | Which calendar to maintain | `primary` |
| `buffer_minutes` | Arrive-early buffer | `10` |
| `scan_days_ahead` | How far ahead to plan | `30` |
| `travel_modes` | Modes to consider | `["walking","transit","taxi"]` |
| `walking_threshold_minutes` | Prefer walking if walk ≤ this | `30` |
| `schedule` | When it runs: `nightly`, `nightly+midday`, `weekly`, `on-demand` | `nightly` |
| `block_color_id` | Google colorId for the skill's blocks | `2` (Sage) |

Tools, loaded as needed: `tool_search("calendar events")` → list/create/update/delete_event ·
`web_search` / `web_fetch` for travel times · `tool_search("weather")` only if needed for
mode choice.

---

## Nightly run - the main contract

Runs as a scheduled task (the prompt contains "leave-by"/"commute"/"nightly", no chat
history) or on demand ("update my commute blocks"). **Zero questions** - decide everything
up front, work idempotently. Steps:

1. **Load config.** If it is missing: in a live chat, run First-time setup and continue. In
   an unattended scheduled run, ask nothing and do not guess a home address: print
   `❌ No configuration - say "set up leave-by" first` and stop.
2. **Fetch events:** `list_events` on the configured calendar from now to +`scan_days_ahead`,
   sorted by start time, paginated.
3. **Run the skip check** (below) per event. If it does not fall out → it's a block target.
4. **Reconcile existing commute blocks.** Find the skill's own blocks (configured color,
   🚶/🚖/🚇 title, and a `leave-by: event=<id>` line in the description) and match each one
   to its event by that id, never by time proximity:
   - A matching block exists and is still correct (event not moved) → **do nothing** (skip).
   - Event was moved → **update the block** (new time, re-check travel time if needed).
   - Event deleted/cancelled, block orphaned → **clean up your own block** (`delete_event`,
     only the colored commute block the skill created, never a real event).
   - No block yet → **create one** (step 4 of "Build the block" below).
5. **Print a run summary**, compact: `created: N · updated: N · skipped: N · cleaned: N`,
   plus one line per create/update (`✅ [event] [date] → leave home HH:MM`) and one per
   unresolved location (`❓ Location unclear: [event] [date]`).

**Why idempotency matters:** the job runs every night over the same window. Without a clean
reconcile (step 4) it would multiply blocks. Always check whether a correct block already
exists before acting.

**Same-day gap and the midday top-up.** Events created during the day are only planned at
the next run, which is too late for an event later that same day. Recommended: schedule a
second, lighter run around midday with the same contract (it may limit the window to the
next 2 days). Because the job is idempotent, it only adds what is missing.

---

## 1. Skip conditions (no commute block)
- Event is in the past (start < now).
- Location is virtual: "online", Zoom, Google Meet, Teams, Virtual, Remote, Call, or a
  meeting link.
- All-day event · cancelled / status ≠ "confirmed".
- A block for this event already exists (matched by its `leave-by: event=<id>` line) →
  reconcile it instead of creating another. A travel event the user made themselves (title
  with 🚶/🚖/🚇 / "Commute" / "Travel", no marker) within 2 h before it → treat it as already
  covered and never touch it.

**Missing / vague location** (only a person's name, a district with no address, "café
somewhere"): do not guess. Try a quick `web_search`; if still unclear → **do not guess**,
flag it in the run summary as `❓ Location unclear: [event] on [date]` (no block).

---

## 2. Travel times - source hierarchy
From the home address to the destination for **walking**, **taxi/rideshare**, **transit**
(and **driving** if enabled). Take the first usable result.

**⚠️ Never use straight-line distance as travel time.** If only a straight-line distance is
known, multiply it by about 1.3 to approximate street distance before applying the rates below.

**Destination hygiene (do this first).** Always use the full street address with number,
postal code and city, never just a venue name. If the event only has a venue name, resolve
its full address first. Watch for namesakes: a street can share its name with a district
that is far away. Sanity-check every result: if it describes a different area, or the time
is wildly off for the rough distance, discard it and fall back to the estimate.

Google Maps links are not used: those pages need JavaScript and return nothing extractable.

1. **Third-party via `web_search`:** try at most two queries, for example
   `Moovit walking time from [home] to [dest] [city]` or `Rome2Rio [home] to [dest]`.
   Accept a result only if it states a travel time for this exact origin and destination.
   Generic stop pages and district-level pages do not count. *In practice this rarely
   succeeds for street addresses, so expect to reach step 2 often.*
2. **Estimate** (the common case): walking ~12 min/km · taxi ~3-4 min/km in a city center ·
   transit incl. transfer usually 15-25 min within a city. Rush hour (8-10, 18-20) →
   taxi +30-50 %.

**Always state the source** in the description (`Source: Moovit` / `Estimate`). For an
estimate also add: `⚠️ Travel time estimated - check in Maps before leaving.`

## 3. Choose the best option
`if walk ≤ walking_threshold_minutes: walk; else: the faster of taxi / transit` (and driving
if enabled). Very early (before 6:00): transit is often "unavailable" → recommend taxi/driving.

## 4. Build the block
- **Title:** `🚶 Commute: [X] min walk` (or `🚖 … taxi` / `🚇 … transit` / `🚗 … drive`)
- **Start:** event start − (travel time + `buffer_minutes`) · **End:** event start − `buffer_minutes`
- **Color:** configured `block_color_id` (default Sage `2`)
- **Do NOT set the `transparency` field** (it causes silent failures). Omit it. If
  `create_event` fails → suspect `transparency` first, then the date format, then retry
  without the field.
- **Description:**
```
🏁 Destination: [event title] at [HH:MM]
📍 Address: [location]

✅ Recommended: [option] - [X] min
🚪 Leave home at: [HH:MM]

Alternatives:  🚶 [X] min · 🚖 [X] min · 🚇 [X] min
Source: [Moovit / Rome2Rio / Estimate]
[⚠️ Travel time estimated - check in Maps before leaving.  ← only for an estimate]
Buffer: [buffer_minutes] min before start.
leave-by: event=[event id]
```
**Starting in the past** (commute start already passed, event still upcoming): create it
anyway with `⚠️ Tight - leave now`.

## 5. Conflict notice
If the gap between event A and event B is less than (travel time A→B + buffer): flag it in
the run summary as `⚠️ Conflict: [A]→[B] needs X min, only Y available`. **Plan the outbound
trip only, never the return.**

---

## Manual triggers (besides the nightly run)
- **"plan my trip to [event]" / "when do I need to leave for [event]"** → just that one event:
  steps 2–4 for it, show the result briefly (`🚪 Leave home at HH:MM · [option] [X] min`).
- **"check my commute blocks" / "update my commute blocks"** → the full nightly contract on demand.

## Limits
- Creates/edits **only its own** commute blocks (configured color, 🚶/🚖/🚇). Never touch
  real / other events.
- Cleans up only **its own, orphaned** blocks - when in doubt, leave it, do not delete.
- No return trip, no commute for virtual events.
- Sends nothing to the user and delivers nothing - that is a companion briefing's job, if any.
