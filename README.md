# Home Assistant Voice PE — OpenAI Realtime 2 fork

> [!IMPORTANT]
> **This is 1 of 2 repos — you need both halves.** This repo is the **device
> firmware**; on its own it does nothing. It streams audio to a backend add-on that
> runs the OpenAI Realtime session and controls Home Assistant. You must set up both:
> - 🔌 **Device firmware** (this repo) — flashed onto the Voice PE
> - 🧠 **Backend add-on** → **[xandervanerven/ha-openai-realtime](https://github.com/xandervanerven/ha-openai-realtime)** (runs in Home Assistant)
>
> 📖 New here? The full **[INSTALL guide](INSTALL.md)** sets up both halves, step by step.

> **Personal fork** of
> [`xandervanerven/home-assistant-voice-pe`](https://github.com/xandervanerven/home-assistant-voice-pe),
> which builds on [`maxmaxme/home-assistant-voice-pe`](https://github.com/maxmaxme/home-assistant-voice-pe)
> and the official [`esphome/home-assistant-voice-pe`](https://github.com/esphome/home-assistant-voice-pe)
> firmware — see [Credits](#credits). The Voice PE runs as a **thin client**: it
> streams microphone audio over a plain WebSocket to a backend add-on, which runs
> an **OpenAI Realtime API** session (`gpt-realtime-2`) for speech-to-speech and
> controls Home Assistant through the official
> **[Home Assistant MCP Server](https://www.home-assistant.io/integrations/mcp_server/)**
> integration. There is no Home Assistant `voice_assistant` pipeline on the audio
> path — STT, TTS and the LLM all live in the Realtime session inside the add-on.
>
> The firmware config is [`home-assistant-voice.realtime.yaml`](home-assistant-voice.realtime.yaml).
> You don't paste it directly — you adopt it via a tiny per-device stub
> ([`esphome-builder.dhcp.yaml`](esphome-builder.dhcp.yaml)) that pulls it as a remote package,
> so updates are **one click** in the ESPHome dashboard (see Setup).

## What this fork changes vs. upstream

- **Custom `va_client` component** replaces the stock `voice_assistant`
  component: a thin WebSocket client (mic up / speaker down + an idle → listening
  → thinking → replying phase/LED state machine). All the voice intelligence
  runs in the backend add-on, not on the device.
- **One-click updates**: instead of pasting the whole config, you adopt a tiny
  per-device stub that pulls the firmware from this repo as a remote ESPHome
  `packages:` include. When a new version ships, the ESPHome dashboard shows
  "Update available" — one click recompiles with the latest. No local checkout,
  no tokens (this repo is public).
- **"stop" word + button interrupt**: say *"stop"* while the assistant is
  talking, or press the center button, to cancel the reply. This is the reliable
  way to interrupt.
- **Conversational, with online answers**: because the brain is `gpt-realtime-2`
  in the backend, you get a natural back-and-forth — and with web search enabled it
  can look things up online (weather, news, facts), not just control your devices.

## Changes in this fork (ngoels)

On top of the upstream fork above, this repo adds reliability tweaks for a home
where Home Assistant and the network reboot every night:

- **No unprompted "cloud not available" announcement.** Upstream plays the stock
  "Home Assistant Cloud" error sound whenever the add-on WebSocket can't be
  reached — e.g. during a nightly reboot. Here that stays **silent while the
  speaker is idle** (the red LED still shows the lost connection). You still hear
  the sound when you say the wake word while disconnected, or when the backend
  reports an error mid-conversation.
- **No freeze on a stalled network.** Control messages to the add-on (start,
  wake, stop, flush) give up after **1 s** instead of waiting forever, so a dead
  TCP link can no longer hang the device until the watchdog reboots it. Replies
  are asynchronous, so long backend work such as web search is unaffected.
- **C++ built from this repo.** The `va_client` component is pulled from this
  fork, not from upstream — upstream C++ fixes have to be merged in manually.
- **Leaner build.** Logging defaults to INFO (set `logger: level: DEBUG` in your
  device YAML to troubleshoot); the always-discarded mic pre-roll buffer, the
  unused stock Nabu Casa configs, their web installer and CI workflows are removed.
- **`va_url` points at a fixed LAN IP** (`ws://192.168.68.51:8080/`) — override it
  for your own setup (see Setup, step 3).

## Setup (ESPHome Builder)

1. Install and configure the **OpenAI Realtime 2 Voice Agent** add-on from
   [xandervanerven/ha-openai-realtime](https://github.com/xandervanerven/ha-openai-realtime)
   (sets your OpenAI API key, the model, and the Home Assistant MCP connection).
2. In **Builder → Secrets**, add the keys from
   [`secrets.yaml.example`](secrets.yaml.example): `wifi_ssid`, `wifi_password`,
   `ota_password`, `api_key` (plus `static_ip`/`gateway`/`subnet`/`dns1`/`dns2`
   only if you want a fixed IP).
3. Create a new device in the dashboard and replace its YAML with a ready-made
   stub — [`esphome-builder.dhcp.yaml`](esphome-builder.dhcp.yaml) for DHCP, or
   [`esphome-builder.static-ip.yaml`](esphome-builder.static-ip.yaml) for a fixed IP.
   Set `name`/`friendly_name` and keep the `packages:`/`dashboard_import:` lines.
   Override the `va_url` substitution with your add-on's address, e.g.
   `ws://homeassistant.local:8080/` — this fork's default is the maintainer's
   own LAN IP. A fixed IP (DHCP reservation) is the most robust choice.
4. **Install** once (USB, then wireless thereafter). The device adopts the
   firmware and connects to the add-on.

After that, when a new firmware version is released the dashboard shows
**"Update available"** for the device — click it to pull the latest config and
re-flash. No more copy-pasting.

## Known limitations

- **No voice timers or alarms yet** — every other Assist action (lights, switches,
  scenes, climate) and online questions work.
- **A brief reconnect about once an hour** (OpenAI's 60-minute session cap; the
  add-on refreshes proactively during a quiet moment, so it rarely interrupts).
- **Rarely, the assistant may stop itself** on a word in its own reply that sounds
  like "stop" — just ask again.
- **Using "stop":** interrupts the assistant *while it's speaking* (during a reply
  or the short listening window right after one); it has no effect before it has
  started answering.

## Credits

This fork stands entirely on the work of the original authors — thank you:

- **[Xander (@xandervanerven)](https://github.com/xandervanerven)** —
  the OpenAI Realtime 2 firmware fork this repo is based on
  ([home-assistant-voice-pe](https://github.com/xandervanerven/home-assistant-voice-pe)),
  the one-click update flow, the "stop"/audio reliability work, and the backend
  add-on [ha-openai-realtime](https://github.com/xandervanerven/ha-openai-realtime).
- **[Maxim Lepekha (@maxmaxme)](https://github.com/maxmaxme)** — the thin-client
  design and the original `va_client` component
  ([maxmaxme/home-assistant-voice-pe](https://github.com/maxmaxme/home-assistant-voice-pe)).
- **[@marcinnowak79](https://github.com/marcinnowak79)** — inspiration from the
  gemini-live-proxy approach
  ([marcinnowak79/home-assistant-voice-pe](https://github.com/marcinnowak79/home-assistant-voice-pe)).
- **[ESPHome](https://esphome.io) / [Nabu Casa](https://www.nabucasa.com)** and
  all contributors to
  [esphome/home-assistant-voice-pe](https://github.com/esphome/home-assistant-voice-pe)
  — the original firmware for the
  [Home Assistant Voice: Preview Edition](https://www.home-assistant.io/voice-pe/),
  including the sounds this firmware still uses.

See [the upstream documentation](https://voice-pe.home-assistant.io/) for hardware
setup and troubleshooting. Licensed under the [ESPHome License](LICENSE)
(MIT + GPLv3), same as upstream.
