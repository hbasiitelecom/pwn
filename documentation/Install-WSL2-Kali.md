# Installing PWN on Windows (Kali VM or WSL2)

On Windows there are two great ways to run PWN, both based on **Kali** - a
first-class PWN target that ships Ruby ≥ 3.3 and most of the external toolchain
`pwn setup` looks for:

1. **A full Kali VM** (Hyper-V / VMware / VirtualBox) - **the most compatible
   option**. It is just a normal Kali machine: real kernel, `systemd`, and
   full network + USB handled by the hypervisor, with none of the WSL caveats.
   If you already have one, jump straight to
   [step 4 - Install PWN](#4-install-pwn); the WSL-specific steps 1-3 do not
   apply.
2. **Kali under WSL2** - best if you want tight Windows integration (shared
   filesystem, launch from a Windows terminal). Covered in full below.

Either way, prefer a **persistent** environment over a throwaway container:
PWN's value is its **persistent feedback loop** (`~/.pwn/`) and its
**cron worker** (`systemd --user`), and both degrade in a stock container.

> **Already have a Kali VM (e.g. VMware)?** You are done with setup - go to
> [step 4](#4-install-pwn) and run `gem install pwn && pwn setup`. Reserve a
> **container** only for isolated, throwaway tasks (a one-shot SAST run, a web
> test in a clean environment).

---

## WSL2 route

The rest of this page sets up the **WSL2 + Kali** route. `systemd`, native
package management, and (with two optional tweaks) full network and USB access
make WSL2 nearly as capable as a full VM while integrating with Windows.

- **Kali** is a first-class PWN target and already ships most of the external
  toolchain `pwn setup` looks for (nmap, metasploit, sqlmap, radare2, tshark,
  ...) plus Ruby ≥ 3.3, so `pwn setup` finds far more on the first pass than a
  bare Ubuntu.
- **WSL2** (not WSL1) gives you a real Linux kernel, `systemd`, raw sockets,
  and USB passthrough - everything PWN's networking, packet, SDR, and hardware
  capabilities expect.

> Prefer a **container** only for isolated, throwaway tasks (a one-shot SAST
> run, a web test in a clean environment). For day-to-day PWN use, WSL2 wins.
> See [Installation](Installation.md) for the platform-agnostic gem install.

---

## 1. Install WSL2 + Kali

From an **elevated PowerShell** on Windows 10/11:

```powershell
wsl --install -d kali-linux     # installs WSL2 + the Kali distro
wsl --set-default-version 2     # ensure new distros are WSL2, not WSL1
wsl --update                    # pull the latest WSL2 kernel (systemd support)
```

Reboot if prompted, then launch **Kali** from the Start menu and create your
UNIX user when asked.

Confirm you are on WSL2 (VERSION must be `2`):

```powershell
wsl -l -v
```

---

## 2. Enable systemd (required for PWN's cron worker)

PWN's background learning/hygiene jobs run under `systemd --user`. Recent WSL2
enables systemd by default, but make it explicit. Inside Kali:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

Then, from **PowerShell**, restart the distro so it takes effect:

```powershell
wsl --shutdown
```

Reopen Kali and verify:

```bash
systemctl is-system-running     # "running" or "degraded" is fine; not "offline"
systemctl --user is-enabled default.target 2>/dev/null || echo "user bus ok once logged in"
```

---

## 3. Recommended Windows-side tweaks (optional but high-value)

Create/edit `C:\Users\<you>\.wslconfig` (Windows side, **not** inside Kali):

```ini
[wsl2]
# Mirrored networking: WSL shares the Windows host's network stack.
# Much better for scanning, packet capture, and reaching lab targets.
networkingMode=mirrored

# Optional resource caps - tune to your machine.
memory=8GB
processors=4
```

Apply with `wsl --shutdown` from PowerShell, then reopen Kali.

**USB / SDR / serial hardware** (only when you need `PWN::SDR`,
`PWN::Plugins::Serial`, BusPirate, RFID, etc.): install
[`usbipd-win`](https://github.com/dorssel/usbipd-win) on Windows and attach
the device into WSL:

```powershell
winget install usbipd
usbipd list                     # find the BUSID of your dongle
usbipd bind   --busid <BUSID>
usbipd attach --wsl --busid <BUSID>
```

---

## 4. Install PWN

Inside Kali:

```bash
sudo apt update
sudo apt install -y ruby ruby-dev build-essential   # Kali already ships Ruby >= 3.3
ruby -v                                              # confirm >= 3.3

gem install --verbose pwn        # 1. get the gem (pure-Ruby core always works)
pwn setup                        # 2. read-only doctor: what's usable / missing
```

`pwn setup` (with no flags) only reports - it changes nothing. Read its
per-capability output, then provision the profile(s) you actually need
(see [Installation § capability profiles](Installation.md#capability-profiles)):

```bash
pwn setup --list-profiles
pwn setup --profile web --yes    # TransparentBrowser · Burp · ZAP · Tor · sqlmap
pwn setup --profile net --yes    # NmapIt · Packet · extro_packet · tshark/tcpdump
pwn setup --profile re  --yes    # Radare2 · GDB · Frida · ropper · ...
# or everything:
pwn setup --profile full --yes
```

Tip: `pwn setup --profile net --dry-run` prints the exact `apt` commands
without running them - handy to review first.

---

## 5. Launch & verify

```bash
pwn                               # the REPL (Pry with every PWN:: preloaded)
```

```ruby
pwn[CURRENT_VERSION]:001 >>> PWN::Setup.check[:ok]          # => true
pwn[CURRENT_VERSION]:002 >>> PWN::Plugins.constants.count   # plugin count
pwn[CURRENT_VERSION]:003 >>> PWN.help                       # list every module
pwn[CURRENT_VERSION]:004 >>> pwn-ai                         # agent TUI (needs an AI key, see Configuration)
```

Headless one-shot (CI-friendly), same code path as the REPL:

```bash
pwn --ai 'What ports are listening on this host?'
```

Configure at least one AI engine key/oauth in `~/.pwn/pwn.yaml` before using
`pwn-ai` - see [Configuration](Configuration.md) and
[AI Integration](AI-Integration.md).

---

## Keeping the fork current

If you are running from a fork, add the upstream once so you can sync later:

```bash
git remote add upstream https://github.com/0dayInc/pwn.git
git fetch upstream
git merge upstream/master        # or: git rebase upstream/master
```

For a gem install, just:

```bash
gem update pwn
pwn setup                         # re-doctor: new versions may add capabilities
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `pwn setup` reports Ruby too old | Kali system Ruby should be ≥ 3.3; if not, use `rbenv`/`asdf` to install 3.3+ |
| Native gem (e.g. `pg`, `pcaprub`) fails to build | Install the OS header `pwn setup` names, then `gem pristine <gem>` |
| Cron worker shows `offline` | systemd not enabled - redo step 2, then `wsl --shutdown` |
| Can't reach lab targets / capture packets | Switch to `networkingMode=mirrored` (step 3), `wsl --shutdown` |
| SDR/serial device invisible in Kali | Attach it with `usbipd attach --wsl` (step 3) |

**Next:** [Configuration](Configuration.md) ·
[General Usage](General-PWN-Usage.md) ·
[Persistence](Persistence.md) ·
[← Installation](Installation.md)

[← Home](Home.md)
