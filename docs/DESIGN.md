# Training Center – Design Proposal

> Status: **Draft for discussion**. Nothing here is set in stone. The goal is to
> agree on the overall shape of the system before writing code. Open questions
> are collected at the end – please comment on them.

## 1. Goals

| # | Goal | Notes |
|---|------|-------|
| G1 | **Greeting** – detect when I enter the room and greet me on the display and through audio. | E.g. "Godmorgen – klar til træning?" / "Good morning – ready to train?" |
| G2 | **Workout helper** – guide me through sets: show the list of activities, the current activity, time/reps left, and announce transitions by voice. | Must support several kinds of sets (see §7). |
| G3 | **History** – keep a log of what I have done (activities, reps, times, distances, how it felt). | Viewable on the display, exportable later. |
| G4 | **Voice control in Danish and English** – start/pause/skip/answer questions hands-free. | Mixed-language use must work ("start", "næste", "done", "færdig"). |
| G5 | **Use the hardware I already have** – Raspberry Pis, ESP32s, Arduino Uno, sensors, buttons. | Prefer local/offline processing; no cloud dependency for the core loop. |
| G6 | **Hands-free *and* hands-on control** – buttons are a first-class input, not a fallback: one press per rep/round, start/pause, next. | See §4. Voice and buttons enter the engine through the same path. |
| G7 | **Manage and review from a phone/laptop** – start a session remotely, edit plans, view statistics and completion trends. | Same web app as the display, different UI mode (§9). |

Non-goals (for now): multi-user accounts, a cloud-primary datastore, a native mobile
app, video/form coaching. The system is **local-first**: it must run a full workout
with the internet down. Optional, disableable outbound sync and remote access are
in scope (§10.4); a cloud dependency in the core loop is not.

## 2. High-level picture

```
                ┌──────────────────────────── Raspberry Pi (hub, "brain") ────────────────────────────┐
                │                                                                                      │
 ESP32 presence │   ┌──────────────┐    ┌───────────────┐    ┌───────────────┐    ┌────────────────┐   │
 node(s) ──MQTT─┼──▶│  presence    │    │ voice service │    │ workout engine│    │ storage        │   │
 (BLE + mmWave) │   │  service     │    │ wake word,    │    │ state machine,│    │ SQLite history │   │
                │   └──────┬───────┘    │ STT, intents, │    │ timers, plans │    └───────▲────────┘   │
 ESP32 buttons, │          │            │ TTS           │    └──────┬────────┘            │            │
 LEDs, buzzer ──┼──MQTT────┤            └──────┬────────┘           │                     │            │
                │          │   ┌─────────────┐ │   ┌─────────────┐  │                     │            │
 ESP32 sensors ─┼──MQTT────┤   │ serial      │ │   │ input       │  │                     │            │
 (HR, rower)    │          │   │ bridge      │ │   │ normaliser  │  │                     │            │
                │          │   └──────┬──────┘ │   └──────┬──────┘  │                     │            │
 Arduino Uno ───┼─ serial ─┼──────────┘        │          │         │                     │            │
 (USB)          │          ▼                   ▼          ▼         ▼                     │            │
                │   ═══════════════════ MQTT event bus (Mosquitto) ═══════════════════════╧══          │
                │                                   │                                                  │
                │                          ┌────────▼────────┐  WebSocket ──▶ Chromium kiosk on display │
                │                          │ web API + UI    │                                          │
                │                          │ (FastAPI)       │  HTTPS ──────▶ companion UI (phone/laptop)│
                │                          └─────────────────┘                                         │
                │   USB mic / speaker (or USB conference speakerphone)                                 │
                └──────────────────────────────────────────────────────────────────────────────────────┘
```

Key ideas:

* **One Raspberry Pi (ideally a Pi 5, 8 GB) is the hub.** It drives the display
  (HDMI), the microphone and speaker, runs the workout engine and stores history.
* **Small devices are "dumb" sensors/actuators** that publish events and receive
  commands over **MQTT** (ESP32 over Wi‑Fi) or serial/USB (Arduino Uno). A small
  hub-side **serial bridge** service reads the Arduino's line-based serial
  protocol and republishes it on MQTT (and vice versa for buzzer/LED commands),
  so the rest of the system only ever sees MQTT.
* **Every component talks through an event bus (MQTT).** This keeps services
  independent, lets us add sensors later without touching the engine, and makes
  it easy to test each piece in isolation (and to integrate with Home Assistant
  later if wanted).
* **The display is a web page** shown in a full-screen Chromium kiosk. The same
  FastAPI app also serves a **companion UI** for phone/laptop – plan editing,
  remote control and statistics – so there is one codebase, not two (§9).
* **All inputs are equal.** Buttons, voice, web taps and sensors are normalised
  into the same control intents before they reach the engine (§4). A new input
  device is a new MQTT `node`, not an engine change.
* **Sensors are optional and degradable.** Every activity can always be completed
  by timer, button or voice if a sensor is missing (§5).
* **Local-first.** A full workout runs with the internet down; backup, sync and
  remote access are separate, optional concerns (§10).

## 3. Hardware allocation

| Device | Role | Hardware attached |
|--------|------|-------------------|
| Raspberry Pi 5 (or Pi 4) | Hub: engine, voice, UI, storage, MQTT broker | HDMI display/TV, USB speakerphone (mic + speaker with echo cancellation) or USB mic + powered speakers |
| ESP32 #1 | Presence node near the door | Built-in BLE scanner + mmWave radar (e.g. HLK‑LD2410) or PIR sensor |
| ESP32 #2 | **Main control panel** – hands-on controls | Big arcade buttons (Rep/Lap, Start/Pause, Next, Previous, Done), WS2812 LED ring, piezo buzzer |
| ESP32 #3 (optional) | **Satellite control node** near the rower/mat | 1–2 arcade buttons + LED, same firmware, different `node` id |
| ESP32 #4 (optional) | Heart-rate bridge | BLE central reading a standard HRM strap → MQTT |
| Arduino Uno (optional) | Extra I/O via USB serial to the Pi | Buttons, buzzer, reed switch / hall sensor (e.g. count rowing strokes or bike revolutions) |
| Spare Raspberry Pi (optional) | Second display / dev box / heavier speech model host | – |

### 3.1 Input node inventory

One button is never where you are. The design assumes **several input points**, all
publishing to the same MQTT topic with a `node` field, so adding one later is
configuration, not code:

| Node id | Placement | Buttons | Notes |
|---------|-----------|---------|-------|
| `panel` | Wall, next to the display | Rep/Lap (big green), Start/Pause, Next, Previous, Done, optional Hard/Easy pair | The reference panel; LED ring doubles as a progress indicator (§12.3) |
| `wall-rower` | By the rower / mat | Rep/Lap, Done | Reach it without leaving the machine |
| `floor` | Foot-operated, on the floor | Rep/Lap | For sets where both hands are busy |
| `kiosk` | The display itself, if it is a touch screen | On-screen equivalents | Large hit areas, no precision required (§12.8) |
| `phone` | Companion web UI (§9.3) | On-screen equivalents | Also works as a wireless button |

A USB **speakerphone** is strongly recommended: it gives far-field pickup and
acoustic echo cancellation, which matters because the system talks while it
listens, and the room will have music and heavy breathing.

## 4. Input model

Voice is *one* input, not *the* input. When you are out of breath, with music
playing, mid-burpee, a physical press is faster and more reliable than any speech
pipeline. The engine therefore treats every input source equally.

### 4.1 Unified intent path

```
 button press  ─┐
 voice intent  ─┤
 web UI tap    ─┼──▶ normalise ──▶ control intent ──▶ workout engine
 sensor event  ─┘      (hub)        {intent, value, source, node, ts}   ts = hub time
```

* The normaliser **subscribes** to `tc/input/button` (physical nodes and the
  serial bridge), `tc/input/ui` (kiosk and companion taps), `tc/voice/intent`
  and the goal-relevant sensor topics `tc/sensor/rower`, `tc/sensor/hr`,
  `tc/sensor/equipment` (§11). Tier‑2 rep-detection sensors (`tc/sensor/imu`,
  `tc/sensor/mat`, `tc/sensor/gate`, §5.2) are added to this list as each is
  implemented – any sensor that can complete a goal must be subscribed to here,
  or it can never reach the engine. It converts each of them into a **control
  intent** using exactly the vocabulary of §8.3 (`start_workout`, `pause`,
  `next`, `done`, `report_reps`, `feeling`, …).
* It **publishes** on `tc/input/intent`, which is its output only – nothing
  subscribes to `tc/input/*` as a wildcard, so the normaliser never consumes its
  own messages. The engine subscribes to `tc/input/intent` alone.
* **The engine does not know where an intent came from.** `source`/`node` are
  carried for logging and debugging only. This keeps the state machine (§7.3)
  testable without any hardware and means a new input device never requires
  engine changes.
* Intents are idempotent where it matters: a duplicated `done` for an activity
  that already completed is ignored rather than skipping the next one.

### 4.2 Button set and press semantics

Defining these now stops them being invented ad hoc per device. Debounce happens
in firmware; the hub only ever sees clean events.

| Button | Short press | Long press (≥1 s) | Double press |
|--------|-------------|-------------------|--------------|
| **Rep / Lap** (big green) | Count one rep, or complete one round/lap, depending on the current activity type | Undo the last rep | – |
| **Start / Pause** | Start, or toggle pause/resume | Quick-start the usual plan from idle (§12.7) | – |
| **Next** | Advance to the next activity | – | Skip the rest of the current block |
| **Previous** | Restart the current activity | Go back to the previous activity | – |
| **Done / Confirm** | Complete the current activity, or confirm a feedback prompt | – | – |
| **Hard / Easy** (optional pair) | Record `feeling` without speaking | – | – |

