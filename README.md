# vm-init

Post-install setup for a fresh Ubuntu or Debian VM.

A new VM needs the same ten minutes every time: update it, install the few
tools you always want, get the guest agent talking to the hypervisor, mount
the NAS, turn off IPv6 if your network does not use it, grow the disk to what
you gave it, close SSH to passwords. This script does that in one pass. It
detects the system, asks what you want, shows the plan, then applies it. Every
step can be declined, and nothing secret is written in the script.

```
$ vm-init

System
  os            Ubuntu 24.04.2 LTS
  kernel        6.8.0-58-generic
  hypervisor    QEMU/KVM, guest agent not installed
  root volume   /dev/ubuntu-vg/ubuntu-lv 20 GB, 30 GB free in the volume group
  ipv6          enabled on ens18
  ssh           root login key only, password login allowed, key present for louis
  shares        none mounted

Questions
  press enter to accept the value in brackets

  Update the system and install base tools? [Y/n]
  Enable automatic security updates? [Y/n]
  Owner of /srv and /opt [louis] (- for none):
  Mount a network share? [y/N] y
    Share, e.g. //server/name: //nas.lan/Media
    Mount point [/mnt/media]:
    User: bob
    Password:
    Read-only? [y/N]
  Disable IPv6? [y/N] y
  Extend the root volume over the 30 GB free? [Y/n]
  Harden SSH? [Y/n]

To apply
  system        update, 9 base packages
  auto-updates  security updates applied unattended
  guest agent   installed and started
  owner         louis on /srv and /opt
  share         //nas.lan/Media on /mnt/media, read-write, remounted at boot
  ipv6          disabled
  root volume   extended by 30 GB
  ssh           root login off, password login off

  Apply these 8 steps? [Y/n]

┌─ [1/8] System update  21:40:12
│  52 packages to upgrade, 7 base tools to install
│  ✓ libssl3t64
│  ✓ openssl
│  ✓ curl
│  ...
└─ ✓ 52 packages upgraded, 7 tools installed

┌─ [2/8] Automatic updates  21:41:30
│  /etc/apt/apt.conf.d/20auto-upgrades
└─ ✓ enabled, security fixes install on their own

┌─ [3/8] Guest agent  21:41:31
│  ✓ qemu-guest-agent
└─ ✓ running, the hypervisor can see this VM

┌─ [4/8] Owner of /srv and /opt  21:41:33
│  owner louis, group louis, new files inherit the group
└─ ✓ done

┌─ [5/8] Network share  21:41:33
│  nas.lan answers on port 445
│  credentials in /etc/cifs-media.cred, readable by root only
│  /etc/fstab: //nas.lan/Media on /mnt/media, read-write, mounted at every boot
└─ ✓ mounted, 8 entries, 2.1T free

┌─ [6/8] IPv6  21:41:34
│  /etc/sysctl.d/99-vm-init-ipv6.conf: off now and at every boot
│  /etc/netplan/99-vm-init-ipv6.yaml: no IPv6 on ens18
└─ ✓ disabled, no address left on ens18

┌─ [7/8] Root volume  21:41:36
│  /dev/ubuntu-vg/ubuntu-lv: adding the 30 GB left in the volume group
└─ ✓ root filesystem 20G -> 49G

┌─ [8/8] SSH  21:41:37
│  /etc/ssh/sshd_config.d/10-vm-init.conf
│  config checked by sshd, service restarted
└─ ✓ root login off, password login off, key present for louis

──────────────────────────────────────────────────────────────
Done in 1 min 25 s, 0 warnings
  share         /mnt/media, 2.1T free
  ssh           root login off, password login off, key present for louis
  reboot        required - sudo reboot
  log           /var/log/vm-init.log

  The kernel or a core library was updated and only takes effect after a reboot.
  Reboot now? [Y/n]
  rebooting
```

Each question comes with one plain line saying what it changes, left out
above for brevity. While a step runs, a spinner shows what is going on and
apt prints one tick per package configured, so a long upgrade never looks
stuck. A step that could not do everything ends with `!` instead of a tick,
and the last line counts those.

## Install

```bash
git clone https://github.com/Lousblue/vm-init.git
sudo install -m 755 vm-init/vm-init /usr/local/bin/
```

One bash script. It needs `sudo` or root, and `apt-get`.

## Use

