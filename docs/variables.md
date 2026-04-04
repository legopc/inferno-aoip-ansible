# Variable Reference

All variables that can be set in your inventory file.

---

## Connection variables

These are standard Ansible connection variables. Set them per-host in your inventory.

| Variable | Required | Example | Description |
|----------|----------|---------|-------------|
| `ansible_host` | ✅ | `192.0.2.10` | IP address of the target node |
| `ansible_user` | ✅ | `your_username` | SSH user (typically same as `inferno_user`) |
| `ansible_ssh_private_key_file` | ✅ | `~/.ssh/your_key` | Path to SSH private key |
| `ansible_become_password` | if sudo needs pw | `"CHANGE_ME"` | sudo password. Use Ansible Vault: `!vault \|` ... |

---

## Inferno node variables

### `inferno_user`
**Required.** The non-root user on the target node that will own all Inferno services and files.

- Type: string
- Example: `inferno_user: audiouser`
- Must already exist on the node (created during Arch install)
- All user-scope systemd services run under this user
- The user is added to the `audio` group by the `base` role

---

### `inferno_interface`
**Required.** The network interface used for Dante/PTP traffic.

- Type: string
- Example: `inferno_interface: enp1s0`
- Find with: `ip link show` on the node
- The MAC address of this interface is used to derive `inferno_device_id`
- Must be the wired Ethernet NIC on the Dante network

---

### `inferno_name`
**Required.** The Spotify Connect device name shown in the Spotify app.

- Type: string
- Example: `inferno_name: "Living Room"`
- Only used in `spotify` mode
- Dante TX stream name is derived from the MAC address, not this value

---

### `inferno_mode`
**Optional.** Operating mode of the node.

- Type: string
- Default: `spotify`
- Allowed values: `spotify`, `aux`
- Controls which roles are applied and which services are enabled/disabled
- The `system-state` role enforces the correct service set for the chosen mode

---

### `inferno_audio_card`
**Required for aux mode.** ALSA card index of the physical audio hardware.

- Type: integer
- Example: `inferno_audio_card: 1`
- Find with: `aplay -l` on the node (look for your codec, not Loopback)
- `snd-aloop` is pinned to card 5 by the `base` role — hardware cards land at 0, 1, etc.
- Used in `asoundrc-aux.j2` to build the correct device paths

---

### `inferno_mixer_commands`
**Required for aux mode.** List of `amixer` commands to configure the hardware mixer.

- Type: list of strings
- Example:
  ```yaml
  inferno_mixer_commands:
    - "amixer -c 1 cset numid=15 on"
    - "amixer -c 1 cset numid=14 87"
  ```
- These commands are codec-specific. The numids differ between codecs.
- Run `amixer -c <card> contents` on the node to find the correct numids
- Settings are saved to `/var/lib/alsa/asound.state` via `alsactl store`
- The commands should: unmute playback, set playback volume, enable capture, set capture volume

---

### `inferno_device_id`
**Optional.** The 16-character Dante device ID (hex string).

- Type: string
- Example: `inferno_device_id: "aabbccddeeff0000"`
- Default: **auto-derived** from the MAC address of `inferno_interface` by the site.yml pre_tasks
  - Derivation: strip colons from MAC, append `0000`
  - MAC `aa:bb:cc:dd:ee:ff` → `aabbccddeeff0000`
- Override here only if the auto-derivation is not suitable (e.g. multiple NICs, MAC changes)
- This ID appears in Dante Controller and must be unique across all nodes on the network

---

### `dante_device_ip`
**Optional.** IP address of a hardware Dante device on the same network.

- Type: string
- Example: `dante_device_ip: 192.0.2.20`
- Used in ALSA configuration to locate the Dante hardware interface
- Only needed if your ALSA templates reference a specific hardware Dante device by IP

---

## Variable summary table

| Variable | Required | Mode | Default |
|----------|----------|------|---------|
| `ansible_host` | ✅ | all | — |
| `ansible_user` | ✅ | all | — |
| `ansible_ssh_private_key_file` | ✅ | all | — |
| `ansible_become_password` | if needed | all | — |
| `inferno_user` | ✅ | all | — |
| `inferno_interface` | ✅ | all | — |
| `inferno_name` | ✅ | spotify | — |
| `inferno_mode` | ❌ | all | `spotify` |
| `inferno_audio_card` | ✅ aux | aux | — |
| `inferno_mixer_commands` | ✅ aux | aux | — |
| `inferno_device_id` | ❌ | all | auto from MAC |
| `dante_device_ip` | ❌ | all | — |
