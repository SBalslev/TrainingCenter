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
in scope (§10.4), a cloud dependency in the core loop is not.

## 2. High-level picture

```
                ┌──────────────────────────── Raspberry Pi (hub, "brain") ────────────────────────────┐
                │                                                                                      │
 ESP32 presence │   ┌──────────────┐    ┌───────────────┐    ┌───────────────┐    ┌────────────────┐   │
 node(s) ──MQTT─┼──▶│  presence    │    │ voice service │    │ workout engine│    │ storage        │   │
 (BLE + mmWave) │   │  service     │    │ wake word,    │    │ state machine,│    │ SQLite history │   │
                │   └──────┬───────┘    │ STT, intents, │    │ timers, plans │    └───────▲────────┘   │
 ESP32 buttons, │          │            │ TTS           │    └──────┬────────┘            │            │
 LEDs ──── MQTT─┼──────────┤            └──────┬────────┘           │                     │            │
                │          │   ┌─────────────┐ │                    │                     │            │
 Arduino Uno ───┼─ serial ─┼──▶│ serial      │ │                    │                     │            │
 (USB)          │          │   │ bridge      │ │                    │                     │            │
                │          │   └──────┬──────┘ │                    │                     │            │
                │          ▼          ▼        ▼                    ▼                     │            │
                │   ═══════════════════ MQTT event bus (Mosquitto) ═══════════════════════╧══          │
                │                                   │                                                  │
                │                          ┌────────▼────────┐                                         │
                │                          │ web API + UI    │  WebSocket ──▶ Chromium kiosk on display │
                │                          │ (FastAPI)       │                                         │
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
* **The display is a web page** shown in a full-screen Chromium kiosk. That
  makes the UI easy to build/iterate and also viewable from a phone/laptop.

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
 sensor event  ─┘      (hub)        {intent, value, source, node, ts}
```

* All inputs land on `tc/input/*` (voice publishes `tc/voice/intent`, which the
  normaliser treats the same way) and are converted into a **control intent**
  using exactly the vocabulary of §8.3 (`start_workout`, `pause`, `next`,
  `done`, `report_reps`, `feeling`, …).
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
| **Done / Confirm** | Complete the current activity, or confirm a feedback prompt | Hold 3 s: stop the session (emergency stop, §12.8) | – |
| **Hard / Easy** (optional pair) | Record `feeling` without speaking | – | – |

For `reps` activities the Rep button **replaces the "how many did you do?"
dialogue entirely** – the count is already known, so the engine skips straight to
rest/next.

### 4.3 Feedback and latency

Every press must produce immediate multi-sensory confirmation: LED ring flash +
short buzzer click (locally on the node, not waiting for the hub) and the
on-screen counter incrementing.

> **Non-functional requirement:** button press → visible/audible confirmation in
> **< 100 ms** end-to-end. This is the single biggest "feels great vs. feels
> broken" factor and is treated as a hard requirement, not a nice-to-have.

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
| `manual_button` | One Rep/Lap press = one rep (§4.2) |
| `sensor` | A sensor publishes the count/distance; buttons still work as an override |
| `voice` | Reported verbally after the set ("I did eight") |
| `auto` | Derived from the timer alone (e.g. `timed` activities) |

Default: `manual_button` when a control node is online, otherwise `voice`.

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

* **Plan** – a named workout ("Morning EMOM", "Row + core"), made of blocks.
* **Block** – a group of activities repeated N rounds (e.g. "3 rounds of: …").
* **Activity** – one exercise with a **goal type**:

| Type | Example | Completion | System behaviour |
|------|---------|-----------|------------------|
| `timed` | 1 min burpees | Timer ends | Count down, voice cues at "halfway", "10 seconds", "3‑2‑1", announce next. Covers EMOM / 1‑minute interval sessions. |
| `reps` | 10 push-ups | Target reps counted (button/sensor), or I say "done"/"færdig" | Shows target reps and a live counter, waits (no time limit), optional rest timer afterwards. With `count_mode: manual_button` the count is already known, so the "how many did you do?" question is skipped entirely (§4.2). |
| `reps_in_time` | 10 reps within 1 min | Done **or** timer ends | When finished, asks "How did it go? / Hvordan gik det?" → answers like "all of them", "eight", "too hard", "let's do the last 2" → engine adapts (logs actual reps, optionally adds a mini-set for the remaining reps). |
| `distance` | 1600 m rowing | I say "done", or a sensor reports distance | Periodic check-ins (e.g. every 2 min or every 400 m if distance is known): "How is it going? / Hvordan går det?" → answers like "fine", "halfway", "800 meters", "pause". |
| `rest` | 30 s rest | Timer ends | Shows what is next so I can get ready. |

