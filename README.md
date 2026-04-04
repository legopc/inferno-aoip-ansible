# Inferno AoIP — Ansible Provisioning

Provision Arch Linux machines as [Inferno AoIP](https://gitlab.com/lumifaza/inferno) nodes using Ansible. Inferno exposes network audio (Dante/AES67) devices to any ALSA-capable Linux application — no proprietary drivers, no Windows VM.

---

## What this does

Running this playbook against a fresh Arch Linux install will:

- Install all required packages (ALSA stack, avahi, build tools)
- Build and install the Inferno ALSA PCM plugin (from source)
- Build and install the statime PTP daemon (Inferno fork)
- Configure `snd-aloop` as a pinned loopback card (card 5)
- Set up systemd user services for the chosen node mode
- Apply ALSA mixer settings for hardware I/O (aux mode)
- Enable lingering so services survive without a login session

### Node modes

| Mode | What it does |
|------|-------------|
| `spotify` | Spotify Connect receiver → Dante transmitter. Play music from the Spotify app and it appears as a Dante source. |
| `aux` | Analog line-in → Dante TX and Dante RX → analog line-out. Bridges a 3.5mm source/sink to the Dante network. |

---

## Prerequisites

**Control machine** (where you run Ansible):
- Python 3 + pip
- `ansible` and `ansible-galaxy`

```bash
pip install ansible
ansible-galaxy install -r requirements.yml
```

**Target nodes** (Arch Linux):
- Fresh Arch Linux install with network access
- A non-root user with sudo privileges (this is your `inferno_user`)
- SSH access from the control machine

---

## Quick start

```bash
# 1. Clone this repo
git clone https://github.com/YOUR_USERNAME/inferno-aoip-ansible
cd inferno-aoip-ansible

# 2. Install Ansible collection dependency
ansible-galaxy collection install -r requirements.yml

# 3. Create your inventory (copy the example, fill in your values)
cp inventory.example.yml inventory-mynode.yml
$EDITOR inventory-mynode.yml

# 4. Run the playbook
ansible-playbook -i inventory-mynode.yml site.yml
```

The playbook is idempotent — safe to re-run. Re-running after a `git pull` on the Inferno or statime repos will rebuild only if new commits are present.

---

## Inventory variables

See [`inventory.example.yml`](inventory.example.yml) for a fully annotated example and [`docs/variables.md`](docs/variables.md) for a complete variable reference.

---

## Project structure

```
.
├── site.yml                 # Main playbook
├── inventory.example.yml    # Annotated example — copy and fill in your values
├── ansible.cfg              # Defaults (remote_user, inventory)
├── requirements.yml         # Ansible Galaxy dependencies
├── roles/
│   ├── base/                # Packages, snd-aloop, yay AUR helper, audio group
│   ├── rust/                # Rust toolchain (rustup) — required by inferno + statime
│   ├── statime/             # PTP clock daemon (statime inferno fork)
│   ├── inferno/             # Inferno ALSA PCM plugin (build from source)
│   ├── alsa/                # ALSA system config + per-user .asoundrc
│   ├── librespot/           # Spotify Connect daemon
│   ├── aux/                 # AUX mode services (analog I/O bridge)
│   └── system-state/        # Mode enforcement (enables correct services, disables wrong ones)
├── templates/               # Jinja2 templates for systemd units and config files
└── docs/
    ├── architecture.md      # How Inferno AoIP works end-to-end
    ├── node-modes.md        # Spotify vs AUX mode detail
    └── variables.md         # Complete variable reference
```

---

## Documentation

- [Architecture](docs/architecture.md) — How Inferno AoIP works, Dante/PTP overview
- [Node modes](docs/node-modes.md) — Spotify vs AUX mode, hardware requirements
- [Variables](docs/variables.md) — Every Ansible variable explained

---

## Keeping credentials out of git

The `.gitignore` excludes `inventory-*.yml` (except `inventory.example.yml`). Name your personal inventory file `inventory-yournode.yml` and it will never be committed. For added safety, use [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) to encrypt `ansible_become_password`.
