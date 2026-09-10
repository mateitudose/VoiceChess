# VoiceChess
Link to the demo video of the project: https://youtu.be/a33Phzzp-k0

<img width="1024" height="768" alt="79A6BB0E-306D-4A46-9292-D3522A3C612A_1_105_c" src="https://github.com/user-attachments/assets/9faac36c-38a7-41a6-a285-309dde7c8e05" />

A voice-controlled automatic chess board, built as a Cyber-Physical Systems course
project. You **say** your move ("pawn to e4"), the board **confirms it out loud**,
a gantry with a claw physically moves your piece, then Stockfish replies and the
robot moves its own piece too. No screen, no phone, no hands — designed for
accessibility (handless, immobile, or blind players).

```
Voice ──> Brain (Raspberry Pi 5) ──> ESP32 (FluidNC) ──> motors + claw
           · Vosk speech-to-text        · stepper pulses
           · phonetic move matching     · homing / limits
           · python-chess + Stockfish
           · text-to-speech feedback
```

Everything in this repo runs as **one Python process** on the Pi. One turn:

> you say *"pawn to e4"* → offline STT → phonetic match against the current legal
> moves → *"You said Pawn to e4"* → gantry moves your piece → Stockfish thinks →
> *"A I plays Knight to f6"* → gantry moves its piece → listening again.

The key trick: spoken chess letters are highly confusable (b/d/e/g/p/t/c/z), so we
never transcribe freely. At any position there are only ~20–40 legal moves — we
render each as the phrases a person might say and pick the best **phonetic** match
to what was heard. The spoken read-back catches the rare miss.

## Repo layout

| Path | What it is |
|---|---|
| `voice_matching/` | Vosk STT + phonetic matching (Role 1) |
| `chess_ai/` | Rules authority, Stockfish opponent, text-to-speech (Role 2) |
| `motion/` | Square→XY→G-code motion planning (Role 3) |
| `orchestrator/` | Turn state machine (Role 5) + ESP32 serial link (**stub** until Role 4) |
| `main.py` | Entry point — wires the modules together |
| `speak_test.py` | Standalone audio check — run this first on new hardware |
| `docs/` | `pi-setup.md` (Pi runbook), `interfaces.md`, `agents.md`, `changelog.md`, per-role READMEs |
| `tests/` | pytest suites |

## Essential commands