### 7.2 Plan file format (YAML, stored in git)

```yaml
name: Morning EMOM
language: da            # default language for spoken cues (da | en)
count_mode: manual_button   # plan-wide default, overridable per activity (§5.4)
blocks:
  - name: Warm-up
    rounds: 1
    activities:
      - { type: timed, exercise: jumping_jacks, duration: 60s }
      - { type: timed, exercise: air_squats, duration: 60s }
  - name: EMOM
    rounds: 5
    activities:
      - { type: reps_in_time, exercise: burpees, reps: 10, time_limit: 60s,
          count_mode: manual_button }
      - { type: reps, exercise: push_ups, reps: 10, count_mode: manual_button }
      - { type: rest, duration: 30s, until_hr_below: 130, max_duration: 90s }
  - name: Finisher
    rounds: 1
    activities:
      - { type: distance, exercise: rowing, distance: 1600m, check_in_every: 2min,
          count_mode: sensor, hr_zone: 3 }
```

Schema additions beyond the original draft:

| Field | Applies to | Meaning |
|-------|------------|---------|
| `count_mode` | plan, block, activity | How reps/laps are counted: `manual_button`, `sensor`, `voice`, `auto` (§5.4). Innermost wins. |
| `until_hr_below` | `rest` | End the rest when heart rate drops below this value – requires a Tier‑1 HR sensor (§5.1). |
| `max_duration` | `rest` | Safety cap for `until_hr_below`, and the fallback when no HR sensor is available. |
| `hr_zone` | any work activity | Target zone shown on screen; out-of-zone is a hint, never a blocker. |

**Degradation rule:** if a field depends on a sensor that is missing or offline,
the engine falls back to the plain time/rep/button behaviour and shows a degraded
indicator (§12.6). A plan never becomes unrunnable because a sensor is absent.

An **exercise library** (`exercises.yaml`) holds names in both languages and
optional cues, so the voice can say "Armstrækninger" or "Push-ups":

```yaml
push_ups:      { en: Push-ups,      da: Armstrækninger }
burpees:       { en: Burpees,       da: Burpees }
air_squats:    { en: Air squats,    da: Squats }
jumping_jacks: { en: Jumping jacks, da: Sprællemænd }
rowing:        { en: Rowing,        da: Roning }
```

### 7.3 Engine state machine

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
| ACTIVE (`reps`, `reps_in_time`) | `rep` (button press or sensor pulse) | stays ACTIVE, increments the counter; completes the activity when the target is reached |
| ACTIVE | `undo_rep` (long press) | stays ACTIVE, decrements the counter (never below 0) |
| REST (`until_hr_below`) | HR below threshold, or `max_duration` reached | next activity |

* The engine is **pure logic driven by a clock tick and events** (voice intents,
  button presses, sensor data). It emits events (`activity_started`,
  `countdown`, `ask_feedback`, `session_finished`, …) which the UI and voice
  service render. This makes it easy to unit-test without hardware.
* All timers use a monotonic clock on the hub (not on the ESP32s).
* Every state change is persisted so a crash/reboot can resume the session.

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

Buttons on the ESP32 give a **fallback** for the most important commands when
voice fails (noise, out of breath).

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

## 9. Web UI: kiosk display and companion

Big, high-contrast, readable from 3–4 m while sweating.

* **Idle / welcome**: clock, greeting, last session summary, "say *Hey Coach, start* to begin".
* **Workout screen**:
  * Large current activity name + big countdown / rep target / distance.
  * Progress ring or bar for the current activity, round counter ("Round 3/5").
  * Side list of all activities in the plan with the current one highlighted and finished ones ticked.
  * "Up next" preview.
  * Small indicator showing the microphone state (listening / heard: "næste").