A third gesture, **hold (≥3 s)**, is reserved for stopping the session
(§12.8): on Done, and on Rep/Lap for single-button nodes that have no Done
button. Because a hold necessarily passes through the `long` threshold, the
firmware **buffers the gesture until the button is released or the 3 s threshold
is reached**, then emits *either* `long` *or* `hold` – never both. A 3 s hold on
Rep/Lap therefore stops the session without first undoing a rep.

For `reps` activities the Rep button **replaces the "how many did you do?"
dialogue entirely** – the count is already known, so the engine skips straight to
rest/next.

### 4.3 Feedback and latency

Every press must produce immediate multi-sensory confirmation: LED ring flash +
short buzzer click (locally on the node, not waiting for the hub) and the
on-screen counter incrementing.

> **Non-functional requirement:** button press → **node-local** LED flash and
> buzzer click in **< 100 ms**. This is the single biggest "feels great vs.
> feels broken" factor and is treated as a hard requirement, not a
> nice-to-have. The hub-driven screen update is a separate, looser budget of
> < 200 ms (§12.1), because it crosses the network.

The local click/flash is fired by the node itself so it is never blocked by Wi‑Fi,
MQTT or hub latency; the hub-driven screen update follows.

## 5. Sensors

Sensors are ranked by value-per-effort. Nothing below is required for the core
loop.

> **Design principle – optional and degradable.** The engine's contract is "an
> activity completes when its goal is reached, **from whatever source**". If a
> sensor is absent, offline or wrong, the button/voice/timer path still
> completes the activity. Sensor work must never block the core loop.

### 5.1 Tier 1 – high value, low effort

| Sensor | Signal / wiring | MQTT topic | What it unlocks |
|--------|-----------------|------------|-----------------|
| **BLE heart-rate strap** | Standard BLE HRM GATT profile, read by the Pi or an ESP32 bridge | `tc/sensor/hr` | Live HR + zone on screen, rest-until-HR logic ("wait until you are under 130"), recovery and calorie estimates, far better post-session stats |
| **Reed / hall sensor on rower or bike** | Magnet + reed switch → Arduino/ESP32 GPIO | `tc/sensor/rower` | Automatic distance, stroke/cadence rate – no pressing anything |
| **Ambient temp/humidity** | DHT22 / BME280 on any node | `tc/sensor/ambient` | Context in the history ("why was that session so hard?") |

### 5.2 Tier 2 – meaningful, more work

| Sensor | Signal / wiring | MQTT topic | What it unlocks |
|--------|-----------------|------------|-----------------|
| **Accelerometer / IMU** | ESP32 + MPU6050 on equipment or body | `tc/sensor/imu` | Automatic rep detection. Start with the simplest reliable case (jump rope, kettlebell swings) rather than trying to count every movement |
| **Load cell / pressure mat** | HX711 + load cell, or a switch mat, under a platform | `tc/sensor/mat` | "You are on the mat", jump counting, presence confirmation at the exercise spot |
| **Light gate / ToF (VL53L0X)** | I²C sensor aimed across the movement | `tc/sensor/gate` | Push-up or squat depth counting – one sensor, one movement, surprisingly robust |
| **Smart equipment** | Concept2 PM5 over BLE, FTMS bikes (power/cadence) | `tc/sensor/equipment` | Reading an existing monitor beats building a sensor |

### 5.3 Tier 3 – experiments / out of scope

| Idea | Status |
|------|--------|
| **Camera-based rep counting / form feedback** (pose estimation) | **Explicitly out of scope.** Powerful, but heavy on the Pi, privacy-sensitive (video of a person exercising at home), and a large scope increase in both compute and UX. Recorded here so it is not rediscovered every few months. |
| **BLE bathroom scale** | Nice-to-have; would add a weight trend to the stats screens (§9.4) |

### 5.4 Rep counting modes

Each activity declares how its reps are counted (§7.2, `count_mode`):

| Mode | Meaning |
|------|---------|
| `manual_button` | One Rep/Lap press = one rep (§4.2). Degrades to `voice` at runtime when no physical button node is online, even when the plan set it explicitly – the plan states the *intent*, availability decides what is actually possible (§12.6) |
| `sensor` | A sensor publishes the count/distance; buttons still work as an override |
| `voice` | Reported verbally after the set ("I did eight") |
| `auto` | Derived from the timer alone (e.g. `timed` activities) |

Default: `manual_button` while at least one **physical button node** (`panel`,
`wall-rower`, `floor`, or the Arduino I/O board) is online, otherwise `voice`
(§12.6). The `kiosk` and `phone` nodes deliberately do **not** count: an open
browser tab is not something you can press mid-burpee, so it must not keep the
system in a mode that assumes a reachable physical button. Both can still send
every intent at any time.

## 6. Presence detection & greeting (G1)

Detecting "a phone" is less reliable than it sounds: modern iOS/Android phones
randomise their Bluetooth and Wi‑Fi MAC addresses and sleep their radios. We
therefore combine **"someone is in the room"** with **"it is me"**:

| Signal | How | Pros | Cons |
|--------|-----|------|------|
| mmWave radar / PIR on ESP32 | GPIO/UART → MQTT `tc/presence/motion` | Instant, reliable that *someone* entered | Does not identify who |
| BLE beacon from phone | Phone app broadcasting an iBeacon with a fixed UUID (e.g. Home Assistant companion app "BLE transmitter"), ESP32 scans RSSI (e.g. ESPresense firmware or own sketch) | Identifies me, distance estimate by RSSI | Needs an app setting on the phone; iOS limits background broadcasting |
| BLE key-fob / tag | Cheap BLE beacon on the keyring/in the gym bag | Very reliable, fixed ID | One more thing to carry |
| Wi‑Fi | Phone with fixed (non-random) MAC/IP on the home network, Pi pings it / checks router | No extra hardware | Phone Wi‑Fi sleeps → slow/unreliable "arrival" signal |

**Proposed logic** (presence service):

1. Motion detected **and** my BLE ID seen with RSSI above threshold within ±30 s → state `ARRIVED(user)`.
   If the BLE ID was already seen before the motion, greet immediately.
2. Motion detected but no known ID → wake the display at once (welcome screen
   without name) and wait up to a short grace period (e.g. 5 s) for a BLE
   sighting. If none arrives → state `ARRIVED(unknown)` → generic spoken greeting
   ("Hej! / Hi!"); voice can still be used.
3. Upgrade: if my BLE ID is seen later (within the 30 s window) while in
   `ARRIVED(unknown)`, switch to `ARRIVED(user)` and update the screen with the
   personal greeting/last-session summary. The *spoken* greeting is not repeated.
4. No motion for N minutes **and** BLE ID gone → `LEFT` → display goes to idle/screensaver, unfinished session is saved.
5. Debounce: no new spoken greeting within e.g. 30 minutes of the last one
   (the unknown→user upgrade in rule 3 is exempt because it is silent).

**Greeting** = display wakes up (HDMI-CEC or DPMS on), shows a welcome screen
with time of day, last workout summary, and suggested next workout; TTS says a
short greeting in the preferred language, e.g.
"Godmorgen! Sidste gang lavede du 20 minutters intervaller. Skal vi køre samme program?"

## 7. Workout model (G2)

### 7.1 Concepts

* **Program** – an optional multi-week schedule of plans (§7.6). A plan can be
  run on its own; a program just says *which* plan on *which* day.
* **Plan** – a named workout ("Morning EMOM", "Row + core"), made of blocks.
* **Block** – a group of activities repeated N rounds (e.g. "3 rounds of: …"),
  optionally with its own **block goal** (§7.4).
* **Activity** – one exercise with a **goal type**:

| Type | Example | Completion | System behaviour |
|------|---------|-----------|------------------|
| `timed` | 1 min burpees | Timer ends | Count down, voice cues at "halfway", "10 seconds", "3‑2‑1", announce next. Covers EMOM / 1‑minute interval sessions. |
| `reps` | 10 push-ups | Target reps counted (button/sensor), or I say "done"/"færdig" | Shows target reps and a live counter, waits (no time limit), optional rest timer afterwards. With `count_mode: manual_button` the count is already known, so the "how many did you do?" question is skipped entirely (§4.2). |
| `reps_in_time` | 10 reps within 1 min | Done **or** timer ends | When finished, asks "How did it go? / Hvordan gik det?" → answers like "all of them", "eight", "too hard", "let's do the last 2" → engine adapts (logs actual reps, optionally adds a mini-set for the remaining reps). |
| `weighted_reps` | 8 back squats at 60 kg | Target reps counted, or "done" | As `reps`, but carries a **load** (§7.3). Shows the target weight prominently, asks or accepts a corrected weight afterwards, and is the type that feeds progressive overload (§12.5). |
| `distance` | 1600 m rowing | I say "done", or a sensor reports distance | Periodic check-ins (e.g. every 2 min or every 400 m if distance is known): "How is it going? / Hvordan går det?" → answers like "fine", "halfway", "800 meters", "pause". |
| `rest` | 30 s rest | Timer ends | Shows what is next so I can get ready. |

### 7.2 Plan file format (YAML, stored in git)

