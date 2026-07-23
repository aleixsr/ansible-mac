# ansible-mac

> **Temporal:** alguns casks (p. ex. Microsoft Teams) instal·len un `.pkg` que crida `sudo` internament, i com que Ansible executa els mòduls sense terminal interactiu, `sudo` no pot demanar la contrasenya i el run falla. Mentre no hi hagi una solució millor, cal afegir aquesta línia amb `sudo visudo`:
> ```
> %admin          ALL = (ALL) NOPASSWD:ALL
> ```
> Recorda revertir-ho quan ja no calgui, ja que dona a qualsevol usuari del grup `admin` sudo sense contrasenya.
>
> També cal desactivar Gatekeeper globalment perquè el `.pkg` s'instal·li sense bloquejar-se en apps sense notaritzar:
> ```
> sudo spctl --global-disable
> ```
> Torna-ho a activar quan acabis (`sudo spctl --global-enable`), ja que deixa el Mac sense la verificació de Gatekeeper per a tot el software instal·lat.

Personal Ansible playbook to provision and configure a macOS machine: Homebrew packages/casks, Mac App Store apps, login items, shell setup (Starship + Zsh), macOS system preferences, and Karabiner-Elements key mappings.

## What it does

Running the playbook will:

1. **Install Homebrew packages and casks** (`main.yml` + `config.yml`)
   - CLI packages grouped by category: development, cloud/DevOps, networking, system utilities
   - GUI apps (casks) grouped by category: browsers, development, terminal, communication, productivity, networking/VPN, remote access, file management/cloud, system utilities, Microsoft suite, media, documents, hardware
   - Taps `teamookla/speedtest`
2. **Install Mac App Store apps** (`roles/appstore`) via `mas`, skipping gracefully (with a summary) if an app isn't available.
3. **Add login items** (`roles/startup`) so selected apps launch automatically at login.
4. **Configure the shell** (`roles/shell`):
   - Installs Starship and Zsh plugins (`zsh-autosuggestions`, `zsh-syntax-highlighting`)
   - Installs the Meslo Nerd Font (for Starship icons)
   - Writes/updates `~/.zshrc` with plugin sourcing, Starship init, and aliases (idempotent via `blockinfile` markers)
   - Deploys a custom `starship.toml`
5. **Apply macOS system preferences** (`roles/desktop`) via `osx_defaults`:
   - Hide desktop icons
   - Set hot corners (top-left/top-right → Mission Control)
   - Trackpad tuning (tap to click, Force Click, two-finger secondary click, dragging, no drag lock, disable natural scrolling)
   - Accessibility zoom with Ctrl + scroll
   - Use F1–F12 as standard function keys
6. **Deploy Karabiner-Elements complex modifications** (`roles/karabiner`) by copying JSON rule files from `files/karabiner/` into `~/.config/karabiner/assets/complex_modifications/`.
7. **Install Tabby plugins** (`roles/tabby`) via `npm install --prefix`, into `~/Library/Application Support/tabby/plugins` (requires `node`, installed as part of step 1). Currently installs `terminus-sync-config` (syncs Tabby's config to a GitHub/Gitee Gist — configure the token/gist ID from Tabby's own Settings UI).

## Requirements

