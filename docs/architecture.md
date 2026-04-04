# Architecture — Inferno AoIP on Linux

## What is Inferno AoIP?

[Inferno](https://gitlab.com/lumifaza/inferno) is an open-source ALSA PCM plugin that makes Dante/AES67 network audio devices appear as standard ALSA sound cards on Linux. Once installed, any application that uses ALSA (librespot, alsaloop, arecord/aplay) can read from or write to Dante devices on the network — no proprietary drivers, no Windows requirement.

---

## The stack

```
┌─────────────────────────────────────────────────────┐
│                  Linux node (Arch)                   │
│                                                      │
│  ┌─────────────┐   ALSA PCM    ┌──────────────────┐ │
│  │  librespot  │──────────────▶│ inferno_spotify   │ │
│  │ (Spotify)   │               │ (ALSA virtual dev)│ │
│  └─────────────┘               └──────────┬─────────┘ │
│                                           │           │
│  ┌─────────────┐   ALSA PCM    ┌──────────▼─────────┐ │
│  │  alsaloop   │◀──────────────│  hw:Loopback,1,0   │ │
│  │  (bridge)   │               │  (snd-aloop card5) │ │
│  └──────┬──────┘               └────────────────────┘ │
│         │ ALSA PCM                                    │
│  ┌──────▼──────┐                                      │
│  │   Inferno   │◀────────────── PTP clock sync        │
│  │  ALSA plugin│                (statime)             │
│  └──────┬──────┘                                      │
└─────────┼───────────────────────────────────────────-─┘
          │ Dante / AES67 (UDP/RTP over Ethernet)
          ▼
    Dante network (switches, hardware devices, Dante Controller)
```

---

## Components

### Inferno ALSA PCM plugin
A shared library (`libasound_module_pcm_inferno.so`) that ALSA loads as a PCM type. It implements the Dante AoIP protocol stack directly, so any application opening the `inferno_spotify` or `inferno_aux_*` virtual device is actually streaming to/from the Dante network.

The plugin is built from source (`cargo build --release`) because it is not packaged for Arch. The `inferno` role handles cloning, building, and installing it.

### statime (Inferno fork)
A Rust implementation of IEEE 1588 PTPv1 that the Inferno plugin requires for sample-accurate clock synchronisation. The Inferno fork (`inferno-dev` branch) adds the Unix domain socket interface that the plugin uses to read the PTP clock.

The `statime` role builds and installs it as a system service (`statime-inferno.service`).

### snd-aloop (ALSA loopback)
The `snd_aloop` kernel module creates a virtual loopback sound card. In **spotify mode**, librespot writes decoded audio to loopback device 0 and `alsaloop` reads it from loopback device 1 and writes it to the Inferno plugin — bridging Spotify playback onto the Dante network.

The loopback is pinned to card index 5 via `/etc/modprobe.d/snd-aloop.conf` to avoid conflicts if other audio hardware is present.

### librespot
An open-source Spotify Connect implementation. In spotify mode, it decodes audio and writes PCM to `hw:Loopback,0,0`. The Inferno bridge (`inferno-bridge.service`) reads from `hw:Loopback,1,0` and forwards to the Dante network.

### avahi
Provides mDNS/DNS-SD so Spotify clients on the local network can discover the Inferno node as a Spotify Connect device.

---

## Dante and PTP

### Dante protocol
Inferno speaks Dante, which is a proprietary layer on top of AES67/AES70. Dante devices discover each other via mDNS/Bonjour (`_netaudio-arc._udp` service). Audio flows as RTP streams. Routing (which TX feeds which RX) is controlled via Dante Controller (Windows/macOS application).

### PTP clock synchronisation
Dante uses PTPv1 (IEEE 1588-2002) to keep all devices on the same clock. One device acts as grandmaster; all others slave to it. statime disciplines the Linux system clock to track the Dante grandmaster so Inferno streams are in sync.

**Hardware PTP** (`/dev/ptp0`) gives sub-microsecond accuracy and is preferred when the NIC supports it (Intel I219-LM, I210 etc.). Software PTP (via `SO_TIMESTAMPING`) works but has higher jitter.

### Why PTP matters for reliability
If the PTP clock offset drifts too far, Inferno's TX plugin detects it and destroys the flow before sending RTP. Subscribers see "flow creation in progress" looping. Always ensure statime is active and tracking the grandmaster before troubleshooting audio issues.

---

## Network requirements

- All Inferno nodes and Dante hardware on the **same L2 network segment** (or L3 with multicast routing)
- UDP/RTP ports open between devices (Dante uses dynamic ports in the 14336–14591 range plus discovery on port 8700)
- PTP multicast traffic not blocked (`224.0.1.129`)
- mDNS not blocked between nodes (`224.0.0.251`, UDP port 5353)