```yaml
schema_version: 1       # see §10.7 – bumped whenever the plan schema changes
name: Morning EMOM
language: da            # default language for spoken cues (da | en)
count_mode: manual_button   # plan-wide default, overridable per activity (§5.4)
blocks:
  - name: Warm-up
    rounds: 1
    activities:
      - { type: timed, exercise: jumping_jacks, duration: 60s }
      - { type: timed, exercise: air_squats, duration: 60s }
  - name: Strength
    rounds: 3
    activities:
      - { type: weighted_reps, exercise: back_squat, reps: 8, weight: 60kg }
      - { type: rest, duration: 120s }
  - name: EMOM
    rounds: 5
    activities:
      - { type: reps_in_time, exercise: burpees, reps: 10, time_limit: 60s,
          count_mode: manual_button }
      - { type: reps, exercise: push_ups, reps: 10, count_mode: manual_button }
      - { type: rest, duration: 30s, until_hr_below: 130, max_duration: 90s }
  - name: Finisher
    rounds: 1
    goal: { type: amrap, time_cap: 8min }     # block-level goal, §7.4
    activities:
      - { type: reps, exercise: kettlebell_swings, reps: 15, weight: 16kg }
      - { type: reps, exercise: push_ups, reps: 10 }
  - name: Cool-down
    rounds: 1
    optional: true          # trimmed first by the time budget, §7.7
    activities:
      - { type: distance, exercise: rowing, distance: 1600m, check_in_every: 2min,
          count_mode: sensor, hr_zone: 2 }
```

Schema additions beyond the original draft:

| Field | Applies to | Meaning |
|-------|------------|---------|
| `schema_version` | plan | Which version of this schema the file targets (§10.7). |
| `count_mode` | plan, block, activity | How reps/laps are counted: `manual_button`, `sensor`, `voice`, `auto` (§5.4). Innermost wins. This is a *preference*: if the hardware it needs is offline, the engine degrades it at runtime (§5.4, §12.6) rather than failing. |
| `weight` | `weighted_reps`, and any `reps`/`reps_in_time` activity using a loaded implement | Target load (§7.3). |
| `goal` | block | Block-level goal – `amrap` or `for_time` (§7.4). Without it, a block is simply its rounds. |
| `optional` | block, activity | May be trimmed to fit a time budget (§7.7) without counting as skipped. |
| `until_hr_below` | `rest` | End the rest when heart rate drops below this value – requires a Tier‑1 HR sensor (§5.1). |
| `max_duration` | `rest` | Safety cap for `until_hr_below`, and the fallback when no HR sensor is available. |
| `hr_zone` | any work activity | Target zone shown on screen; out-of-zone is a hint, never a blocker. |

**Degradation rule:** if a field depends on a sensor that is missing or offline,
the engine falls back to the plain time/rep/button behaviour and shows a degraded
indicator (§12.6). A plan never becomes unrunnable because a sensor is absent.

### 7.3 Load (strength training)

Without a load concept the system cannot express "3×8 at 60 kg", which means the
per-exercise progression in §9.4 and the progressive-overload suggestions in
§12.5 would have nothing to work on. So:

* **Unit: kilograms**, always, stored as a number. There is no pounds mode; a
  display preference can be added later without touching the data.
* `weight` on an activity is the **target**. What actually happened is logged
  separately (`target_weight` / `actual_weight`, §10.1), because changing the
  weight mid-session is normal and must not silently rewrite the plan.
* **Where the target comes from**, innermost first: the activity's `weight`, then
  the last logged `actual_weight` for that exercise, then `default_weight` from
  the exercise library (§7.5). This means a plan can say "back squat 3×8" with no
  number at all and still show you a sensible target.
* **Bodyweight exercises carry no weight.** For exercises marked
  `bodyweight: true` the field is absent, not zero. Added load (weighted vest,
  dip belt) is expressed as `weight` on an otherwise bodyweight exercise.
* **Changing the load is an intent**, not an edit: "sixty-five" or the Hard/Easy
  buttons adjust the working weight for the remaining rounds of the block and log
  the change.
* **Progression** uses `progression_step` from the exercise library (§7.5) – e.g.
  +2.5 kg for a squat, +1 kg for a press – so suggestions land on plates that
  actually exist.

### 7.4 Block goals: AMRAP and "for time"

Some very common workout shapes are goals of a *block*, not of an activity, and
cannot be expressed by rounds alone:

| Block goal | Means | Completion | Score |
|------------|-------|------------|-------|
| `amrap` | "As many rounds as possible in 12 minutes" – the activity list repeats until the time cap | Time cap reached | Rounds completed **plus** reps into the partial round, e.g. `7 + 12` |
| `for_time` | "3 rounds of … as fast as you can" – fixed work, variable time | All rounds done, or the optional `time_cap` is hit | Elapsed time, e.g. `9:41`; `time_cap` reached without finishing is logged as capped, with the work completed |

Both need a **score**, which is a property of a block, not of an activity – so
`block_log` exists for exactly this (§10.1). A score is the thing you compare
between sessions, so it is what personal bests (§12.4) are computed from: "same
block, better score".

Rules:

* Scores are only comparable when the block is **unmodified** – same activities,
  same reps, same loads, same cap. A trimmed or adjusted block is logged with
  `modified: true` and excluded from PB comparison, so an easier version never
  sets a record.
* In an `amrap` block the engine does not announce "next round" as an
  achievement; it tracks the round counter quietly and calls out the time
  remaining instead.
* `for_time` suppresses the per-activity rest prompts – the clock is running.

### 7.5 Exercise library

`exercises.yaml` is relied on by the voice (names), the plan editor (a browsable
library), the statistics (grouping), the equipment check (§9.7), the load
defaults (§7.3) and the demo clips (§9.8). It therefore holds more than names:

```yaml
back_squat:
  en: Back squat
  da: Squat med vægtstang
  category: strength          # strength | conditioning | mobility | core
  muscles: [quads, glutes]
  equipment: [barbell, rack]
  default_weight: 60kg
  progression_step: 2.5kg
  media: back_squat.webm      # 3 s looping clip, §9.8
  cues:
    en: "Chest up, knees out"
    da: "Brystet op, knæene ud"

push_ups:
  en: Push-ups
  da: Armstrækninger
  category: strength
  muscles: [chest, triceps]
  equipment: []
  bodyweight: true
  media: push_ups.webm

kettlebell_swings:
  en: Kettlebell swings
  da: Kettlebell sving
  category: conditioning
  equipment: [kettlebell]
  default_weight: 16kg
  progression_step: 4kg       # kettlebells come in fixed sizes

rowing:
  en: Rowing
  da: Roning
  category: conditioning
  equipment: [rower]
```

Every field except the two names is optional, so the library can start small and
grow. A missing `media` simply means no clip is shown; a missing `equipment`
means the exercise never appears in the equipment check.

### 7.6 Programs: what should I do today?

A plan is one workout. A **program** is an optional schedule of plans over weeks,
and it is what turns the system from a very good interval timer into something
that feels like a coach – §12.5's adaptivity assumes it exists.

```yaml
schema_version: 1
name: Winter base
weeks: 4
days:
  mon: { plan: strength_a }
  tue: { plan: morning_emom }
  wed: { rest: true }
  thu: { plan: strength_b }
  fri: { plan: row_intervals }
  sat: { plan: long_row, optional: true }
  sun: { rest: true }
```

* **Programs are suggestions, never obligations.** The idle screen shows "today:
  Strength A" with a one-press start, and any other plan is always one tap away.
* **Rest days are first-class** – they keep a streak alive (§10.8) rather than
  breaking it, and the display says so instead of staying blank.
* **Falling behind is normal.** The program tracks which sessions were done
  rather than demanding a particular date; missing Tuesday does not shift
  everything or produce a guilt-trip screen.
