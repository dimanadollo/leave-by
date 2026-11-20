# 🚶 Leave-By

**Never wonder "when do I need to leave?" again.**

A Claude skill that puts a "leave by" block in front of every calendar event that has an address: how long it takes, when to leave home, and alternatives (walk / transit / taxi). It can run every night, so your morning calendar already has the answer.

`Claude skill` · `Google Calendar` · `5-minute setup` · `MIT`

Built by someone in go-to-market, not a career dev. Why: I like cutting small, unnecessary tasks out of my day. Looking up how long it takes to get somewhere is one of them. This makes every day about 1% easier.

---

## What it does

On every run it:

- Scans your upcoming events (next 30 days, configurable)
- Skips anything virtual (Zoom, Meet, Teams, calls) or without a real address
- Looks up the travel time from your home address for each remaining event, and labels it as an estimate when no real routing source answers
- Creates a color-coded block titled like `🚶 Commute: 14 min walk` with a **leave-home time** and alternatives
- Follows events when they move, and cleans up its own blocks when an event is cancelled
- Is **idempotent**: it never creates duplicates, it reconciles what already exists

It never messages you and never touches your real events. It only manages its own blocks.

### Example block

```
🏁 Destination: Dentist at 10:00
📍 Address: Calle Mayor 12, Madrid

✅ Recommended: transit - 18 min
🚪 Leave home at: 09:32

Alternatives:  🚶 34 min · 🚖 11 min · 🚇 18 min
Source: Estimate
⚠️ Travel time estimated - check in Maps before leaving.
Buffer: 10 min before start.
```

---

## Requirements

- **Claude** (claude.ai, the desktop app, or Claude Code) with skills enabled
- **Google Calendar** connected with read + write access. Block colors use Google Calendar color ids; other calendars may work but are untested
- **Web search** enabled (used as the travel-time fallback)
- Optional: a **routing connector** (Mapbox, TomTom, or a Google Routes API key) for accurate travel times. Without one, times are estimates
- Optional: **scheduled tasks**, for the automatic nightly run

---

## Install

**claude.ai or the Claude desktop app**

1. Download `leave-by.zip` from the [latest release](../../releases/latest).
2. In Claude, open **Customize → Skills**, click **+**, then **Create skill → Upload a skill**, and select the ZIP.

**Claude Code**

```bash
git clone https://github.com/dimanadollo/leave-by.git
cp -r leave-by/leave-by ~/.claude/skills/
```

**Build the ZIP yourself** (the skill folder must be the top level of the ZIP):

```bash
zip -r leave-by.zip leave-by/
```

Then run the guided setup below. Prefer editing files? Copy `leave-by/config.example.json` to `leave-by/config.json` and fill it in.

> 🔒 `config.json` contains your home address. It is listed in `.gitignore`, so it is never committed. Keep it that way if you fork this repo.

## Setup (guided)

On first use, say:

> set up leave-by

Setup is a short **tap-through form**, not an interview. Every question has a recommended default, and every question also has an **Other** option where you can type your own value.

- 🏠 **Home address** (the only free-text answer, and the only required one)
- 📋 **Form, round 1:** calendar, travel modes (walk / transit / taxi / own car), how early you want to arrive
- 📋 **Form, round 2:** how many days ahead to plan, walking threshold, color of the commute blocks, how often it runs

Your city and timezone are worked out from your address, so nothing to answer there. You see both at the end and can correct them. If your assistant has no tappable question tool, the same questions are asked in plain text.

When you are done, it runs one pass immediately so you can see it working.

## When it runs

The setup asks how often it should run, and creates the schedule for you:

- 🌙 **Every night** (recommended)
- 🌙☀️ **Every night + a midday top-up**, which also catches events you add during the day
- 📅 **Once a week**
- 🙋 **Only when I ask**

If your assistant cannot create scheduled tasks, setup shows you the exact prompt and time to set up yourself.

You can also trigger it any time:

- "update my commute blocks"
- "when do I need to leave for [event]?"
- "plan my trip to [event]"

---

## Configuration reference

| Key | Meaning | Default |
|-----|---------|---------|
| `home_address` | Start address for all commutes | required |
| `timezone` | IANA timezone | from address |
| `city` | City/region for routing searches | from address |
| `calendar` | Which calendar to maintain | `primary` |
| `buffer_minutes` | Arrive-early buffer | `10` |
| `scan_days_ahead` | How far ahead to plan | `30` |
| `travel_modes` | Modes to consider | `walking, transit, taxi` |
| `walking_threshold_minutes` | Prefer walking if the walk is at most this | `30` |
| `schedule` | When it runs: nightly, nightly + midday, weekly, on demand | `nightly` |
| `block_color_id` | Calendar color id for the skill's blocks | `2` (Sage) |

---

## Design notes

- **Idempotent by design.** The job runs over the same window again and again, so it reconciles (checks what already exists) before acting. No duplicate blocks.
- **Accurate when you connect routing.** If a routing connector (Mapbox, TomTom, Google Routes) is available, it uses real street and line times. Without one, it falls back to web lookups and then estimates, which are clearly labeled (⚠️) so you double-check them before leaving. It never uses straight-line distance as travel time.
- **Careful with place names.** It always uses full addresses with postal code and sanity-checks results, because a street can share its name with a district across town.
- **Blocks are linked to their event by id.** Moved events get their block updated, and two events close together never confuse it.
- **Outbound only.** It plans the trip to an event, never the return.
- **Stays in its lane.** It only creates, edits, or removes blocks it created itself, recognized by their color and emoji.

## Privacy

Your home address and calendar stay in your own assistant's environment and config. Nothing is stored outside them. The only outbound calls are web searches for travel times, and those searches contain your origin and destination addresses.

## Limits

- No return-trip planning
- No blocks for virtual events
- Without a routing connector, travel times are estimates, because free web sources rarely give exact address-to-address times. Blocks say so
- Routing connectors like Mapbox and TomTom cover walking and driving, not public transit, so transit stays an estimate on those
- Events created after the last run are only planned at the next run, unless you add the midday run or ask on demand

---

## Status

Early version, built as a side project and maintained when I have time. Issues and ideas are welcome.

## License

MIT. Use it, fork it, make it yours.
