# DeskBuddy

A desk companion on a FREE-WiLi. It announces your iPhone messages and missed calls, then reads them aloud only when you ask.

**Devpost:** [devpost.com/software/superdesk](https://devpost.com/software/superdesk) (MHacks 2026)

## How it works

```
iPhone --BLE (ANCS)--> Bottlenose Orca (ESP32-C6, ESPHome) --UART--> FREE-WiLi --USB--> desk_assistant.py (laptop)
```

1. The Bottlenose Orca runs [`desk_ancs.yaml`](desk_ancs.yaml), an ESPHome config using [esphome-ancs](https://github.com/wonderslug/esphome-ancs). It pairs with the iPhone over Bluetooth and subscribes to its notifications (Apple ANCS).
2. For each notification it sends a line such as `WILIMSG|com.apple.MobileSMS|Sender|message text` over UART at 115200 baud (GPIO16 TX / GPIO17 RX on the 20-pin header).
3. `desk_assistant.py` reads those lines from the FREE-WiLi. It can also read them from the Bottlenose's own USB port: `MESSAGE_SOURCE = "auto"` picks USB when it sees an Espressif device (VID `0x303A`) and the header otherwise.
4. When a message arrives, DeskBuddy beeps, turns the LEDs blue and shows the sender on the FREE-WiLi screen. Then it listens for about 8 seconds.
5. Speech recognition is offline: [Vosk](https://alphacephei.com/vosk/) runs on the FREE-WiLi microphone audio, upsampled from 8 kHz to 16 kHz.
6. Replies are spoken with **ElevenLabs** TTS. The audio is band-limited (250 Hz–3.4 kHz), downsampled to an 8 kHz WAV and played on the FREE-WiLi speaker. `pyttsx3` is the offline fallback.
7. Summaries come from **Google Gemini**, which falls back through `gemini-flash-latest`, `gemini-2.5-flash` and `gemini-flash-lite-latest`.

## Voice commands

| Say | What happens |
|---|---|
| "read my message" | Reads the newest message |
| "repeat" / "say that again" | Reads it again |
| "tell me my messages" | Reads unread messages, or the latest few if none are unread |
| "summarize my last three messages from Siddu" | Gemini summary of that sender's last N messages (default 5, history keeps 10) |

**Buttons:** GREEN = listen at any time. RED = dismiss.

**Calls:** Only missed calls are announced ("You have a missed call from X"). Incoming calls are ignored, and a missed-call notification only counts if that person's call came in within the last 2 minutes, so old ones the iPhone re-sends aren't repeated.

## LED colors

| Color | Meaning |
|---|---|
| Blue | New message |
| Green | Listening / reading |
| Orange | Missed call |
| Cyan | Thinking (Gemini) |
| Dim blue | Something is still unread |

## Setup

**1. Flash the Bottlenose Orca** (once)

```bash
pip install esphome
esphome run desk_ancs.yaml
```

`components/ancs_bt/` is a small shim that keeps the ANCS component working on ESPHome 2026.9 and later. After flashing, pair the iPhone with the `superdesk` device in Bluetooth settings and allow notifications.

**2. Python environment**

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install freewili vosk numpy pyttsx3 requests pyserial google-genai
```

Download [`vosk-model-small-en-us-0.15`](https://alphacephei.com/vosk/models) and unzip it into this folder.

**3. API keys**

```bash
cp .env.example .env
```

Fill in `ELEVENLABS_API_KEY`, `ELEVENLABS_VOICE_ID` (optional) and `GEMINI_API_KEY`. Without ElevenLabs it falls back to `pyttsx3`. Without Gemini, summaries are off.

**4. FREE-WiLi**

Set the FREE-WiLi UART to 115200 8N1 so it passes the Bottlenose's lines through. Then plug the FREE-WiLi into the laptop over USB. If you use the Bottlenose's USB port as the message source, plug that in too; both need data cables.

## Run

```bash
python desk_assistant.py                 # normal
python desk_assistant.py --test          # fake an incoming message, no phone needed
python desk_assistant.py --speaker-test  # check the FREE-WiLi speaker
```

## Other files

- `wili_voice_dj.py`: a side experiment. Hold GREEN and say "play <song>", and the laptop plays a matching MP3 with pygame (set `MUSIC_DIR` to your music folder first).
- `photon_bridge/` and `contacts.example.json`: an optional Node bridge (spectrum-ts, port 8787) for replying to iMessages by voice. It is **off by default** (`REPLIES_ENABLED = False`) and was not used in the demo.

## Tech stack

Python, FREE-WiLi, Bottlenose Orca (ESP32-C6), ESPHome, Bluetooth LE / Apple ANCS, Vosk, ElevenLabs, Google Gemini

Built at MHacks 2026.