Everything runs from the repo root. Stockfish must be installed (see [Setup](#setup)).

```bash
# --- Setup (once) ---
pip install -r requirements.txt                    # Python deps (not Stockfish — that's a system binary)
pip install pyttsx3                                 # optional: spoken audio on Windows

# --- Run the game ---
python main.py                                      # live mic if available, else auto-falls back to text
python main.py --text                               # force text mode — type your moves, no mic/model needed
python main.py --text --script "e2e4,e7e5,g1f3"     # scripted, fully hardware-free (best quick smoke test)
python main.py --tts espeak                         # live mic + spoken feedback (Pi: espeak-ng)
python main.py --tts espeak --voice male            # ...with the male narrator voice
python main.py --text --tts espeak --voice female   # typed input + spoken feedback (female = default)
python main.py --tts pyttsx3                        # spoken feedback on a Windows dev box

# tunables (combine with any of the above):
#   --skill 0-20   opponent strength (default 5)
#   --think 0.5    AI seconds per move
#   --turns 40     max turns (demo safety limit)
#   --tts print|espeak|pyttsx3|piper   audio backend (default print — just prints [SPEAK] lines)
#   --voice female|male                narrator voice preset (espeak backend only, default female)

# --- Audio & voice hardware checks ---
python speak_test.py                                # do you hear the spoken test sentences?
python -m voice_matching.demo                       # list audio devices + live mic recognition loop

# --- ESP32 / G-code output ---
python serial_test.py /dev/ttyUSB0                  # prove the USB serial link in isolation
python main.py --text --script "e2e4,e7e5" --serial /dev/ttyUSB0   # stream real G-code to the ESP32
# (without --serial, planned G-code is printed as [SERIAL-STUB] lines instead of sent)

# --- Tests ---
python -m pytest -q                                 # whole suite (a Stockfish-less box skips 1 test — fine)
python -m pytest tests/test_voice_matching.py -q    # Role 1: phonetic matching
python -m pytest tests/test_chess_ai.py -q          # Role 2: rules + Stockfish + TTS fallback
python -m pytest tests/test_motion.py -q            # Role 3: square→mm + G-code planner
```

## Setup

Needs **Python 3.10+** and the **Stockfish binary** (a system program, not pip):

```
Windows:            winget install --id Stockfish.Stockfish -e
Raspberry Pi/Debian: sudo apt install stockfish
```

Then install the Python packages:

```bash
pip install -r requirements.txt
```

That's everything for **text mode** — the whole pipeline with typed moves instead
of a microphone:

```bash
python main.py --text                            # type your moves
python main.py --text --script "e2e4,g1f3"       # scripted, fully hands-free
```

### Live voice (microphone)

Download a Vosk model (~40 MB) into `voice_matching/model/`:

```bash
cd voice_matching
wget https://alphacephei.com/vosk/models/vosk-model-small-en-us-0.15.zip
unzip vosk-model-small-en-us-0.15.zip && mv vosk-model-small-en-us-0.15 model
```

If the model or mic is missing, the game logs it and falls back to text mode —
it never crashes.

### Spoken audio (text-to-speech)

```bash
python speak_test.py                        # quick check: do you hear four sentences?
python main.py --tts espeak --voice male    # Pi: play with spoken feedback
python main.py --tts pyttsx3                # Windows dev box: same, via the system voice
```

Pick the backend per machine — **there is no automatic cross-platform fallback
between them**:

| Backend | Where it works | Needs |
|---|---|---|
| `--tts espeak` | Raspberry Pi / Linux | `sudo apt install espeak-ng` |
| `--tts pyttsx3` | Windows (SAPI5), Linux (via espeak-ng) | `pip install pyttsx3` |
| `--tts piper` | Pi / Linux, natural-sounding | `piper` on PATH + `PIPER_MODEL=/path/voice.onnx` |
| `--tts print` | everywhere (default) | nothing — prints `[SPEAK] ...` lines |

If a backend can't start (binary missing, no audio device), the game **never
crashes** — it prints a one-line note and degrades to the print speaker. So on a
Windows box `--tts espeak` logs `espeak unavailable: espeak-ng binary not found
on PATH; falling back to print` and runs silently; use `--tts pyttsx3` there, or
install espeak-ng with `winget install --id eSpeak-NG.eSpeak-NG -e` (then add
`C:\Program Files\eSpeak NG` to PATH and reopen the terminal).

#### Narrator voice

`--voice female|male` selects an espeak-ng voice preset (**espeak backend only** —
it is ignored by `pyttsx3`/`piper`/`print`). Default is `female`, so running
without the flag sounds exactly as it did before.

| Preset | espeak voice | Rate | Pitch |
|---|---|---|---|
| `female` (default) | `en-us+f2` | 130 | 40 |
| `male` | `en-us+m3` | 130 | 40 |

The presets live in `VOICE_PRESETS` in [`chess_ai/speech.py`](chess_ai/speech.py);
add an entry there and it becomes a new `--voice` choice (also add it to the
`choices=` list in `main.py`). For finer control, `get_speaker` takes the same
knobs directly — explicit arguments beat the preset:

```python
from chess_ai import get_speaker
get_speaker("espeak", preset="male").say("knight to f6")
get_speaker("espeak", preset="male", rate=110, amplitude=200).say("check")
```

### Raspberry Pi

Full fresh-Pi runbook (apt packages, audio device discovery, test ladder):
**[docs/pi-setup.md](docs/pi-setup.md)**.

## Playing

You are White; speak when you see `[VOICE] listening...`. Say moves any natural
way: *"pawn to e4"*, *"e4"*, *"knight f3"*, *"kingside castle"*. The board
confirms every move out loud before executing it. Take your time — silence never
fails a turn.

Useful flags: `--skill 0-20` (opponent strength, default 5), `--think 0.5`
(seconds per AI move), `--turns 40` (demo safety limit), `--tts espeak` +
`--voice female|male` (spoken feedback and which narrator voice).

## Tests

```bash
python -m pytest tests/ -q
```

## Status

Voice input, chess AI, audio feedback, and motion planning (Role 3) are fully
working. The Pi-side serial sender is real too — `--serial <port>` streams the
planned G-code to the ESP32 over USB (with a tolerant `ok`/`<Idle>` handshake).
What's still pending on the hardware side (Role 4) is flashing FluidNC onto the
ESP32 and its pin/motor/homing config; until then the bytes go out on the wire
but nothing answers, which the link handles gracefully.
