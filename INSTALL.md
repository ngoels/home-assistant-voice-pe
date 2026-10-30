# Install guide — Home Assistant Voice PE + OpenAI Realtime 2

A complete, from-zero walkthrough for a newcomer.

> This is the **ngoels fork** of Xander's
> [xandervanerven/home-assistant-voice-pe](https://github.com/xandervanerven/home-assistant-voice-pe)
> and [xandervanerven/ha-openai-realtime](https://github.com/xandervanerven/ha-openai-realtime).
> Both halves below come from the ngoels forks.

You'll set up two halves:

1. the **backend app** (the voice "brain" that runs the OpenAI Realtime session), and
2. the **device firmware** (turns the Voice PE into a thin client that listens and speaks).

Plan ~30–45 minutes the first time. After that, updates are one click.

```
Home Assistant Voice PE          Home Assistant (your box)             Cloud
┌──────────────────────────┐   plain WS   ┌────────────────────────┐  WS  ┌─────────────┐
│ custom firmware           │ ───────────▶│ OpenAI Realtime 2      │ ───▶ │ OpenAI       │
│  (va_client thin client)  │ 16k mic up  │  Voice Agent app       │ 24k  │ gpt-realtime │
│  wake word + XMOS DSP      │ ◀───────────│  (Python / pipecat)    │ ◀─── │  -2          │
└──────────────────────────┘ 24k spkr dn  │          │ tools         │     └─────────────┘
                                           ▼          ▼
                                  HA MCP Server (/api/mcp) → controls your home
```

Once it's set up you can **control your smart home by voice** (lights, switches,
scenes, climate), hold a **natural back-and-forth** conversation, and **look things
up online** (weather, news, facts) — web search is on by default.

## What you need

- A **Home Assistant Voice PE** device (this firmware is **only** for that hardware).
- **Home Assistant OS** (so you can install apps).
- An **OpenAI account** with **billing enabled** (the voice runs on OpenAI's paid API).
- A few minutes at the keyboard, and the device on the **same network** as Home Assistant.

---

## Part 1 — The backend app (the brain)

### 1.1 Add the repository & install

1. In Home Assistant: **Settings → Apps → App store**.
2. Top-right **⋮ → Repositories** → paste and add:
   `https://github.com/ngoels/ha-openai-realtime`
3. Find **OpenAI Realtime 2 Voice Agent** in the store and click **Install**.
   Home Assistant builds it locally — this takes a few minutes the first time.

### 1.2 Add your OpenAI API key

1. Go to <https://platform.openai.com/> → **API keys** → **Create new secret key**,
   and make sure **billing** is enabled on the account.
2. Open the app's **Configuration** tab and paste the key into **`openai_api_key`**.

> New OpenAI accounts start on a low rate-limit tier. If you later see *"Rate limit
> reached"* in the log, raise your usage tier on the OpenAI dashboard.

### 1.3 Let it control your home (Home Assistant MCP)

The assistant controls your home through Home Assistant's official, built-in
**[MCP Server](https://www.home-assistant.io/integrations/mcp_server/)** integration —
that's what lets the voice turn your lights, switches, scenes and climate on and off.

1. Add it: **Settings → Devices & Services → Add Integration**, search **"Model Context
   Protocol Server"**, and add it
   ([one-click add](https://my.home-assistant.io/redirect/config_flow_start/?domain=mcp_server)).
2. **Settings → Voice assistants → Expose** → tick the lights, switches, scenes and
   climate you want to control by voice. **Only exposed entities are controllable** —
   this is your safety boundary. Expose your **scripts** too if the assistant should
   run them; leave **automations un-exposed** (see 1.6).
3. In the app's Configuration, leave **`ha_mcp_url`** and **`longlived_token`**
   **blank**. The app then uses Home Assistant's built-in MCP endpoint with its own
   token. (Only fill `longlived_token` if the startup log shows a 401/403 on
   `/core/api/mcp`.)

### 1.4 Recommended starting settings

Set these on the **Configuration** tab (each option has inline help text). This is a
known-good starting point — you can fine-tune later:

| Option | Value |
|---|---|
| `openai_model` | `gpt-realtime-2` |
| `openai_voice` | `marin` (or `cedar`) |
| `openai_speed` | `1.0` |
| `max_output_tokens` | `0` (unlimited) |
| `noise_reduction` | `off` |
| `vad_eagerness` | `low` |
| `transcription_language` | blank (set your ISO code, e.g. `nl`, to log what it heard) |
| `transcription_model` | `gpt-4o-transcribe` |
| `follow_up_listen_seconds` | `8` |
| `follow_up_open_delay_ms` | `700` |
| `wake_open_delay_ms` | `700` |
| `playback_prebuffer_ms` | `150` |
| `max_context_messages` | `12` |
| `mcp_tool_allowlist` | **blank** (use all of the official server's tools — it's already a small set) |
| `enable_web_search` | `true` (online lookups on by default; set `false` to disable) |
| `web_search_model` | `gpt-5.5` (best quality; mini/nano are cheaper) |
| `web_search_city` | your town, e.g. `Trumau` (country and time zone come from Home Assistant) |
| `enable_voice_commands` | `true` (your sentence automations work by voice — see 1.6) |
| `instructions` | the English default (to change language, see Part 5) |

### 1.5 Start it

Click **Start**, then open the **Log** tab. A healthy start shows
`✅ Fetched N MCP tools` and `Creating session with N tools` (with `Hass*` names).
With voice commands on, it also shows `🗣️ Voice commands: N automation(s) …`.
The app now listens on port **8080**.

### 1.6 Your own voice commands (automations)

Automations with a **sentence trigger** ("When a sentence is said") work by voice,
as with the stock Voice PE firmware: the assistant picks the matching sentence and
Home Assistant runs the automation. They do **not** need to be exposed — keep
automations **un-exposed**, so the assistant can't switch them on or off. To have
the assistant say a result, end the automation with `set_conversation_response`.

---

## Part 2 — The device firmware

This replaces the stock Home Assistant voice pipeline on the Voice PE with a thin
client that streams audio to the app. You set it up **once** via a tiny "stub"
config; after that, firmware updates are one click (no copy-pasting).

### 2.1 Install the ESPHome Builder app

You build and flash the firmware with the **ESPHome Device Builder** app — the
official [ESPHome](https://esphome.io/) tool that runs inside Home Assistant. You need
it to adopt the device and flash this firmware onto it.

1. Open **Settings → Apps → App store**, search **ESPHome Device Builder**, and
   click **Install**
   ([one-click open](https://my.home-assistant.io/redirect/supervisor_addon/?addon=5c53de3b_esphome)).
2. Enable **Show in sidebar**, then **Start** → **Open Web UI**.

### 2.2 Adopt the Voice PE

1. The Voice PE (on its stock firmware) should appear in ESPHome Builder as a
   **discovered device**. If it doesn't, add it by its `home-assistant-voice-xxxx.local`
   address.
2. Click **Adopt**. ESPHome creates a device entry. **Don't install the stock config
   yet.**
3. ESPHome generates an **API encryption key** for the device — note it; you'll
   put it in `secrets.yaml` next. (It also secures wireless updates; any OTA
   password ESPHome shows is not needed.)

### 2.3 Add your secrets

In ESPHome Builder → **Secrets** (top-right ⋮), add:

```yaml
wifi_ssid: "Your-WiFi"
wifi_password: "your-wifi-password"
api_key: "the-API-encryption-key-from-step-2.2"   # 44-char base64

# Optional — ONLY if you want a fixed IP (otherwise the device uses DHCP):
# static_ip: "192.168.1.50"
# gateway:   "192.168.1.1"
# subnet:    "255.255.255.0"
# dns1:      "1.1.1.1"
# dns2:      "1.0.0.1"
```

> The firmware itself is pulled from the **public** repo at build time — no token needed.

### 2.4 Paste the device stub & flash

1. In ESPHome Builder, **Edit** the adopted device and **replace its entire YAML** with
   a ready-made stub from the firmware repo:
   - DHCP: [`esphome-builder.dhcp.yaml`](https://github.com/ngoels/home-assistant-voice-pe/blob/main/esphome-builder.dhcp.yaml)
   - Fixed IP: [`esphome-builder.static-ip.yaml`](https://github.com/ngoels/home-assistant-voice-pe/blob/main/esphome-builder.static-ip.yaml)

   Set `name` and `friendly_name`, and **keep** the `packages:` / `dashboard_import:`
   lines — those are what pull the full firmware from the repo. Save.

   > **Required:** add a `va_url:` line under `substitutions:` with your Home
   > Assistant's address, e.g. `va_url: "ws://192.168.1.10:8080/"`. This fork's
   > default is the maintainer's own LAN IP. A fixed IP (DHCP reservation) for
   > Home Assistant is the most robust choice.

2. Click **Install →**
   - **First time:** choose **Plug into this computer** — the first flash from stock
     firmware needs the device connected by **USB** to the machine running your browser.
   - **After that:** **Wirelessly (OTA)** — every later flash goes over Wi-Fi.

That's it — the device boots, connects to the app, and you're ready to talk to it.

---

## Part 3 — First run & verify

1. After boot, the LED ring should settle to **idle (blue)** — that means the device
   reached the app's WebSocket.
2. Say **"Okay Nabu"** → a wake chime plays and the ring turns to **listening**.
   (Tip: wait for the ring to start moving before you speak — or switch the
   device's **Wake sound** off, then the microphone opens right away.)
3. Ask for something you exposed, e.g. *"turn on the bedroom lamp"* → the ring shows
   **thinking** → it acts and replies.
4. To interrupt a reply: say **"stop"** or press the **center button**.

**If something's off, check the logs:**
- App **Log** tab: `🗣️ user:` / `🤖 assistant:` lines, tool calls, and
  `🔌 reconnecting` / `✅ reconnected`.
- Device logs: ESPHome Builder → your device → **Logs**.

---

## Part 4 — Updating later (one click)

- **Firmware:** when a new version is released, ESPHome Builder shows **"Update
  available"** for your device. Click it → it recompiles with the latest config + code
  and flashes over Wi-Fi. No copy-pasting, ever again.
- **App:** Home Assistant shows an **Update** badge on the app (with a changelog).
  Click **Update** — it rebuilds and restarts.

Your device-specific settings (name, Wi-Fi, IP) live in your stub + `secrets.yaml` and
are **never** overwritten by an update.

---

## Part 5 — Tuning tips

- **It hears its own tail** (a stray turn right after a reply, or it repeats
  its last answer): the default `follow_up_open_delay_ms` of `700` normally
  prevents this — raise it further (~1000) if it still happens.
- **Crackle at the start of a reply:** raise `playback_prebuffer_ms` above the
  default 150 (e.g. ~250).
- **See what it understood:** set `transcription_language` to your ISO code (e.g. `nl`)
  → the app log gains `🗣️ user: …` lines.
- **Web search:** it answers weather / news / facts online out of the box
  (`enable_web_search` is on by default, model `gpt-5.5`; a few cents per search —
  set it `false` or pick a mini model to cut cost). Set `web_search_city` so
  questions like "weather tomorrow" use your town. See the app's DOCS.md.
- **Announcements from automations:** call the device action
  `esphome.<device name>_announce` with a `message` — it is spoken in your
  OpenAI voice (waits until a running conversation has finished).
- **The system prompt (`instructions`)** sets the assistant's language, style and
  behavior. The shipped default (English) is voice-tuned — short spoken replies, silent
  tool calls, varied confirmations, and strict language pinning:

  ```
  You are the cheerful, friendly voice assistant of Home Assistant and you control the
  smart home. LANGUAGE: Speak and understand only English, with a natural, neutral
  accent. Never switch language, not even for stray foreign words or an accent; do not
  infer the language from accent or a single word. STYLE: Your replies are read aloud,
  so keep them short and natural. Do not read out entity IDs, lists, or technical names;
  use ordinary names ("the bedroom lamp"). Call your tools silently — say nothing like
  "Okay, let me check" or "one moment" beforehand; just do it and then give one complete
  answer. Do not start every answer with "Okay"; vary your confirmation naturally, e.g.
  "The bedroom lamp is on now" or "Done, the light is off". BEHAVIOR: Carry out requested
  actions immediately with your tools and never guess. If you are unsure which device,
  room, or action is meant, ask a short clarifying question first. If something fails,
  briefly say what went wrong.
  ```

  To use another language, swap the wording but keep the same LANGUAGE/STYLE/BEHAVIOR
  structure. Example (Dutch):

  ```
  Je bent de vrolijke, vriendelijke spraakassistent van Home Assistant en bestuurt
  het smart home in huis. TAAL: Spreek en versta uitsluitend Nederlands, met een
  natuurlijke, neutrale tongval. Wissel nooit van taal — ook niet bij losse Engelse
  woorden ("living", "dimmer") of een accent. STIJL: Je antwoorden worden hardop
  voorgelezen, dus houd ze kort en spreektaal-natuurlijk. Lees geen entity-id's of
  technische namen voor; gebruik gewone namen ("de slaapkamerlamp"). Roep je
  gereedschappen stil aan: zeg vooraf NIETS als "Oké, ik kijk even" — voer de actie
  direct uit en geef daarna één compleet antwoord. Varieer je bevestiging natuurlijk,
  bijvoorbeeld "De slaapkamerlamp staat nu aan" of "Klaar, het licht is uit". GEDRAG:
  Voer gevraagde acties meteen uit en gok nooit; weet je iets niet zeker, vraag dan
  eerst kort om verduidelijking.
  ```

---

## Credits

- **[Xander (@xandervanerven)](https://github.com/xandervanerven)** — the OpenAI Realtime 2 firmware and app this fork builds on.
- Backend originally **[fjfricke/ha-openai-realtime](https://github.com/fjfricke/ha-openai-realtime)** (Felix Fricke).
- Firmware thin-client design based on **[maxmaxme/home-assistant-voice-pe](https://github.com/maxmaxme/home-assistant-voice-pe)**, a fork of **[esphome/home-assistant-voice-pe](https://github.com/esphome/home-assistant-voice-pe)** (Nabu Casa / ESPHome).
- Inspiration from **[marcinnowak79/home-assistant-voice-pe](https://github.com/marcinnowak79/home-assistant-voice-pe)** (gemini-live-proxy).
- Built on **[pipecat](https://github.com/pipecat-ai/pipecat)**, the **OpenAI Realtime API**, and the official **[Home Assistant MCP Server](https://www.home-assistant.io/integrations/mcp_server/)** integration.
