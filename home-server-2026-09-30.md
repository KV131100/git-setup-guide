# Home Server Session Notes: 2026-09-30

What I did and learned today on my Ubuntu 24.04 home server (`karthik-nfs`): pushing a repo to GitHub, choosing a way to reach my files from anywhere, and starting a Nextcloud install.

> **This repo is public**, so private details are placeholders:
> `<tailnet>` = my tailnet name (from `tailscale status --json`, field `MagicDNSSuffix`), `<tailscale-ip>` = the server's Tailscale IP (`tailscale ip -4`), `<lan-ip>` = the server's home-network IP.

## Contents

1. [Where I stopped](#1-where-i-stopped)
2. [The server at a glance](#2-the-server-at-a-glance)
3. [Pushing linux-cheatsheet to GitHub](#3-pushing-linux-cheatsheet-to-github)
4. [Reaching my files from anywhere: the options](#4-reaching-my-files-from-anywhere-the-options)
5. [Choosing the app](#5-choosing-the-app)
6. [Speed: Nextcloud vs rsync vs rclone](#6-speed-nextcloud-vs-rsync-vs-rclone)
7. [Nextcloud: how it works](#7-nextcloud-how-it-works)
8. [Nextcloud: install steps](#8-nextcloud-install-steps)
9. [Showing external drives in Nextcloud](#9-showing-external-drives-in-nextcloud)
10. [Sharing links](#10-sharing-links)
11. [Command cheat sheet](#11-command-cheat-sheet)
12. [Lessons learned](#12-lessons-learned)

---

## 1. Where I stopped

| Step | Status |
|---|---|
| Push `linux-cheatsheet` to GitHub | ✅ Done |
| Log in the GitHub CLI (`gh auth login`) | ✅ Done |
| Nextcloud Step 1: enable HTTPS in Tailscale | ❓ Not confirmed yet |
| Nextcloud Step 2: `snap install nextcloud` (v34.0.4) | ✅ Done |
| Nextcloud Step 3: move to port 8081 | ✅ Done |
| Nextcloud Step 4: create admin account | ⏭️ **Next: do this soon** (see the warning in Step 4) |
| Nextcloud Steps 5–10 | ⬜ To do |
| External drive bind mounts | ⬜ To do |

---

## 2. The server at a glance

| Thing | Value | How I found it |
|---|---|---|
| OS | Ubuntu 24.04.5 LTS, x86_64 | `lsb_release -ds`, `uname -m` |
| CPU / RAM | 8 cores, 7.5 GB RAM | `nproc`, `free -h` |
| System disk | 256 GB NVMe, 100 GB system volume (LVM) | `lsblk`, `df -h` |
| External drive | 500 GB SanDisk Extreme, exFAT, mounted at `/srv/external` (98% full) | `lsblk`, `df -h /srv/external` |
| Samba shares | `[storage]` → `/srv/storage`, `[external]` → `/srv/external` | `grep path /etc/samba/smb.conf` |
| Network | Wi-Fi (`wlo1`) on `192.168.1.0/24`, plus Tailscale (`tailscale0`) | `ip -br addr` |
| Tailnet devices | server, MacBook, iPhone | `tailscale status` |

---

## 3. Pushing linux-cheatsheet to GitHub

**The problem:** the cheat sheet was committed locally, but it couldn't be pushed because the GitHub repo didn't exist, and the GitHub CLI (`gh`) wasn't logged in to create one.

**What I checked:**

```bash
gh auth status                                      # is the GitHub CLI logged in?
ssh -T git@github.com                               # does SSH to GitHub work?
git ls-remote git@github.com:KV131100/linux-cheatsheet.git   # does the repo exist?
```

`Repository not found` from `git ls-remote` meant the repo didn't exist on GitHub yet.

**The fix:**

```bash
gh auth login                                       # log in the CLI (choose GitHub.com → SSH)
cd ~/linux-cheatsheet
git remote add origin git@github.com:KV131100/linux-cheatsheet.git
git push -u origin main
```

Result: <https://github.com/KV131100/linux-cheatsheet>

**Next time,** `gh` can create the repo and push in one go:

```bash
gh repo create <name> --public --source . --push
```

---

## 4. Reaching my files from anywhere: the options

### Tailscale-only (what I chose)

The web app only answers on the Tailscale network. Only my own devices (MacBook, iPhone) can reach it. Traffic is already encrypted by Tailscale, and the Tailscale login itself is protected by my Google account's MFA.

### Public on the internet + MFA

| | Tailscale only | Public + MFA |
|---|---|---|
| Who can reach the login page | Only my devices | Anyone on the internet |
| Password-guessing bots | Can't reach it | Constant attempts |
| A security bug in the app | Only reachable by me | Can be exploited by anyone, possibly without logging in |
| Home IP address | Hidden | Visible, if I open a port on my router |
| Works from a borrowed computer | No | Yes |

**Key idea:** MFA protects the *login*, not the *program behind it*. The safest designs put the MFA check **in front of** the app, so strangers never reach the app at all.

Ways to go public, best first:

1. **Cloudflare Tunnel + Cloudflare Access.** No open router ports and the home IP stays hidden. Cloudflare asks for a login (with MFA) before anyone reaches the server. Needs a domain name (about $10 a year).
2. **Tailscale Funnel + an MFA gateway (Authelia or Authentik).** No domain needed, but Funnel lets *anyone* in, so the MFA has to come from something I set up myself.
3. **Router port forwarding + reverse proxy (Caddy) + Authelia.** The most exposure, and it may not work if my internet provider uses CGNAT.

If I ever go public: limit what the app can see to specific folders, keep it updated, and turn MFA on for every account.

---

## 5. Choosing the app

### File Browser vs Nextcloud

| | File Browser | Nextcloud |
|---|---|---|
| What it is | A web file manager | A self-hosted Google Drive/Dropbox |
| Setup | One program, about 10 minutes | Many parts, about 1–2 hours (easier with snap) |
| RAM used | About 20–50 MB | About 300 MB–1 GB+ |
| Web upload/download speed | Fast | Slower, especially with many small files |
| MFA | None built in | Yes (authenticator-app codes, security keys) |
| Desktop and phone apps | None, browser only | Folder sync on Mac, iPhone photo backup |
| Share links | Yes (expiry, password) | Yes, plus upload-only links |
| Upkeep | Almost none | Regular updates |

**I chose Nextcloud** because I want phone photo backup, folder sync and built-in MFA.

<details>
<summary>File Browser setup (the lighter alternative, not done)</summary>

```bash
curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash
mkdir -p ~/.config/filebrowser
DB=~/.config/filebrowser/filebrowser.db
filebrowser -d $DB config init
filebrowser -d $DB config set --address <tailscale-ip> --port 8080 --root /home/kvarma
 filebrowser -d $DB users add kvarma '<long-password>' --perm.admin   # leading space = not saved in history
```

Then run it as a systemd service with `User=kvarma`, `After=tailscaled.service` and `Restart=on-failure`. Listening on `<tailscale-ip>` (instead of `0.0.0.0`) is what makes it Tailscale-only.

</details>

### Why not CasaOS?

CasaOS is an app-store-style dashboard for home servers, and it runs every app in Docker. I skipped it because:

- Its main release has had no updates since v0.4.15 (December 2024). The company moved its focus to ZimaOS.
- It has had login-bypass security bugs in the past.
- **Docker ignores `ufw` firewall rules.** Apps Docker opens to the network skip ufw entirely. This catches many people out.
- It hides the Linux details I want to learn.

Worth trying later instead: **Cockpit** (`sudo apt install cockpit`, Ubuntu's own server dashboard on port 9090), **Immich** (a Google Photos replacement), and **Portainer** (a web page for managing Docker).

---

## 6. Speed: Nextcloud vs rsync vs rclone

| Situation | rsync | rclone | Nextcloud |
|---|---|---|---|
| One big file | Full network speed | Full network speed | Close to full speed |
| Thousands of small files | Fast | Fastest (sends several at once) | Much slower (per-file overhead) |
| Re-sync after small edits | Fastest (sends only the changed parts) | Re-sends changed files | Re-uploads the whole file |

- **The real speed limit is usually the network:** my home internet's *upload* speed when I'm away, and whether Tailscale connects **directly** (`tailscale status` shows `direct`, not `relay`).
- **Measure it:** run `iperf3 -s` on the server and `iperf3 -c karthik-nfs` on the Mac.
- **Never rsync straight into Nextcloud's own data folder.** Nextcloud won't know the files exist (it would need `occ files:scan`).
- **For big imports, use rclone into Nextcloud through its WebDAV address:** `https://karthik-nfs.<tailnet>.ts.net/remote.php/dav/files/kvarma/`. Use an **app password** (Personal settings → Security), because normal logins won't work for tools once MFA is on.
- **Samba over Tailscale** (`smb://karthik-nfs` in Finder) is also faster than any web page for big copies.

---

## 7. Nextcloud: how it works

| Part | Simple version | Job |
|---|---|---|
| Web server (Apache) | The receptionist | Answers the browser and apps |
| PHP | The brain | Nextcloud itself; runs on every click |
| Database (MySQL) | The notebook | Remembers users, the list of files, shares and settings (not the files themselves) |
| Redis | Short-term memory | Speeds things up and stops two edits clashing |
| Data folder | The warehouse | Where the actual files are stored |
| Background job | The cleaner | Runs every 5 minutes: previews, cleanup, notifications |

I installed it with **snap**: all six parts in one package, with automatic updates. After installing, `systemctl list-units 'snap.nextcloud.*'` shows each part running as its own service (`apache`, `php-fpm`, `mysql`, `redis-server`, `nextcloud-cron`).

**How the pieces connect:**

```
iPhone / Mac (Tailscale on)
        │  https://karthik-nfs.<tailnet>.ts.net
        ▼
tailscale serve   ← adds HTTPS with a real certificate, tailnet devices only
        │  forwards to 127.0.0.1:8081 (inside the server)
        ▼
Nextcloud (snap)  ← ufw blocks 8081 from Wi-Fi and the internet
```

---

## 8. Nextcloud: install steps

### Step 1: Enable HTTPS in Tailscale (browser) ❓

Go to **login.tailscale.com/admin/dns** → make sure **MagicDNS** is on → **Enable HTTPS**.
*Why:* Tailscale can then get a trusted certificate for `karthik-nfs.<tailnet>.ts.net`.

### Step 2: Install ✅

```bash
sudo snap install nextcloud
snap list nextcloud
```

### Step 3: Move off port 80 ✅

```bash
sudo snap set nextcloud ports.http=8081
ss -tln | grep -E ':(80|8081) '       # expect :8081 and nothing on :80
```

*Why:* the snap listens on **every** network and can't be limited to one, so I moved it to a spare port and block that port with the firewall.

### Step 4: Create the admin account ⏭️ next

```bash
 sudo nextcloud.manual-install kvarma '<long-unique-password>'
```

⚠️ **Until this is done,** anyone on my Wi-Fi who opens `http://<lan-ip>:8081` sees the setup page and could create the admin account themselves.
The **leading space** keeps the password out of `~/.bash_history`.

### Step 5: Tell Nextcloud its address

```bash
sudo nextcloud.occ config:system:set trusted_domains 1 --value=karthik-nfs.<tailnet>.ts.net
sudo nextcloud.occ config:system:set overwrite.cli.url --value=https://karthik-nfs.<tailnet>.ts.net
sudo nextcloud.occ config:system:set overwriteprotocol --value=https
sudo nextcloud.occ config:system:set trusted_proxies 0 --value=127.0.0.1
sudo nextcloud.occ config:system:set default_phone_region --value=US

sudo nextcloud.occ config:system:get trusted_domains   # check
sudo nextcloud.occ status                               # expect installed: true
```

- `occ` is Nextcloud's command-line control panel.
- `trusted_domains` is the list of addresses Nextcloud will answer to; any other address is refused.
- `overwriteprotocol https` is needed because Tailscale hands Nextcloud plain http inside the server while my browser uses https. Without it: wrong `http://` links and redirect loops.
- `trusted_proxies 127.0.0.1` trusts Tailscale's forwarder to pass along my device's real details.

### Step 6: HTTPS through Tailscale

```bash
sudo tailscale serve --bg http://127.0.0.1:8081
tailscale serve status
```

`--bg` keeps it running after I close the terminal and after reboots. To undo: `sudo tailscale serve reset`.

### Step 7: Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow in on tailscale0
sudo ufw allow from 192.168.1.0/24 to any app Samba
sudo ufw enable
sudo ufw status verbose
```

- ufw blocks all incoming connections except what I allow.
- SSH stays open, Tailscale devices are trusted, and Samba keeps working on home Wi-Fi.
- Port 8081 is blocked from outside, but `tailscale serve` still reaches it because it connects from inside the server.
- ⚠️ **Keep the current SSH session open,** and test a new SSH login in a second terminal before closing it. If it fails: `sudo ufw disable`.

### Step 8: Log in and turn on MFA

Open `https://karthik-nfs.<tailnet>.ts.net`, then:
**Apps → Security → Two-Factor TOTP Provider → Enable**, and **Personal settings → Security → enable TOTP**. Scan the QR code with an authenticator app and **save the backup codes**.

### Step 9: Health check

**Administration settings → Overview**: fix any warnings it lists.

### Step 10: Connect devices

- **iPhone:** Nextcloud app → server address → **More → Auto upload** for camera photos.
- **Mac:** Nextcloud desktop app → pick a folder to keep in sync.

### Check it worked

- On the phone, with Wi-Fi on and **Tailscale off**, `http://<lan-ip>:8081` must **fail to load**.
- With Tailscale **on**, the `https://…ts.net` address must work.

### Upkeep

- **Updates:** snap updates Nextcloud automatically. `snap refresh --time` shows the next check.
- **Backup:** `sudo nextcloud.export` saves settings, database and files to `/var/snap/nextcloud/common/backups/`. Copy that to another disk.
- **Uninstall:** `sudo snap remove nextcloud` also **deletes the data**, so back up first.

---

## 9. Showing external drives in Nextcloud

Drives don't appear automatically, for two reasons:

1. Nextcloud only shows its own data folder unless I add other folders with the **External storage** app.
2. The snap is sandboxed: it can only see `/mnt` and `/media`, and only after `snap connect … removable-media`. My drive is at `/srv/external`.

**Fix: a bind mount.** This makes the same folder also appear under `/mnt`. It's not a copy. Samba keeps using `/srv/…` and Nextcloud uses `/mnt/…`.

### Part A: Bind mounts (can be done any time)

```bash
sudo mkdir -p /mnt/external /mnt/storage
sudo nano /etc/fstab
```

Add these lines:

```
/srv/external  /mnt/external  none  bind,nofail,x-systemd.requires-mounts-for=/srv/external  0  0
/srv/storage   /mnt/storage   none  bind,nofail  0  0
```

Then:

```bash
sudo systemctl daemon-reload
sudo mount -a                # apply now; a typo errors here, not at the next boot
findmnt /mnt/external
```

- `bind`: show an existing folder at a second location as well.
- `nofail`: if the USB drive is unplugged, the server still boots.
- `x-systemd.requires-mounts-for=`: only mount once the drive is actually there, so Nextcloud never shows an empty folder by mistake.

### Part B: Connect to Nextcloud (after Step 8)

```bash
sudo snap connect nextcloud:removable-media
sudo nextcloud.occ app:enable files_external
sudo nextcloud.occ files_external:create "External Drive" local null::null -c datadir=/mnt/external
sudo nextcloud.occ files_external:create "Storage" local null::null -c datadir=/mnt/storage
sudo nextcloud.occ files_external:list
```

Notes:

- Files added through Samba or rsync still appear. Nextcloud checks external folders for changes when you open them, so rsync into *these* folders is fine.
- The first time you open a big folder it will be slow, because Nextcloud is indexing it.
- **Don't** move Nextcloud's own data folder onto the exFAT drive. exFAT doesn't support Linux file permissions.
- The drive is **98% full**, so uploads will fail once it's out of space.

---

## 10. Sharing links

**Share icon → Share link → ⋯** gives these options: read-only, allow editing, **file drop** (upload only), password, expiry date, hide download.

**Catch:** with a Tailscale-only setup, **only tailnet devices can open the links.** Options:

1. **Share the server through Tailscale** (admin page → Machines → ⋯ → Share). The other person installs Tailscale and can reach only this server. Best for family and friends.
2. **Tailscale Funnel** (`sudo tailscale funnel --bg http://127.0.0.1:8081`) makes all of Nextcloud public, *including the login page*. Only do this with MFA on. Undo with `sudo tailscale funnel reset`, then re-run the `serve` command.
3. Use iCloud or Google Drive for the occasional share with strangers.

---

## 11. Command cheat sheet

### Looking around (read-only)

| Command | What it shows |
|---|---|
| `tailscale status` | Devices on my tailnet, and `direct` vs `relay` connections |
| `tailscale ip -4` | This server's Tailscale IP |
| `tailscale serve status` | What Tailscale is currently forwarding |
| `ss -tlnp` | Which ports are open, and on which address (`0.0.0.0`/`*` = every network) |
| `ip -br addr` | Network cards and their IPs |
| `lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS,MODEL` | Disks, partitions and where they're mounted |
| `findmnt <path>` | What is mounted at a path |
| `df -h` | Disk space used and free |
| `free -h` / `nproc` | RAM / CPU cores |
| `cat /etc/fstab` | Mounts that happen at boot |
| `snap list` | Installed snaps and versions |
| `systemctl list-units 'snap.nextcloud.*'` | Nextcloud's parts and whether each is running |
| `systemctl status <service>` | One service's status and recent log lines |
| `gh auth status` | Whether the GitHub CLI is logged in |
| `git ls-remote <url>` | Whether a remote repo exists and what's in it |

### Changing things

| Command | What it does |
|---|---|
| `sudo snap install nextcloud` | Install Nextcloud |
| `sudo snap set nextcloud ports.http=8081` | Change Nextcloud's port |
| `sudo nextcloud.occ …` | Nextcloud's command-line control panel |
| `sudo tailscale serve --bg <url>` | Serve something over HTTPS on the tailnet |
| `sudo ufw allow … / enable / status` | Firewall rules |
| `sudo mount -a` | Apply `/etc/fstab` now (tests for typos) |
| `gh repo create <name> --public --source . --push` | Create a GitHub repo and push in one step |

---

## 12. Lessons learned

- **Check before you change.** Most of today's decisions came from read-only checks: `ss`, `lsblk`, `tailscale status`, `df`.
- **Where a service listens is a security decision.** `0.0.0.0` or `*` means every network. Listen on a specific IP, or add a firewall.
- **Tailscale-only is the easy safe default.** Going public means planning for bots, security bugs and MFA.
- **MFA belongs in front of the app,** not only inside it.
- **Docker skips `ufw`.** A port "blocked" in ufw can still be open if Docker opened it.
- **snap packages are sandboxed.** They can't see arbitrary folders such as `/srv`. Use `/mnt` or `/media` and `snap connect`.
- **Bind mounts show one folder at two paths** without copying anything.
- **After editing fstab, run `mount -a`** so a typo shows up now, not at the next boot.
- **A leading space before a command** keeps secrets out of bash history.
- **Keep one SSH session open** while changing the firewall or SSH settings.
- **The network, not the app, usually limits speed.** Check your home upload speed and whether Tailscale connects directly.
