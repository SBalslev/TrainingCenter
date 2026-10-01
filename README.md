# TrainingCenter

Tools and apps for a private home training center built from Raspberry Pis,
ESP32s, Arduinos and assorted sensors/buttons.

Planned features:

- A display that greets you visually and by voice when you enter the room (presence detection).
- A workout helper that tracks time and reps, shows the list of activities and the
  current activity, and announces each new interval by voice.
- Support for timed intervals (e.g. 1-minute EMOM), rep sets, "reps within a time limit"
  sets, and distance activities (e.g. 1600 m rowing) with periodic check-ins.
- **Big physical buttons as a first-class input** – one press per rep or round,
  start/pause, next – with sub-100 ms LED/buzzer feedback. Buttons, voice, web
  taps and sensors all become the same control intents.
- **Optional sensors**, none of them required: BLE heart-rate strap (live zone,
  rest-until-recovered), reed/hall sensors on the rower or bike, ambient
  temperature.
- A history of completed workouts, plus a **companion web UI** for phone/laptop:
  plan editor, remote control, streaks, per-exercise progression and completion
  statistics.
- Voice commands in Danish and English.
- **Local-first**: a full workout runs with the internet down. Encrypted backups,
  VPN remote access and optional outbound sync are separate, disableable concerns.

See [docs/DESIGN.md](docs/DESIGN.md) for the proposed design, roadmap and open questions.
