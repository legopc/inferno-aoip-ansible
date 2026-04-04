# Node Modes

Each Inferno node is provisioned in one of two modes, set by the `inferno_mode` variable in your inventory.

---

## spotify — Spotify Connect Transmitter

**Use when:** You want to stream music from Spotify to a Dante-connected speaker or amplifier.

### Signal flow

```
Spotify app (phone/desktop)
        │  Spotify Connect (Wi-Fi/LAN)
        ▼
   librespot (Spotify receiver daemon)
        │  PCM audio → hw:Loopback,0,0
        ▼
   snd-aloop (kernel loopback, card 5)
        │  PCM audio ← hw:Loopback,1,0
        ▼
   alsaloop → inferno_spotify (ALSA virtual device)
        │  RTP stream
        ▼
   Dante network → hardware speaker / amplifier
```

### Services enabled (spotify mode)

| Service | Description |
|---------|-------------|
| `statime-inferno.service` | PTP clock sync (system) |
| `avahi-daemon.service` | mDNS discovery for Spotify (system) |
| `librespot.service` | Spotify Connect receiver (user) |
| `librespot-watchdog.service` | Restarts librespot on audio key error (user) |
| `inferno-bridge.service` | alsaloop: loopback → inferno_spotify (user) |
| `inferno-keepalive.service` | Keeps Dante device registered when idle (user) |

### Hardware requirements

- Any NIC (hardware PTP via `/dev/ptp0` preferred but not required)
- No physical audio hardware needed if routing to Dante-only devices
- If you want local analog output too, use an HDMI or USB audio device and route via Dante Controller

---

## aux — Analog I/O Bridge

**Use when:** You have a physical 3.5mm line-in source you want to put on the Dante network, or you want to drive a 3.5mm line-out from a Dante source.

### Signal flow

**TX (analog in → Dante):**
```
3.5mm line-in jack
        │  analog signal
        ▼
   ALSA hardware capture (hw:<card>,0)
        │  PCM via inferno_aux_tx
        ▼
   Dante network TX stream
```

**RX (Dante → analog out):**
```
Dante network RX stream
        │  PCM via inferno_aux_rx
        ▼
   alsaloop → hw:<card>,0 (hardware playback)
        │  analog signal
        ▼
   3.5mm line-out jack
```

### Services enabled (aux mode)

| Service | Description |
|---------|-------------|
| `statime-inferno.service` | PTP clock sync (system) |
| `avahi-daemon.service` | mDNS discovery (system) |
| `inferno-aux-tx.service` | Capture → Dante TX (user) |
| `inferno-aux-rx.service` | Dante RX → playback (user) |

> ⚠️ **Do not enable `inferno-aux-keepalive` in aux mode.** The keepalive opens the
> `inferno_aux_rx` device simultaneously with `inferno-aux-rx`, creating two competing
> Dante subscription owners. Routing changes from Dante Controller reach only one opener,
> causing streams to stop. The `system-state` role enforces this by explicitly stopping
> and disabling keepalive in aux mode.

### Hardware requirements

- Physical audio card with analog I/O (3.5mm jacks)
- Codec-specific ALSA mixer configuration (`inferno_mixer_commands` in inventory)
- Run `amixer -c <card> contents` to find the correct numids for your codec

### Mixer configuration

The `alsa` role runs the `inferno_mixer_commands` list against the hardware card. These commands are codec-specific — the numids differ between Realtek ALC221, CX20632, and other codecs. Always verify with `amixer -c <card> contents` on the actual hardware before writing the inventory.

---

## Bluetooth bridge (bt-bridge)

**Use when:** You want to receive audio from a Bluetooth device (phone, tablet) and put it on the Dante network.

This is not a separate `inferno_mode` value — provision the node in `spotify` mode first, then deploy the `inferno-bt-loop` template as an additional user service. The bt-bridge is an overlay on a standard spotify node.

### Signal flow

```
Bluetooth device (phone)
        │  A2DP
        ▼
   bluealsa / bluealsa-aplay
        │  PCM → hw:Loopback,0,0
        ▼
   snd-aloop (card 5)
        │  PCM ← hw:Loopback,1,0
        ▼
   alsaloop → inferno_spotify
        │  RTP
        ▼
   Dante network
```

### Important: phone pause / TX ring buffer lag

When the Bluetooth source pauses (phone call, pause button), `bluealsa-aplay` stops writing to the loopback. `alsaloop` in the bridge continues reading from an idle loopback, accumulating a TX ring buffer lag. New Dante subscribers then see "flow creation in progress" looping.

**Fix:** Use `PartOf=inferno-bt-bridge.service` in `inferno-bt-loop.service` so the loop restarts whenever the Bluetooth bridge restarts (which happens on A2DP connect/disconnect). Also set `RuntimeMaxSec=6h` to auto-restart periodically and prevent lag accumulation from extended uptime.