* A program can specify a **progression rule** per plan (e.g. "+2.5 kg on the
  squat each week"), which is applied to the target weights (§7.3) when the plan
  is started from the program.
* Programs are YAML in the repository, like plans, and editable in the companion
  UI (§9.2).

### 7.7 Time budget: "I have 20 minutes"

The most common reason a session does not happen is not motivation – it is not
having the planned 45 minutes. The plan editor already computes an estimated
duration (§9.2), so the engine can use it the other way round:

* Say "I have twenty minutes" (or pick a budget in the UI) and the engine
  proposes a **trimmed version** of the plan that fits.
* Trimming order: drop blocks and activities marked `optional` first (cool-down,
  accessory work), then reduce `rounds` in conditioning blocks, then shorten
  rests – **never** silently reduce the load or the reps of a strength set,
  because that corrupts the progression history.
* The proposal is **shown before starting**, never applied silently, and the
  session is logged as `trimmed` with the original plan recorded, so statistics
  can tell a short session from a skipped one.
* The inverse is also useful: "I have an hour" can offer the optional blocks back.

### 7.8 Engine state machine

```
 IDLE ──start(plan)──▶ READY ──"start"/countdown──▶ ACTIVE ──goal reached──▶ FEEDBACK? ──▶ REST? ──▶ next activity
   ▲                                                 │  ▲                                              │
   │                                    "pause"      ▼  │ "resume"                                     │
   │                                              PAUSED                                               │
   └────────────── "stop" / plan finished ◀── SUMMARY ◀─────────────── last activity done ◀───────────┘
```

Additional transitions (driven by the intents in §8.3):

| From | Intent / event | To |
|------|----------------|----|
| ACTIVE, PAUSED, REST | `next` | READY for the next activity (current one logged as `skipped`) |
| ACTIVE, PAUSED, REST, READY | `previous` | READY for the previous activity (restart it) |
| ACTIVE, PAUSED, REST, FEEDBACK | `stop` | SUMMARY (session saved as `stopped`) |
| FEEDBACK | `report_reps`, `feeling`, `done` | REST (if configured) or next activity |
| FEEDBACK | `adjust` ("let's do the last 2") | ACTIVE with an inserted mini-activity for the remaining reps; afterwards continues to REST/next |
| FEEDBACK | no answer within e.g. 15 s | log without feedback, continue |
| ACTIVE (`distance`) | check-in timer | stays ACTIVE, emits `ask_check_in`; answers update the log |
| ACTIVE (`reps`, `reps_in_time`, `weighted_reps`) | `rep` (button press or sensor pulse) | stays ACTIVE, increments the counter; completes the activity when the target is reached |
| ACTIVE (`weighted_reps`) | `set_weight` ("sixty-five", Hard/Easy) | stays ACTIVE; updates the working weight for the remaining rounds of the block and logs it (§7.3) |
| ACTIVE (block goal `amrap`) | last activity of a round done | back to the first activity of the block, round counter +1, until the time cap (§7.4) |
| ACTIVE (block goal `amrap`) | time cap reached | block complete; score = rounds + partial reps |
| ACTIVE (block goal `for_time`) | all rounds done, or `time_cap` reached | block complete; score = elapsed time, flagged capped if the cap was hit |
| ACTIVE | `undo_rep` (long press) | stays ACTIVE, decrements the counter (never below 0) |
| FEEDBACK, REST (within a 5 s grace window after the target was reached) | `undo_rep` | back to ACTIVE with the counter decremented, so a miscounted final press can be corrected |
| REST (`until_hr_below`) | HR below threshold, or `max_duration` reached | next activity |

* The engine is **pure logic driven by a clock tick and events** (voice intents,
  button presses, sensor data). It emits events (`activity_started`,
  `countdown`, `ask_feedback`, `session_finished`, …) which the UI and voice
  service render. This makes it testable without any hardware (§17).
* All timers use a monotonic clock on the hub (not on the ESP32s). Anything
  *calendar*-related – streaks, weekly totals – uses civil time instead, with its
  own rules (§10.8).
* Every state change is persisted so a crash/reboot can resume the session.

### 7.9 What counts as "completed"

§9.4 promises a completion rate per plan, which is meaningless without a
definition. Every activity is logged with a **status**, and the status is
derived, not guessed:

| Status | Meaning |
|--------|---------|
| `completed` | The goal was reached: the timer ran out, or ≥ 100 % of the target reps/distance were recorded |
| `partial` | Started, and ≥ 50 % of the target recorded, but the goal was not reached (e.g. 6 of 10 reps before `next`) |
| `skipped` | Started but < 50 % recorded, or skipped outright with `next` |
| `not_reached` | The session ended before this activity was reached at all – **not** the same as skipping it |
| `trimmed` | Removed up front by the time budget (§7.7) – excluded from completion statistics entirely |

* A **session** is `completed` when every non-optional activity is `completed` or
  `partial`, `stopped` when it was ended early, and `abandoned` when it was never
  closed and got auto-closed (§10.8).
* **Completion rate per plan** (§9.4) = non-trimmed activities that are
  `completed` ÷ activities reached, per activity, across sessions of that plan.
  Counting it per *activity* is what surfaces "you skip the finisher 60 % of the
  time"; a session-level rate would hide it.
* `not_reached` is excluded from the rate, otherwise stopping early would make
  every later activity look deliberately skipped.

## 8. Voice (G4) – Danish and English

### 8.1 Pipeline

```
mic ─▶ wake word ─▶ record until silence (VAD) ─▶ speech-to-text ─▶ intent parser ─▶ engine
                                                                                     │
speaker ◀──────────────────────── text-to-speech ◀── response/cue text ◀────────────┘
```

| Stage | Proposed (offline, runs on Pi) | Alternatives |
|-------|--------------------------------|--------------|
| Wake word | **openWakeWord** with a custom wake word (e.g. "Hey Coach" / "Hej Træner") | Porcupine (Picovoice, needs access key) |
| VAD | **Silero VAD** or WebRTC VAD | – |
| Speech-to-text | **faster-whisper** (Whisper `base`/`small`, int8) – supports Danish and English and auto-detects the language | Vosk (very light, but check Danish model availability/quality); run Whisper on a stronger PC on the LAN if the Pi is too slow |
| Intent parsing | Own small **keyword/grammar matcher** per language (see table below), plus number parsing ("ti", "otte", "800 meter") | Rasa/LLM later if needed |
| Text-to-speech | **Piper** – has Danish (`da_DK`) and English voices, runs fast on a Pi | espeak-ng (robotic), cloud TTS |

**During a workout we can avoid the wake word for a small set of commands**
("next", "pause", "done", "færdig") when the engine is waiting for an answer,
e.g. right after it asked "How did it go?". Outside those windows the wake word
is required to avoid false triggers from music/TV.

Buttons are **not** merely a fallback for voice – they are an equal input path
(§4). Voice shines for reporting and asking ("I did eight", "how long left?");
buttons shine for anything you do while moving. Both produce the same control
intents.

### 8.2 Language handling

* Whisper returns the detected language per utterance. The intent parser tries
  that language first, then the other.
* Responses are spoken in the **user's preferred language** (setting), or
  "mirror" mode: answer in the language the command was spoken in.

### 8.3 Initial command set

| Intent | English examples | Danish examples |
|--------|------------------|-----------------|
| `start_workout` | "start", "start morning EMOM" | "start", "start morgentræning" |
| `pause` / `resume` | "pause", "wait", "resume", "continue" | "pause", "vent", "fortsæt" |
| `next` / `previous` | "next", "skip", "back" | "næste", "spring over", "tilbage" |
| `done` | "done", "finished" | "færdig", "jeg er færdig" |
| `report_reps` | "eight", "I did eight", "all of them" | "otte", "jeg tog otte", "alle sammen" |
| `report_distance` | "800 meters", "halfway" | "800 meter", "halvvejs" |
| `adjust` | "let's do the last two", "add one more round" | "lad os tage de sidste to", "en runde mere" |
| `status` | "how long left?", "what's next?" | "hvor lang tid er der tilbage?", "hvad er det næste?" |
| `feeling` | "easy", "good", "hard", "too hard" | "let", "fint", "hårdt", "for hårdt" |
| `stop` | "stop workout" | "stop træningen" |
| `rep` | "one", "rep" (mostly a button intent, §4.2) | "en", "rep" |
| `undo_rep` | "undo", "scratch that" | "fortryd" |

Every intent above is also reachable from a button or the companion UI where it
makes sense, and every button press maps onto one of these intents – there is one
vocabulary, not three.

## 9. Web UI: kiosk display and companion

The FastAPI service already serves the display; the cheapest big win is to let it
serve a **second, management surface** for phone/laptop rather than building a
separate system. **One codebase, one API, one WebSocket stream, two UI modes.**

| | Kiosk mode | Companion mode |
|---|------------|----------------|
| Device | Wall display / TV, Chromium kiosk | Phone, tablet, laptop |
| Design | Huge type, glanceable at 3–4 m, no interaction required | Dense, interactive, thumb-friendly |
| Purpose | Guide the session in progress | Manage plans, control remotely, review statistics |
| Auth | None, but reachable only from the local display (§13) | Login required (§13) |

### 9.1 Kiosk screens

* **Idle / welcome**: clock, greeting, last session summary, current streak,
  today's suggestion from the program (§7.6), a one-line readiness note (§12.9),
  "press the green button or say *Hey Coach, start* to begin".
* **Ready screen** (after choosing a plan, before starting): estimated duration,
  the equipment check (§9.7), and the time-budget option (§7.7).
* **Workout screen**:
  * Large current activity name + big countdown / rep counter / distance.
    Exactly **one** number is dominant (§12.3); everything else is secondary.
  * Progress ring or bar for the current activity, round counter ("Round 3/5").
  * Side list of all activities in the plan with the current one highlighted and finished ones ticked.
  * "Up next" preview, with a looping demo clip of the next exercise (§9.8).
  * Working weight for `weighted_reps`, large enough to read from the rack
    (§7.3), and the block score so far for an `amrap`/`for_time` block (§7.4).
  * Live heart rate and zone when an HR sensor is connected (§5.1).
  * Last-session comparison for the same activity ("last time: 18").
  * Small status strip: microphone state, connected input nodes, sensor status,
    and any degraded mode (§12.6).
* **Summary screen**: total time, per-activity results, feeling scores, personal
  bests hit, one highlighted positive takeaway (§12.4).
* **History screen**: calendar/list of sessions; say "show history / vis historik".

### 9.2 Plan editor

Create and edit plans in the browser instead of hand-editing YAML:

* **YAML stays the storage format** so plans remain diffable and reviewable in
  git. The editor reads and writes those files; it is a view over the files, not
  a second source of truth.
* Plan library with search, duplication and versioning (via git history).
* Templates for the common shapes: EMOM, AMRAP, Tabata, circuit, intervals,
  strength sets – every one of which the model can now actually express
  (§7.3, §7.4).
* An **exercise picker** browsing the library (§7.5) by category, muscle group
  and equipment, with the demo clip as a preview.
* Validation against the schema (§7.2) with a dry-run preview of the timeline
  and the total session duration – the same estimate the time budget uses (§7.7).
* The same editor edits **programs** (§7.6).
* **Import / export** of plans and programs as YAML files. Plans are already
  plain files in git, so sharing one is just sending a file – no account, no
  service, no sign-up.

### 9.3 Live remote control

From the phone: start a plan, pause, skip, adjust, record feeling. This doubles
as a **wireless button** when you are away from the wall panel – it publishes the
same control intents as a physical node, with `node: phone` (§3.1, §4.1).

### 9.4 Statistics and completion views

* **Streaks** and a calendar heat-map of sessions, using the civil-time rules in
  §10.8 (and respecting scheduled rest days).
* **Per-exercise progression** – reps, **weight** (§7.3) and pace over time, with
  personal bests marked. For strength work this is estimated 1RM and top set;
  for conditioning it is pace or score.
* **Block scores** – AMRAP and "for time" results over time, comparing only
  unmodified blocks (§7.4).
* **Completion rate per plan** – e.g. "you skip the finisher 60 % of the time"
  → suggest shortening it. Computed per activity from the statuses defined in
  §7.9, so skipped, partial, trimmed and never-reached are all distinguishable.
* **Session detail** – timeline of the actual session against the plan, HR
  trace, feeling scores, inputs used.
* **Volume and time totals** by week and month, with simple trend lines. The aim
  is a handful of charts you actually look at, not an analytics suite.
* Every view also offers **edit/delete** for the underlying session (§10.10),
  because a statistic you cannot correct is a statistic you stop believing.

### 9.5 Post-session review

Immediately after a session the companion UI offers a short review page (notes,
feeling score, "how was that?"). Asking afterwards, while it is still fresh,
gives far better data quality than asking mid-workout when you are gasping.

### 9.6 Export

* CSV / JSON export of sessions and activity logs.
* Optionally write sessions as `.fit` or `.tcx` so they can be uploaded to
  Strava/Garmin if that is ever wanted. Export is a manual, explicit action.

### 9.7 Equipment check

The ready screen lists what the plan needs – "16 kg kettlebell, barbell + rack,
rower, mat" – collected from the `equipment` and `default_weight` fields of the
exercise library (§7.5). Finding out mid-session that the plates are still in the
shed is a small thing that ruins a session.

### 9.8 Exercise demo clips

A short (≈3 s) looping clip per exercise, shown in the "up next" preview and in
the plan editor's exercise picker. This is a large quality jump for unfamiliar
movements and is **completely distinct from the camera work ruled out in §5.3**:
the clips are static files recorded once, stored locally in `assets/exercises/`,
referenced by `media` in the library, and involve no live camera, no pose
estimation and no privacy exposure whatsoever. A missing clip is simply not
shown.

### 9.9 Guest and partner mode

Training with a visitor, or with a partner side by side, must not pollute your
own statistics – otherwise you will avoid doing it, or the history becomes
untrustworthy.

* A session can be started **as a guest** from the ready screen. It is logged
  against a `is_guest` user row (§10.1), kept out of your streaks, progression
  and personal bests, and can be deleted wholesale afterwards.
* A **partner session** logs the same plan for two users, so both get their
  history. Rep counting stays single-source – whoever presses the button – so
  this is deliberately simple rather than trying to track two people's reps from
  one panel.
* No login is required for a guest: the point is a friend visiting, not an
  account system. Multi-user *accounts* remain a non-goal (§1).

### 9.10 Nudges

Optional web-push notifications from the companion UI: "4 days since your last
session", or a reminder at your usual training time learned from history. Off by
default, one tap to disable, and never more than one per day – a nagging system
gets muted, which is worse than a silent one.

Tech: plain HTML/CSS + a small JS framework (e.g. Svelte or vanilla), live
updates over WebSocket from the API service. Both modes share components and
differ mainly in layout and type scale.

## 10. Data, history and sync (G3)

### 10.1 Schema

SQLite database on the Pi is the **single source of truth**:

```
user(id, name, language, hr_max, is_guest, created_at)
session(id, user_id, plan_name, plan_version, program_name, started_at, ended_at,
        local_date, status, trimmed_from, notes, edited_at)
block_log(id, session_id, block_name, block_index, goal_type,
          score_rounds, score_reps, score_seconds, capped, modified,
          started_at, ended_at)
activity_log(id, session_id, block_log_id, round, exercise, type, count_mode,
             status,                                   -- §7.9
             target_reps, actual_reps, target_seconds, actual_seconds,
             target_meters, actual_meters,
             target_weight_kg, actual_weight_kg,       -- §7.3
             feeling, avg_hr, max_hr, started_at, ended_at)
sample(id, session_id, ts, metric, value)      -- HR/pace/cadence time series
hrv_reading(id, user_id, taken_at, rmssd, resting_hr, source)   -- §12.9
event_log(id, session_id, ts, source, node, kind, payload_json)  -- raw events (§17.2)
```

**Decision – carry `user_id` from day one.** Multi-user remains a non-goal, but
`user_id` is the one schema change that is cheap now and painful later (it is
also exactly what an online datastore, a guest (§9.9) or a family member would
force). A single row in `user` is created during first-run setup (§16) and
everything references it.

**Decision – weights are stored in kilograms** as numbers (`_kg` suffix), never
as a formatted string, so progression arithmetic is trivial (§7.3).

**Decision – `block_log` exists because a score belongs to a block.** AMRAP and
"for time" results (§7.4) have nowhere else to live, and personal bests compare
*scores for the same unmodified block*, which is exactly what this table holds.

**Decision – `session.local_date`** is stored alongside the UTC timestamps,
because streaks and calendars are civil-time questions (§10.8).

Plans, programs and the exercise library live as YAML files in this repository;
history lives only in the database (it is personal data, not source code).

### 10.2 Local-first, by decision

The workout must run with the internet down (G5). Therefore:

* The engine, UI, voice and storage have **no** network dependency outside the
  LAN during a session.
* Nothing below may become a prerequisite for starting or finishing a workout.

### 10.3 Backup

A nightly **encrypted backup** of the SQLite file to a NAS, a second Pi or object
storage. This is the 90 % solution for "don't lose my history" and is far simpler
and more robust than real synchronisation. Restores are tested periodically –
an untested backup is not a backup.

### 10.4 Optional outbound sync

An optional, **fully disableable** sync service that pushes *completed sessions*
to a remote store so history can be viewed away from home (and later from a
phone app):

* **One-way and append-only** – the hub pushes, nothing is ever pulled back into
  the live system.
* **Completed sessions only** – never live state, never mid-session data.
* **Retry on failure** with a local queue; failures are invisible to the workout.
* Off by default.

### 10.5 Privacy boundary

If a hosted datastore is ever used, the boundary is decided now:

| Data | Leaves the house? |
|------|-------------------|
| `session`, `block_log`, `activity_log` | Yes, if sync is enabled |
| `sample` (HR traces) | Yes, if sync is enabled – this is health data; encrypted at rest and in transit |
| `event_log` (raw events) | **No** – debugging data, stays local |
| Audio / recordings / transcripts | **Never** – they do not leave the device, and audio is not persisted at all |

Training and heart-rate data are health data. They are treated as sensitive by
default: encrypted backups, no third-party analytics, no sharing without an
explicit action by the user.

### 10.6 Remote access without a cloud datastore

For "see my stats while away", a VPN (Tailscale / WireGuard) to reach the hub's
own web UI is simpler, more private and more capable than replicating data to a
hosted service. **Recommendation: do this first**; it very likely removes the
need for an online datastore entirely.

### 10.7 Versioning and migrations

Three things version independently, and all three will change:

| What | How |
|------|-----|
| **Database schema** | Numbered, forward-only migration scripts in `hub/storage/migrations`, applied automatically at startup. The current version is stored in the database. A backup (§10.3) is taken automatically before any migration runs. |
| **Plan / program YAML** | `schema_version` at the top of each file (§7.2). The loader accepts older versions and upgrades them in memory; a `migrate-plans` command rewrites the files in place when it is worth it. An unknown *newer* version is refused with a clear message rather than half-parsed. |
| **Firmware** | Semantic version reported in `tc/node/status` (§11), so the UI can show which nodes are behind (§15.2). |

The engine must tolerate history written by older versions: a session logged
before `weighted_reps` existed simply has no weight, and the statistics must
treat that as "unknown", never as zero.

### 10.8 Calendar semantics (civil time)

Timers use a monotonic clock (§7.8), but streaks, heat-maps and weekly totals are
**civil-time** questions, and getting them wrong is a classic source of
"why does it say I broke my streak?":

* Each session stores a **`local_date`** computed in the hub's configured local
  time zone when the session *starts*. A session starting at 23:50 and ending at
  00:20 belongs to the day it started.
* A **"training day"** is a local date on which at least one session reached
  `completed` or `partial`.
* A **streak** counts consecutive training days, and a **rest day scheduled by a
  program (§7.6) does not break it**. An unplanned gap does.
* **Weeks start on Monday** (Danish/ISO convention).
* **DST** is handled by storing UTC timestamps plus `local_date`, never by
  storing a local timestamp alone; a day is not assumed to be 24 h long.
* If the time zone is ever changed, existing `local_date` values are left as
  they were recorded. Rewriting history to a new zone would silently alter past
  streaks.

### 10.9 Retention and SD-card wear

`event_log` records every raw event, which is exactly what makes replay (§17.2)
possible – and is also a continuous write load on an SD card, which is the most
likely hardware failure in the whole system.

* **Retention:** full raw events for the last 30 days; after that, only sessions
  explicitly flagged "keep for debugging" retain their events. Aggregates
  (`session`, `block_log`, `activity_log`, `sample`) are kept **forever** – they
  are the actual history and are small.
* Writes are **batched** and the database uses WAL mode, so a session is not
  thousands of individual card writes.
* **Prefer an SSD/USB boot** or an industrial-grade card on the hub. This is a
  cheap hardware decision that prevents the most boring possible failure.
* Backups (§10.3) are what make card failure survivable, so backup health is
  shown in the companion UI (§15.1), not silently assumed.

### 10.10 Correcting history

Button counting will miscount, and a session will eventually be left running
while you walk away. If the statistics drift from reality you stop trusting them,
and once you stop trusting them you stop looking – which would kill §9.4
entirely. So editing history is a feature, not an afterthought:

* The companion UI can **edit** a logged session: reps, weight, feeling, notes,
  and the status of an individual activity.
* It can **delete** a session, or mark it **"not a real session"** (a test, a
  demo to a friend) so it is excluded from statistics and streaks but still
  visible in the raw log.
* **Auto-close:** a session with no events for 2 hours is closed automatically
  with status `abandoned` and excluded from completion rates. It is offered for
  review next time you open the UI – "did you finish this?".
* Edits set `session.edited_at` and are recorded in `event_log`, so an edited
  session is visibly edited. History is correctable, not rewritable without
  trace.
* Personal bests (§12.4) are **recomputed** after an edit, so a corrected typo
  cannot leave a phantom record standing.

## 11. Event bus topics (MQTT)

| Topic | Publisher | Example payload |
|-------|-----------|-----------------|
| `tc/presence/motion` | ESP32 presence node | `{"motion": true}` |
| `tc/presence/ble` | ESP32 presence node | `{"id": "my-phone", "rssi": -62}` |
| `tc/presence/state` | presence service | `{"state": "arrived", "user": "me"}` |
| `tc/input/button` | ESP32 button node / serial bridge | `{"node": "panel", "button": "rep", "action": "press", "age_ms": 12}` |
| `tc/input/ui` | web API (kiosk / companion taps) | `{"node": "phone", "button": "next", "action": "press", "age_ms": 0}` |
| `tc/input/intent` | input normaliser (output only) | `{"intent": "rep", "value": 1, "source": "button", "node": "panel", "ts": ...}` |
| `tc/node/status` | every input/sensor node, periodically (retained) | `{"node": "wall-rower", "online": true, "rssi": -58, "battery": 92, "fw": "1.3.2"}` |
| `tc/node/status` | broker, on behalf of a node (retained MQTT Last Will) | `{"node": "wall-rower", "online": false}` |
| `tc/node/feedback` | engine / UI | `{"node": "panel", "led": "pulse_green", "buzz": "click"}` |
| `tc/sensor/rower` | optional sensor | `{"meters": 812, "spm": 24}` |
| `tc/sensor/hr` | HR bridge | `{"bpm": 142, "rr": [412, 418]}` – the hub derives the zone from `bpm` and the user's `hr_max` (§10.1); the node only reports what it measures |
| `tc/sensor/ambient` | optional sensor | `{"temp_c": 21.4, "humidity": 48}` |
| `tc/sensor/equipment` | rower monitor / FTMS bike bridge | `{"device": "pm5", "meters": 812, "watts": 184, "cadence": 24}` |
| `tc/voice/intent` | voice service | `{"intent": "report_reps", "value": 8, "lang": "da", "text": "otte"}` |
| `tc/voice/say` | engine / presence | `{"text": "Næste øvelse: armstrækninger", "lang": "da", "priority": "high"}` |
| `tc/engine/state` | workout engine | full current state snapshot (retained) |
| `tc/engine/event` | workout engine | `{"event": "countdown", "seconds_left": 3}` |
| `tc/room/state` | workout engine (retained) | `{"phase": "work", "seconds_left": 12, "hr_zone": 4}` – consumed by Home Assistant (§12.10) |
| `tc/node/firmware` | hub | `{"node": "wall-rower", "command": "update", "url": "...", "version": "1.4.0"}` (§15.2) |

`action` is one of `press`, `long` (≥1 s), `hold` (≥3 s) or `double` (§4.2).
`node` identifies *which* panel sent it (§3.1), so a new button is a new `node`
id and nothing else.

Node-published messages carry **`age_ms`** – how long ago the press happened,
measured by the node – and **never a wall-clock timestamp**: an ESP32 has no
trustworthy clock (§12.6). The hub converts `age_ms` into its own monotonic time
on arrival, and only hub-produced messages such as `tc/input/intent` carry an
absolute `ts`.

Node liveness uses two retained messages on the same topic: the node itself
publishes a periodic status (`online: true`, plus signal strength and battery),
and it registers an MQTT **Last Will** (`online: false`) that the broker publishes
if the node disconnects ungracefully or its keepalive expires. Together they let
the UI show which inputs and sensors are actually alive (§12.6).

The broker is only reachable on the local network and uses username/password
authentication (§13).

## 12. Experience principles

These are the things that decide whether the system feels great or merely works.
They are design constraints, not polish to be added later.

### 12.1 Responsiveness

* Button feedback < 100 ms, locally generated (§4.3).
* **Pre-generate and cache TTS** for all static cues – numbers, "halfway",
  "10 seconds", "3‑2‑1", every exercise name in both languages. Cues then play
  instantly and sound identical every time. Live synthesis is reserved for
  dynamic sentences (greetings, summaries, answers).
* Latency budget per cue type:

| Cue | Budget | How |
|-----|--------|-----|
| Button click / LED | < 100 ms | Local on the node |
| Screen update after an input | < 200 ms | WebSocket push |
| Cached audio cue (countdown, next exercise) | < 150 ms | Pre-rendered WAV |
| Synthesised sentence | < 1.5 s | Piper, started as soon as the text is known |

Timing-critical cues (3‑2‑1, interval start) are **scheduled ahead of time**, not
triggered on the tick, so they land on the beat.

### 12.2 Audio design, not just speech

* Distinct, musical **earcons** for: round start, final 3‑2‑1, activity
  complete, session complete, personal best, error/degraded. A sound you
  recognise without listening beats a sentence you must parse while gasping.
* **Music ducking** – lower any music playing through the same output while a
  cue plays, then restore it.
* A **"sounds only" mode** – earcons without speech, for when you have heard the
  same plan a hundred times, and a **"silent" mode** for late evenings.
* Speech is never the only channel for something important; the screen and the
  LED always carry it too.

### 12.3 Glanceability

* From across the room you should be able to read exactly **one** thing: time or
  reps remaining. Everything else is deliberately secondary in size and contrast.
* The **LED ring is a peripheral-vision progress indicator**: colour-coded work
  (green) / rest (blue) / final seconds (amber pulse) / complete (white flash).
  Done well, you can run a whole interval set without looking at the screen.
* No information that requires reading a sentence while moving.

### 12.4 Motivation

Genuine, not gamified noise:

* Last-session comparison inline on the workout screen ("last time: 18").
* Personal-best celebration – a distinct earcon and a brief screen treatment.
* Streak shown on the idle screen.
* End-of-session summary that highlights **one** concrete positive thing.

### 12.5 Adaptivity

* Propose the next session from history (plan rotation, recovery, what you have
  been avoiding).
* Progressive-overload suggestions per exercise, based on the logged trend.
* Auto-scale down when the last session for that plan was reported "too hard";
  scale up after repeated "easy".
* Deload hint after N consecutive hard sessions.
* Progressive overload now has something to operate on: logged `actual_weight_kg`
  per set (§7.3), with suggestions rounded to the exercise's `progression_step`
  so they are achievable with the plates actually in the room.
* Programs (§7.6) are what make "what should I do today?" answerable at all;
  without them adaptivity has no horizon beyond the next session.

All suggestions are proposals shown before the session starts – the system never
silently changes a plan.

### 12.6 Graceful failure

Every failure mode has a defined behaviour and a visible reason. §7.8 already
persists state for resume; this extends it to a visible **degraded mode**
indicator so you always know *why* something is not answering.

| Failure | Behaviour |
|---------|-----------|
| Microphone unavailable / STT failing | Voice indicator turns grey with a reason; buttons and timers continue; no attempt to listen |
| MQTT broker down | The session keeps running **on timers only** – every input reaches the engine over the bus, so remote nodes, voice and the UI all stop producing intents. The kiosk shows a prominent "inputs offline" banner, `timed`/`rest` activities continue uninterrupted, and `reps` activities hold at their current count rather than being lost. Reconnect is automatic; see the replay rule below. |
| Button node offline (LWT received, or keepalive timeout) | Node greyed out in the status strip; other nodes and voice still work. `count_mode` falls back to `voice` **only when no physical button node remains online** (§5.4) |
| Sensor offline mid-activity | Fall back to the time/button goal for that activity (§5), log the gap, show the degraded badge |
| Display asleep or HDMI lost | Audio cues continue uninterrupted; display is re-woken on the next presence or input event |
| Power cut mid-session | On boot, offer "resume your session from 12 minutes ago?" from the persisted state |
| Audio output missing | Screen and LED carry all cues; a one-line warning on the idle screen |

**Replay rule after a broker outage.** Events a node buffered while it was
disconnected are replayed on reconnect, but **only if they are still relevant** –
otherwise a `next` pressed during the outage would be applied to whatever
activity happens to be current afterwards, skipping work or crediting reps to the
wrong exercise. An event is dropped when it is older than a short max age
(e.g. 10 s) or when it predates the start of the currently active activity.
Relevance is judged on hub time derived from the node's `age_ms` (§11), never on
a node-supplied timestamp.

If timer-only operation proves too fragile in practice, the fallback is an
in-process path for hub-local inputs – voice and kiosk – bypassing the broker.

### 12.7 Low friction

* **Quick start**: one long press of Start on the wall panel launches your usual
  plan – no menus, no list, no selection. The best training experience is the one
  with the least friction before the first rep. (If a node by the door turns out
  to be worth it, it is just another `node` id – see §3.1.)
* Presence-triggered wake-up means the system is already on the right screen when
  you walk in (§6).
* Warm-up and cool-down are part of the plan model, not something to remember,
  and the system knows the difference so stats are not polluted by warm-ups.

### 12.8 Accessibility and practicality

* **Sweaty hands**: large physical buttons and large on-screen hit areas; no
  gesture or precision-touch requirement anywhere.
* **Bright room**: high contrast, large type, no thin fonts, no pastel on white.
* **Hold-to-stop emergency stop**: holding Done for 3 s ends everything
  immediately (§4.2). The hold is deliberate – it must be impossible to trigger
  by brushing past the panel – but it works in every engine state, with no
  confirmation dialogue. On single-button nodes such as `floor`, which have no
  Done button, a 3 s hold on Rep/Lap does the same thing, so a stop is always
  within reach of whichever node you are standing at.
* Audio and visual channels are redundant, so the system is usable with the
  sound off or without looking at the screen.

### 12.9 Readiness and recovery

**Decision – this is what `rr` is for.** The HR payload (§11) carries
beat-to-beat intervals, and until now nothing consumed them. They are kept, and
their purpose is readiness:

* A **morning or pre-session reading**: stand still for 60 s on the ready screen
  while the strap is on, and the hub computes **RMSSD** and resting HR from the
  `rr` stream, storing one row in `hrv_reading` (§10.1).
* Readiness combines three cheap signals: HRV trend against your own rolling
  baseline, recent training load (§9.4), and your reported feeling from recent
  sessions (§12.4).
* The output is **one sentence on the idle screen** – "you look recovered, good
  day for intervals" or "take it easy today" – not a score out of 100. A single
  number invites optimising the number instead of the training.
* It is **advisory only**. It can bias the suggested session (§12.5) but never
  blocks or changes a plan you chose.
* It degrades silently: no strap, too few readings, or less than ~2 weeks of
  baseline means no line is shown at all rather than a meaningless one.
* `rr` is noisy from an optical sensor. Artefact filtering is required, and a
  chest strap is strongly preferred for readings (§5.1).

### 12.10 The room as an output (Home Assistant)

Home Assistant is listed as a presence *input* (§6). The stronger idea is using
it as an **output**, so the room itself signals state in peripheral vision:

* **Work/rest lighting** – the room lights shift colour with the LED ring: work
  is bright/white, rest is a calm colour, the last 3 s of rest pulse. You can
  then run a whole interval session without looking at the screen at all, which
  matters when you are face-down on a mat.
* **Automatic fan when HR is high** – fan on above a zone threshold or during
  long work blocks, off during cool-down. In a garage gym this is one of the
  cheapest genuinely great wins available.
* Implementation is one retained topic, `tc/room/state` (§11), which Home
  Assistant subscribes to. The engine publishes state and knows nothing about
  lights, fans or any specific automation – and the system is fully functional
  with Home Assistant absent.
* Everything here is opt-in. Lights changing unexpectedly in a shared house is
  an anti-feature.

### 12.11 Coach personality and phrase variation

Hearing the identical sentence every single morning becomes grating within a
week, and a grating coach gets muted. Each cue therefore draws from a **pool of
phrasings**, varied by time of day, session type, context (a personal best, a
long gap since the last session) and simple randomisation without immediate
repeats. The tone is calm and encouraging, never drill-sergeant – you can change
your mind about that later because the phrasings are data, not code.

The pre-generated TTS cache (§12.1) is exactly the right mechanism: variants are
synthesised ahead of time, so variation costs latency nothing. Safety- and
timing-critical cues ("three, two, one") are deliberately **not** varied –
predictability matters more there than novelty.

### 12.12 Music

Beyond ducking the volume for cues (§12.2):

* **Auto play/pause aligned to the session** – music during work, quieter or
  paused during rest and between-block instruction, resumed automatically.
* **Tempo-matched selection** – a faster playlist for intervals, something
  calmer for mobility and cool-down.
* Control is via whatever already plays music in the room (Home Assistant media
  player, Spotify Connect, Bluetooth). The hub is **not** becoming a music
  player; it only sends play/pause/volume.

## 13. Security and privacy

The core loop is LAN-only, but the companion UI (§9) leaves the kiosk, so this
needs stating explicitly.

| Surface | Protection |
|---------|-----------|
| MQTT broker | LAN-only bind, username/password per client, no anonymous access; nodes use per-node credentials so one can be revoked |
| Web API + UI (kiosk) | **Not** open to the LAN: the unauthenticated kiosk endpoints are bound to localhost (or gated by a per-device token provisioned at kiosk setup), so only the local display can use them. Every other LAN client goes through the authenticated companion path below |
| Web API + UI (companion) | Login required – single user, hashed password, long-lived session cookie, CSRF protection on state-changing requests |
| Transport | HTTPS on the LAN with a locally issued certificate; never plain HTTP for the companion UI |
| Remote access | **VPN only** (Tailscale / WireGuard). No port forwarding, no exposing the hub to the internet |
| Backups | Encrypted at rest (§10.3) |
| Audio | Never persisted; transcripts are discarded after intent parsing |
| Health data | Sensitive by default (§10.5); no third-party analytics; sync is off by default and opt-in |

Secrets (broker passwords, Wi‑Fi credentials, sync tokens) live in a local
config file or environment, **never in this repository**. Firmware reads its
credentials from a provisioning step, not from committed source.

## 14. Proposed repository layout

```
TrainingCenter/
├── docs/                 design notes, wiring diagrams, photos
├── hub/                  Python services running on the Raspberry Pi
│   ├── engine/           workout model + state machine (pure Python, unit tested)
│   ├── input/            input normaliser: buttons/voice/UI/sensors → control intents
│   ├── voice/            wake word, STT, intent parser, TTS (+ cue cache)
│   ├── presence/         presence fusion logic
│   ├── sensors/          HR bridge, rower, ambient, equipment integrations
│   ├── serial_bridge/    Arduino serial ⇄ MQTT bridge
│   ├── api/              FastAPI + WebSocket server (kiosk + companion)
│   ├── storage/          SQLite access, migrations (§10.7)
│   ├── sync/             backup + optional outbound sync (§10.3, §10.4)
│   ├── room/             publishes tc/room/state for Home Assistant (§12.10)
│   └── tests/            unit tests + recorded event logs for replay (§17)
├── ui/
│   ├── kiosk/            glanceable display UI
│   ├── companion/        phone/laptop UI: plans, control, statistics
│   └── shared/           components, API client, WebSocket client
├── firmware/
│   ├── esp32-presence/   BLE scan + mmWave → MQTT
│   ├── esp32-buttons/    buttons + LED ring + buzzer → MQTT (one build, many nodes)
│   ├── esp32-hr/         BLE heart-rate bridge → MQTT
│   └── arduino-io/       serial I/O bridge
├── assets/
│   ├── audio/            pre-rendered TTS cues and earcons (§12.1, §12.2)
│   └── exercises/        short demo clips, one per exercise (§9.8)
├── plans/                workout plans (YAML), programs/, exercises.yaml
├── docs/adr/             decision records (§19)
└── deploy/               systemd units, docker-compose, kiosk setup scripts
```

Language choice: **Python** for the hub (best ecosystem for Whisper/Piper/
openWakeWord on a Pi), **Arduino/C++ (PlatformIO)** or ESPHome for the ESP32s,
**HTML/JS** for the UI. Services run as systemd units (or docker-compose).

## 15. Operations

A system that runs in a garage and is maintained by one person fails in
predictable ways. These sections are not features – they decide whether the
project survives contact with daily use.

### 15.1 Observability

Raw events are stored (§10.1), but stored data is not observability. What is
needed is the ability to answer "why did it just do that?" without a debugger:

* A **system health page** in the companion UI: every node with its last-seen
  time, RSSI, battery and firmware version (§11); broker connection; microphone
  and STT status; disk space; last successful backup (§10.3); database schema
  version.
* A **decision trace** on the session detail view: for each engine transition,
  which intent caused it, from which source and node. This is the single most
  useful debugging artefact and it falls out of `event_log` for free.
* **Voice recognition accuracy tracking.** Every recognition stores the parsed
  intent, the confidence and whether it was undone or corrected within a few
  seconds. A "misrecognitions" view then shows which phrases fail, which is the
  only way the intent grammar (§8.3) actually improves. Only the parsed text is
  kept, never audio (§13), and the view can be cleared at any time.
* **Structured logs** with a session id, plus a visible banner for any degraded
  mode (§12.6). The failure must be visible in the room, not only in a log file.

### 15.2 Firmware provisioning and OTA

The design implies 3–5 ESP32 nodes (§3). Without this, every firmware change
means walking round the room with a USB cable, which in practice means firmware
stops being changed.

* **One build, many nodes**: node identity (`node` id, role) comes from
  provisioned configuration, not from a separate firmware image per node.
* **Wi-Fi provisioning** on first boot via a temporary soft-AP captive portal;
  credentials are stored in NVS and never committed (§13).
* **OTA updates** over the LAN, triggered from the hub via `tc/node/firmware`
  (§11), with a signed image, A/B partitions and automatic rollback if the new
  image fails to connect. A node bricked on a shelf is recoverable; a node
  bricked behind a wall panel is not.
* Each node reports its **firmware version** in `tc/node/status`, so the health
  page can show what is out of date.
* Updates are **never applied during a session**.

### 15.3 Resource budget

The hub runs Whisper, Piper, the broker, the API, the UI and the database at the
same time, on a Pi, in a garage:

* Measure before assuming: wake word is continuous, STT is bursty, TTS is mostly
  cache hits (§12.1). If the budget does not fit, the first lever is a smaller
  Whisper model, then moving STT to another machine (§16, open question 1).
* **Thermal throttling** is a real risk in a hot or cold garage; the hub needs a
  heatsink or fan, and the health page shows CPU temperature.
* The engine's timing must not depend on STT load – timers run in their own loop
  so a slow transcription can never stretch an interval.
* Storage wear and retention are covered in §10.9.

## 16. First-run setup

Everything above assumes a configured system. Getting there must be a described
path, not folklore:

1. **Hub install** – flash the image, run the installer, services start.
2. **Create the user** – name, language, units; this writes the single `user`
   row (§10.1).
3. **Provision each node** – soft-AP, Wi-Fi credentials, node id and role
   (§15.2), confirmed by the node appearing on the health page.
4. **Pair the heart-rate strap** – scan, select, confirm a live BPM reading.
5. **Calibrate `hr_max`** – age formula as a starting point, with an explicit
   option to enter a measured value; zones derive from it (§11).
6. **Pick a starter plan or program** – the system ships with a few, so the
   first session is possible without using the plan editor at all.
7. **Optional integrations** – Home Assistant (§12.10), backups (§10.3), remote
   access (§10.6).

Every step is skippable and resumable, and the system is usable after step 2
with a keyboard alone.

## 17. Testing strategy

"Easy to unit-test without hardware" (§7.8) is a property, not a strategy. For a
system this event-driven, the strategy has three parts.

### 17.1 Unit tests and a hardware-free mode

The engine is pure logic over an injected clock, so every goal type, transition
and edge case is testable with no hardware and no real time passing. Beyond
that, the whole system must run on a laptop: **fake nodes** publish button,
sensor and HR messages to a local broker, and a **simulated session** can be
driven end to end. Development that requires standing in the garage is
development that does not happen.

### 17.2 Session replay

`event_log` already records every raw event with its timing (§10.1). Feeding a
recorded log back through the engine and asserting the outcome gives three
things from one mechanism:

* **Regression tests** – real sessions become test fixtures, so a change that
  would have miscounted last Tuesday's workout fails in CI.
* **Bug reproduction** – "it skipped an exercise" becomes a reproducible case
  attached to the report, rather than a story.
* **Debugging** – stepping through what the engine saw is how §15.1's decision
  trace is verified.

Replay requires the engine to be deterministic given an event sequence and a
clock: no hidden wall-clock reads, no unordered concurrency in the decision
path. That constraint is worth accepting.

### 17.3 What else gets tested

| Area | Approach |
|------|----------|
| Plan/program YAML | Schema validation in CI, so a malformed plan is caught before it is ever loaded (§7.2) |
| Migrations | Applied to a copy of a real database, forwards only, with a restore test (§10.7) |
| Voice intents | A fixture set of Danish and English phrases parsed to expected intents, no audio required (§8.3) |
| UI | Smoke tests on the kiosk screens at the real display resolution; glanceability is checked by eye, not by test (§12.3) |
| Firmware | Button debounce, press/long/hold discrimination (§4.2) and reconnect behaviour on a bench node before deployment |

## 18. Explicit non-goals

Writing down what is *out*, and why, is what stops it creeping back in:

| Not building | Why |
|--------------|-----|
| Camera / pose estimation / form correction | Hard, unreliable, and a camera in a home gym is a privacy cost that outweighs the benefit (§5.3). Demo clips (§9.8) cover the real need |
| A native mobile app | The companion web UI (§9) does everything a native app would, with no app store, no signing, no release process |
| Social features, leaderboards, sharing | This is a private home gym. Comparison with others is a different product, and a worse one for this purpose |
| Nutrition and weight tracking | A large separate domain with its own data model; existing apps do it well |
| A multi-tenant cloud service | Local-first is a decision (§10.2), not a limitation. Hosting other people's health data changes the project entirely |
| Multi-user accounts | Guest/partner mode (§9.9) covers the real case without an account system |
| Being a music player | Control an existing player, do not become one (§12.12) |

If one of these ever becomes genuinely wanted, it should be reopened explicitly
as a decision (§19) rather than arrived at by accretion.

## 19. Decision log

This document is a draft with open questions (§21). As those are answered, the
*rationale* needs somewhere to live – otherwise the same debates are re-litigated
and the open-questions list silently rots.

* Each resolved question becomes a short record in `docs/adr/` – context, the
  decision, the alternatives considered, the consequences – numbered and dated.
* Decisions already embedded in this document (local-first, `user_id` from day
  one, buttons as a first-class input, kg as the only unit, no camera) are
  migrated there as the first records.
* When a decision is reversed, the old record is **superseded, not deleted**.
  The reasoning that turned out to be wrong is the most useful part.
* An answered open question is removed from §21 and linked to its record.

## 20. Roadmap (incremental, each step usable on its own)

Buttons move **early** – they make the MVP genuinely usable without any voice
stack at all. Heart rate moves into the mid-game. Web statistics follow once
there is history worth looking at.

1. **MVP – interval timer on the display**: engine with `timed` and `rest`
   activities, YAML plans, kiosk UI, cached TTS cues + earcons via Piper (no
   voice input yet), start via keyboard. History saved to SQLite.
2. **Buttons & the input model**: ESP32 control panel (Rep/Lap, Start/Pause,
   Next, Previous, Done) with LED ring and buzzer, the unified intent path
   (§4.1), press semantics (§4.2) and the <100 ms feedback budget (§4.3). After
   this step the system is fully usable hands-on, with no voice at all.
3. **Reps & hybrid sets**: `reps` and `reps_in_time` activities with
   `count_mode: manual_button`, live rep counter, undo, and the "how did it go?"
   dialogue for the modes that still need it.
4. **Voice commands**: wake word + faster-whisper + intent parser for
   start/pause/next/done in Danish and English, feeding the same intent path.
5. **Presence & greeting**: ESP32 with mmWave + BLE scan, greeting on arrival,
   quick-start, display sleep on leave.
6. **Heart rate**: BLE HRM, live zone on the kiosk screen, `until_hr_below`
   rests, HR stored as samples for the stats screens.
7. **Distance activities & check-ins**: `distance` type with timed check-ins;
   rower sensor (hall/reed sensor via Arduino, or reading the rowing monitor
   over Bluetooth if supported).
8. **Strength & load**: `weighted_reps`, target/actual weight, the enriched
   exercise library and the equipment check (§7.3, §7.5, §9.7).
9. **Block goals**: AMRAP and "for time" with scores and personal bests (§7.4).
10. **Companion UI & statistics**: login, remote control, history, streaks,
    per-exercise progression, completion rates, post-session review, history
    correction (§10.10), export.
11. **Plan editor** in the companion UI, with templates, the exercise picker,
    validation and import/export.
12. **Programs & the time budget**: multi-week schedules, "what should I do
    today?", and "I have 20 minutes" (§7.6, §7.7).
13. **Backup & optional sync**: encrypted nightly backup, then VPN remote access;
    outbound sync only if remote access proves insufficient.
14. **Adaptivity**: progressive-overload suggestions, auto-scaling from feeling
    scores, deload hints.
15. **The room as an output**: Home Assistant work/rest lighting and automatic
    fan control (§12.10).
16. **Readiness**: HRV from `rr`, one line on the idle screen (§12.9).
17. Nice-to-haves: extra sensors from Tier 2 (§5.2), demo clips (§9.8), coach
    phrase variation (§12.11), music automation (§12.12), guest mode (§9.9),
    nudges (§9.10), satellite button nodes.

**Running alongside, not after:** migrations and schema versioning (§10.7) from
the first database; the replay harness (§17.2) as soon as `event_log` exists;
OTA (§15.2) before the second ESP32 node is mounted; the health page (§15.1) as
soon as there is more than one node to lose.

## 21. Open questions

1. Which Raspberry Pi model(s) do you have (Pi 3/4/5, RAM)? This decides whether
   Whisper runs on the hub or on another machine.
2. What display will be used (TV with HDMI, small monitor, touch screen)? Is touch useful?
3. Phone: iPhone or Android? (Affects how reliably the phone can act as a BLE beacon.)
   Would a small BLE key-fob be acceptable as the "it's me" signal?
4. Which presence/motion sensors do you already have (PIR, mmWave, ultrasonic)?
5. Preferred default language for the voice – Danish, English, or mirror the command?
6. Wake word preference ("Hey Coach", "Hej Træner", something else)?
7. What equipment is in the room (rower model, bike, weights)? Does the rower have
   Bluetooth output we could read distance from?
8. Is it OK for the system to be fully offline (no cloud), or are cloud speech
   services acceptable as a fallback for better Danish recognition?
9. Should music be playing from the same speaker (then we need audio ducking)?
10. **Per-rep or per-set counting?** Do you want to press a button for *every*
    rep (precise, but busy during fast sets), or once per set/round? This
    decides whether the Rep button is the primary interaction or an occasional
    one, and whether a fast 20-rep set should fall back to `voice`.
11. **How many input nodes, and where?** One wall panel, or also a satellite by
    the rower and a foot button? Is the display a touch screen (§3.1)?
12. **Which equipment already has BLE?** Rower monitor, bike, heart-rate strap,
    scale. Reading an existing monitor is far cheaper than building a sensor.
13. **Do you want remote access at all** – checking stats from work/holiday – or
    is LAN-only sufficient? If yes, is a VPN (§10.6) acceptable, or do you
    specifically want a hosted datastore?
14. **Single user forever?** `user_id` is in the schema either way (§10.1), but
    knowing whether family members will use it affects the UI and the greeting
    logic.
15. **How much motivation layer do you want?** Streaks and PB celebrations, or a
    deliberately plain system?
16. Is a "sounds only"/silent mode needed (early mornings, late evenings)?
17. **How is strength trained here** – barbell with plates, dumbbells,
    kettlebells, machines? This decides the `progression_step` defaults and
    whether per-side weights need modelling (§7.3).
18. **Do you want programs, or just a library of plans?** Multi-week scheduling
    (§7.6) is the biggest single feature here, and only worth it if you would
    actually follow one.
19. **Is Home Assistant already running**, and are the gym lights and a fan on
    it? That decides whether §12.10 is a weekend job or a project.
20. **Would you use a readiness line** (§12.9), or is it the kind of metric you
    would start optimising instead of training?
21. **How much history correction do you expect to need** (§10.10) – is a simple
    delete enough, or is full editing worth building?
22. Does anyone else ever train in the room (guest mode, §9.9)?