```bash
vm-init                                # ask, showing what was detected
vm-init --status                       # look, change nothing
vm-init --yes                          # every default, no questions
vm-init --share //nas.lan/Media --no-ipv6 --yes
vm-init --no-update --harden-ssh       # one step only
vm-init --share //nas.lan/Media --dry-run
```

Pass any option and it stops asking - which is also what happens without a
terminal, so it is safe to call from cloud-init or a provisioning tool. What
you did not mention takes its default.

### Steps

| Step | Default | Option |
|---|---|---|
| Update, then install `curl wget git nano htop tree ca-certificates gnupg iputils-ping` | yes | `--no-update` |
| Enable unattended security updates | yes | `--no-auto-updates` |
| Make a user the owner of `/srv` and `/opt`, setgid set | whoever ran sudo | `--user NAME`, `--no-owner` |
| Mount an SMB share, remounted at every boot | no | `--share //SRV/NAME`, `--mount PATH`, `--read-only` |
| Disable IPv6, sysctl and netplan where present | no | `--no-ipv6` |
| Grow the root volume over the free space in its volume group | yes, when there is some | `--no-extend` |
| Harden SSH | yes, when it cannot lock you out | `--harden-ssh`, `--no-harden-ssh` |

The QEMU guest agent is not a question. When the hypervisor exposes its
channel, the agent is installed and started; when the channel is missing on a
KVM guest, the script says which box to tick in the VM options.

### Other options

| | |
|---|---|
| `--status` | show what was detected, change nothing |
| `--dry-run` | show what would be done, then stop |
| `--yes`, `-y` | never ask, take every default |

## The share password

The script reads `NAS_USER` and `NAS_PASS` from its environment. Without
them, it asks, with the password hidden. Without them and without a terminal,
it stops rather than guess. So all three of these work:

```bash
vm-init --share //nas.lan/Media                          # asks
NAS_USER=bob NAS_PASS='...' vm-init --share //nas.lan/Media --yes
pass-cli run --env-file nas.env -- vm-init --share //nas.lan/Media --yes
```

The last one is a password manager filling the variables. Any manager that
can put a secret in the environment will do; the script never knows which.

On the machine, the credentials end up in `/etc/cifs-NAME.cred`, root only,
mode 600, referenced from `/etc/fstab`. They are never printed and never
logged.

## SSH hardening, without locking you out

The script protects whoever is actually logging in: the user who ran `sudo`,
or root when it runs as root.

| Situation | What it writes |
|---|---|
| sudo user with a key in `authorized_keys` | root login off, password login off |
| sudo user without a key | root login off, password login kept, and it tells you |
| root with a key | root login by key only, password login off |
| root without a key | nothing, with the reason |

The drop-in is `/etc/ssh/sshd_config.d/10-vm-init.conf`, named to sort before
the one cloud images ship, since sshd keeps the first value it reads. The
config is validated with `sshd -t` before the service restarts; if it fails,
the file is removed and nothing changes.

## Files it writes

| | |
|---|---|
| `/etc/apt/apt.conf.d/20auto-upgrades` | unattended security updates |
| `/etc/cifs-NAME.cred`, a line in `/etc/fstab` | the share |
| `/etc/sysctl.d/99-vm-init-ipv6.conf`, `/etc/netplan/99-vm-init-ipv6.yaml` | IPv6 off |
| `/etc/ssh/sshd_config.d/10-vm-init.conf` | SSH |
| `/var/log/vm-init.log` | everything apt and mount had to say |

Running it again is safe: an existing fstab line for the same mount point is
replaced, not duplicated, and steps already in effect are skipped.

## Tested on

Ubuntu 24.04 and Debian 12, including a real SMB mount against a Samba
server and the interactive path. The root volume extension is plain
`lvextend -r -l +100%FREE` and has only been exercised on Ubuntu's default
LVM layout.

## Caveats

- **It is opinionated about small things**: the base package list, setgid on
  `/srv` and `/opt`, `file_mode=0644,dir_mode=0755` on the share. Edit the
  top of the script if yours differ.
- **Disabling IPv6 on a network that relies on it** will cut you off. The
  default is to keep it.
- **The share is mounted with `nofail`**, so a NAS that is down at boot delays
  nothing and breaks nothing; the mount point is just empty until `mount -a`.
- Everything but `--help` needs root or sudo, `--status` included, because
  reading the LVM layout and the effective sshd config requires it.

## Licence

MIT.