- macOS
- [Homebrew](https://brew.sh/) — if it's not installed yet, run:
  ```bash
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  ```
  Then follow the "Next steps" printed by the installer to add Homebrew to your `PATH` (usually appending an `eval "$(/opt/homebrew/bin/brew shellenv)"` line to your shell profile on Apple Silicon, or `/usr/local/bin/brew shellenv` on Intel), then reload your shell (`source ~/.zprofile` or open a new terminal).
- [Ansible](https://www.ansible.com/) installed via Homebrew:
  ```bash
  brew install ansible
  ```
- Ansible collection `community.general` (provides `homebrew`, `homebrew_cask`, `homebrew_tap`, `osx_defaults`, `mas` modules):
  ```bash
  ansible-galaxy collection install community.general
  ```

## Usage

Run the full playbook:

```bash
ansible-playbook main.yml
```

Run only specific parts using tags (defined throughout `main.yml` and the roles):

```bash
# Only install Homebrew packages/casks
ansible-playbook main.yml --tags packages,casks

# Only configure the shell (Starship + Zsh)
ansible-playbook main.yml --tags shell

# Only apply desktop/trackpad/hotcorner preferences
ansible-playbook main.yml --tags desktop,hotcorners,input,accessibility,keyboard

# Only deploy Karabiner rules
ansible-playbook main.yml --tags karabiner

# Only Mac App Store apps
ansible-playbook main.yml --tags appstore

# Only login items
ansible-playbook main.yml --tags startup

# Only Tabby plugins
ansible-playbook main.yml --tags tabby
```

## Configuration

All packages, casks, App Store apps, and startup apps are centralized in [`config.yml`](config.yml). Edit this file to add/remove software without touching `main.yml`:

```yaml
packagesdevelopment:
  - gh
  - git

casksbrowsers:
  - brave-browser
  - google-chrome

mas_apps:
  1274495053: "Microsoft To Do"

startup_apps:
  - Maccy
```

## Installed software

### Homebrew packages (CLI)

| Category | Packages |
|---|---|
| Development | `gh`, `git`, `node`, `python@3.14` |
| Cloud / DevOps | `azure-cli`, `oci-cli` |
| Networking | `iperf3`, `rustscan`, `mtr`, `nmap`, `speedtest` (via `teamookla/speedtest` tap), `swaks`, `tcping`, `wget` |
| System utilities | `htop`, `mas` |
| Shell (via `roles/shell`) | `starship`, `zsh-autosuggestions`, `zsh-syntax-highlighting` |

### Homebrew casks (GUI apps)

| Category | Apps |
|---|---|
| Browsers | Brave Browser, Chromium, Firefox, Google Chrome, Microsoft Edge |
| Development | Apache Directory Studio, Copilot CLI, DBeaver Community, draw.io, GitHub Desktop, SoapUI, Sublime Text, Visual Studio Code, Visual Studio Code Insiders |
| Terminal | iTerm2, PowerShell, Tabby, Warp |
| Communication | Mailspring, Shortwave, Telegram, WhatsApp, Zoho Mail |
| Productivity | AltTab, Caffeine, Claude, DockDoor, Keka, Maccy, Notion, Numi, Rectangle, Shottr, Stats |
| Networking & VPN | SwitchHosts, Tailscale, Tunnelblick |
| Remote access | Royal TSX, RustDesk, Windows App |
| File management & cloud | Box Drive, LocalSend, Synology Drive |
| System utilities | balenaEtcher, BetterDisplay, Bluesnooze, Disk Inventory X, Karabiner-Elements (bundles EventViewer), LICEcap, macFUSE, NovaBench, Resolutionator, SuperDuper!, Wins, XCA |
| Microsoft suite | Microsoft AutoUpdate, Microsoft Azure Storage Explorer, Microsoft Office (bundles OneDrive) |
| Media | HandBrake, VLC |
| Documents | Adobe Acrobat Reader, MacDown, Mark Text, Modern CSV, ONLYOFFICE, RevPDF Editor, Xournal++ (deprecated by its author — macOS Gatekeeper will disable it on 2026-09-01) |
| Hardware | Logi Options+ |
| Fonts (via `roles/shell`) | Meslo Nerd Font (`font-meslo-lg-nerd-font`) |

Not available as a Homebrew cask — install manually if needed: **FortiClient** (forticlient.com), **XtraFinder** (discontinued/incompatible with recent macOS).

**Microsoft Teams** is intentionally left out of `config.yml` — its `.pkg` requires an interactive `sudo` prompt that fails under Ansible (see the temporary `sudoers` workaround at the top of this README). Install it manually with `brew install --cask microsoft-teams`, or re-add it to `casksmicrosoftsuite`/`caskscommunication` once the `NOPASSWD` rule is in place.

### Mac App Store apps (via `mas`)

| App | ID |
|---|---|
| Ping Status | 1158928913 |
| Microsoft To Do | 1274495053 |
| MuteKey | 1509590766 |
| Azure VPN Client | 1553936137 |
| WireGuard | 1451685025 |

### Startup / login items

BetterDisplay, Maccy, Shottr, Stats, MuteKey, OneDrive

## Project structure

```
main.yml                          Entry point: Homebrew tasks + role includes
config.yml                        Centralized package/cask/app list
roles/
  appstore/tasks/main.yml         Mac App Store app installation (mas)
  desktop/tasks/main.yml          macOS defaults: desktop, hot corners, trackpad, accessibility, keyboard
  karabiner/tasks/main.yml        Deploys Karabiner-Elements complex modifications
  shell/tasks/main.yml            Starship + Zsh setup
  shell/files/starship.toml       Starship prompt configuration
  startup/tasks/main.yml          Login items management
  tabby/tasks/main.yml            Installs Tabby plugins via npm
files/karabiner/                 Karabiner-Elements JSON rule files
```

## Notes

- Tasks are idempotent where possible (`blockinfile`, `osx_defaults`, `homebrew`/`homebrew_cask` with `state: present`).
- The shell role fetches the Homebrew prefix dynamically (`brew --prefix`) so it works on both Apple Silicon and Intel Macs.
- App Store installs are wrapped with `ignore_errors` so a missing/unavailable app doesn't stop the whole run; skipped apps are reported at the end.