* **Summary screen**: total time, per-activity results, feeling scores.
* **History screen**: calendar/list of sessions; say "show history / vis historik".

Tech: plain HTML/CSS + a small JS framework (e.g. Svelte or vanilla), live
updates over WebSocket from the API service.

## 10. Data, history and sync (G3)

SQLite database on the Pi (backed up nightly to another machine/USB stick):

```
session(id, plan_name, started_at, ended_at, status, notes)
activity_log(id, session_id, block, round, exercise, type,
             target_reps, actual_reps, target_seconds, actual_seconds,
             target_meters, actual_meters, feeling, started_at, ended_at)
event_log(id, session_id, ts, source, kind, payload_json)   -- raw events for debugging/analysis
```

Plans and the exercise library live as YAML files in this repository; history
lives only in the database (it is personal data, not source code).

## 11. Event bus topics (MQTT)

| Topic | Publisher | Example payload |
|-------|-----------|-----------------|
| `tc/presence/motion` | ESP32 presence node | `{"motion": true}` |
| `tc/presence/ble` | ESP32 presence node | `{"id": "my-phone", "rssi": -62}` |
| `tc/presence/state` | presence service | `{"state": "arrived", "user": "me"}` |
| `tc/input/button` | ESP32 button node / serial bridge | `{"button": "next"}` |
| `tc/sensor/rower` | optional sensor | `{"meters": 812, "spm": 24}` |
| `tc/voice/intent` | voice service | `{"intent": "report_reps", "value": 8, "lang": "da", "text": "otte"}` |
| `tc/voice/say` | engine / presence | `{"text": "Næste øvelse: armstrækninger", "lang": "da", "priority": "high"}` |
| `tc/engine/state` | workout engine | full current state snapshot (retained) |
| `tc/engine/event` | workout engine | `{"event": "countdown", "seconds_left": 3}` |

The broker is only reachable on the local network and uses username/password
authentication.

## 14. Proposed repository layout

```
TrainingCenter/
├── docs/                 design notes, wiring diagrams, photos
├── hub/                  Python services running on the Raspberry Pi
│   ├── engine/           workout model + state machine (pure Python, unit tested)
│   ├── voice/            wake word, STT, intent parser, TTS
│   ├── presence/         presence fusion logic
│   ├── serial_bridge/    Arduino serial ⇄ MQTT bridge
│   ├── api/              FastAPI + WebSocket server
│   ├── storage/          SQLite access
│   └── tests/
├── ui/                   web UI for the kiosk display
├── firmware/
│   ├── esp32-presence/   BLE scan + mmWave → MQTT
│   ├── esp32-buttons/    buttons + LED ring → MQTT
│   └── arduino-io/       serial I/O bridge
├── plans/                workout plans (YAML) + exercises.yaml
└── deploy/               systemd units, docker-compose, kiosk setup scripts
```

Language choice: **Python** for the hub (best ecosystem for Whisper/Piper/
openWakeWord on a Pi), **Arduino/C++ (PlatformIO)** or ESPHome for the ESP32s,
**HTML/JS** for the UI. Services run as systemd units (or docker-compose).

## 15. Roadmap (incremental, each step usable on its own)

1. **MVP – interval timer on the display**: engine with `timed` and `rest`
   activities, YAML plans, web UI in kiosk mode, TTS cues via Piper (no voice
   input yet), start via keyboard/button. History saved to SQLite.
2. **Voice commands**: wake word + faster-whisper + intent parser for
   start/pause/next/done in Danish and English.
3. **Reps & hybrid sets**: `reps` and `reps_in_time` activities with "how did it
   go?" dialogue and adaptive "do the last 2" follow-ups.
4. **Presence & greeting**: ESP32 with mmWave + BLE scan, greeting on arrival,
   display sleep on leave.
5. **Distance activities & check-ins**: `distance` type with timed check-ins;
   optional rower sensor (hall/reed sensor via Arduino, or reading the rowing
   monitor over Bluetooth if supported).
6. **History & stats screens**, progress trends, simple suggestions for next workout.
7. Nice-to-haves: heart-rate strap over BLE, music ducking during voice cues,
   Home Assistant integration (lights on arrival), plan editor in the UI.

## 16. Open questions

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
