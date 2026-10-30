# Homepage: One Dashboard for the Whole Lab (Proxmox LXC)

[← Back to lab overview](../README.md) · [← Previous: Uptime Kuma](../03-uptime-kuma/)

Part of my **Home SOC Lab v2** rebuild, and the last piece of Phase 1. With
remote access, DNS and monitoring running, I had four separate web UIs on four
addresses. Homepage puts all of them on **one page with live data**, so a
single glance answers "is everything OK?"

![Final dashboard](screenshots/69-final-dashboard.png)

---

## What Homepage does

Homepage is a free, open-source (GPL-3.0) dashboard. It's configured entirely
with **YAML files**, and it doesn't just link to services. It talks to their
APIs and shows live numbers on each tile:

| Tile | Live data it shows |
|---|---|
| Proxmox | VMs and containers running, host CPU and RAM |
| AdGuard Home | DNS queries, blocked count, filtered, latency |
| Uptime Kuma | Sites up, sites down, uptime % |
| Router, Tailscale, portfolio site | Green/red status dot (ping or HTTP check) |

The top bar adds a greeting, date and time, live weather and a search box,
and the bottom holds bookmark groups.

## Why it's worth running

| Benefit | What it means in practice |
|---|---|
| **One screen for the whole lab** | No more remembering four IPs and ports. Every service is one click away. |
| **Health at a glance** | Status dots and live stats show a problem before I open anything. Uptime Kuma *alerts*; Homepage gives the *overview*. |
| **Config as code** | The whole dashboard is four YAML files, committed to this repo ([`config/`](config/)). Rebuilding it is a copy-paste. |
| **Practice with real security patterns** | API tokens with least privilege, secrets kept out of config files, file permissions. The same habits I'd use at work. |
| **SOC toolkit on hand** | Bookmarks for the tools an analyst uses daily: CyberChef, VirusTotal, AbuseIPDB, MITRE ATT&CK, Shodan, NVD. |
| **Small** | About 170 MB of RAM in use |

**What this shows:** least-privilege API access · secrets management ·
YAML config-as-code · reading logs to troubleshoot · file signatures
(magic bytes) · Linux file permissions

---

## Goal

| Item | Value |
|---|---|
| Container | Unprivileged LXC, Debian 13 |
| CT ID / hostname | `103` / `homepage` |
| Resources | 2 vCPU · 4096 MB RAM · 6 GB disk (the script's default profile) |
| Network | `vmbr0`, static **192.168.50.103/24**, gateway `192.168.50.1` |
| DNS server | `9.9.9.9` (Quad9) |
| Web UI | `http://192.168.50.103:3000` |
| Homepage | v2.7.0 |
| Config | `/opt/homepage/config/*.yaml` · secrets in `/opt/homepage/.env` |

> **Right-sizing:** I kept the script's default of 4 GB RAM, but Homepage
> uses about 170 MB. It can be lowered to 1024 MB later
> (**CT 103 → Resources → Memory**) without reinstalling.

---

## Part 1: Find the install script

Same source as the last two builds: the Proxmox VE Community Scripts project.
The page lists the config path (`/opt/homepage/config/`), the `.env` file
for allowed hosts, and the default profile.

![Homepage on community-scripts.org](screenshots/01-community-scripts-homepage.png)

Run it from the **Proxmox host shell**:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/homepage.sh)"
```

![Install command in the Proxmox shell](screenshots/02-helper-command-shell.png)

The menu now has an extra option, **App Defaults for Homepage**. That's a
clue: settings saved in my *old* lab were still on the host (see
Challenge #1). I chose **Advanced Install**.

![Advanced Install, with saved App Defaults](screenshots/03-advanced-install-app-defaults.png)

---

## Part 2: Advanced Install

Same approach as Uptime Kuma: every setting chosen on purpose.

### Identity, resources and network

| Setting | Value | Why |
|---|---|---|
| Container type | **Unprivileged** | Root in the container isn't root on the host |
| CT ID | `103` | Matches the last octet of the IP, like 100–102 |
| Hostname | `homepage` | |
| Disk / CPU / RAM | 6 GB / 2 / 4096 MB | Script default (see right-sizing note) |
| Bridge | `vmbr0` (home LAN) | It has to reach every service it displays |
| IPv4 | static `192.168.50.103/24` | |
| Gateway | `192.168.50.1` | The router. Only the container's own IP changes between builds; the gateway stays `.1`. |
| DNS search domain | blank | No local domain until Active Directory |
| DNS server | `9.9.9.9` | Keeps the dashboard loading even if AdGuard is down, so it can *show* AdGuard as down |

![Unprivileged](screenshots/04-container-type-unprivileged.png)
![Container ID 103](screenshots/07-container-id-103.png)
![Hostname](screenshots/08-hostname-homepage.png)
![Disk](screenshots/09-disk-size.png)
![CPU](screenshots/10-cpu-cores.png)
![RAM](screenshots/11-ram-size.png)
![Bridge vmbr0](screenshots/12-network-bridge-vmbr0.png)
![IPv4 static](screenshots/13-ipv4-static.png)
![Static IP](screenshots/14-static-ip.png)
![Gateway](screenshots/15-gateway.png)
![DNS search domain](screenshots/18-dns-search-domain.png)
![DNS server](screenshots/19-dns-server-quad9.png)

> **Why not AdGuard for DNS?** Two valid options here. With Quad9, the
> dashboard keeps working during an AdGuard outage. With AdGuard, Homepage's
> own lookups would show up in AdGuard's query log, which is useful telemetry
> in a SOC lab. Homepage's widgets connect by IP, so I chose the independent
> option, the same as Uptime Kuma.

### Access and capabilities

| Setting | Value | Why |
|---|---|---|
| SSH key source | **none** | Managed through the Proxmox console |
| Root SSH access | **No** | One less login on the network |
| TUN/TAP | **No** | No tunneling needed |
| FUSE, GPU, mknod, extra mounts | **No / none** | Not used |
| Nesting | Yes (script default) | Needed by modern systemd in unprivileged Debian CTs |

![SSH key source: none](screenshots/23-ssh-key-source-none.png)
![Root SSH: No](screenshots/24-ssh-access-no.png)
![TUN/TAP: No](screenshots/26-tun-tap-no.png)

**The TUN/TAP question across the lab:**

| Container | TUN/TAP |
|---|---|
| 100 Tailscale | **Yes** (VPN) |
| 101 AdGuard | No |
| 102 Uptime Kuma | No |
| 103 Homepage | No |

<details>
<summary><b>Show every other wizard screen</b> (all left at default / No)</summary>

| Screen | Choice |
|---|---|
| Root password + verification | set |
| IPv6 | auto |
| MTU | blank (1500) |
| MAC address | blank (auto) |
| VLAN tag | blank (none) |
| Container tags | `community-script;dashboard` |
| FUSE / GPU / mknod | No |
| Nesting | Yes |
| APT cacher / HTTP proxy | No |
| Container timezone | inherited from host (redacted) |
| Container protection | No |
| Mount filesystems | none |
| Post-install hook | none |
| Verbose mode | No |

![Root password](screenshots/05-root-password.png)
![Password verification](screenshots/06-password-verification.png)
![IPv6 auto](screenshots/16-ipv6-auto.png)
![MTU](screenshots/17-mtu-default.png)
![MAC address](screenshots/20-mac-address-auto.png)
![VLAN tag](screenshots/21-vlan-tag-none.png)
![Container tags](screenshots/22-container-tags.png)
![FUSE: No](screenshots/25-fuse-no.png)
![Nesting: Yes](screenshots/27-nesting-yes.png)
![GPU: No](screenshots/28-gpu-passthrough-no.png)
![APT cacher: No](screenshots/29-apt-cacher-no.png)
![HTTP proxy: No](screenshots/30-http-proxy-no.png)
![Timezone](screenshots/31-container-timezone.png)
![Protection: No](screenshots/32-container-protection-no.png)
![mknod: No](screenshots/33-mknod-no.png)
![Mount filesystems](screenshots/34-mount-filesystems-none.png)
![Post-install hook](screenshots/35-post-install-hook-none.png)
![Verbose: No](screenshots/36-verbose-mode-no.png)

</details>

### Confirm, and a stale default caught

![Confirm settings](screenshots/37-confirm-settings.png)

Because defaults from the old lab existed, the script reported
**differences**. **View Diff** showed exactly one: the DNS server.

![App defaults: differences detected](screenshots/38-app-defaults-differences.png)
![The diff: old DNS 192.168.50.24 → new 9.9.9.9](screenshots/39-app-defaults-diff.png)

`192.168.50.24` was my **old AdGuard's IP**, a container that no longer
exists. I chose **Update Defaults**, so any future rebuild uses the
current values.

---

## Part 3: Install, then updates

![Install complete](screenshots/40-install-complete.png)

Homepage 2.7.0 installed on Node.js 24. The script also updated the
container's OS during install.

Next I ran updates, but **in the Proxmox host shell**, the same window the
install script used, not inside CT 103. So this updated the **Proxmox host
itself**: 48 packages, including a new kernel and `pve-manager` (PVE 9.2.20
→ 9.2.24).

![apt update && apt upgrade on the Proxmox host](screenshots/41-host-apt-upgrade.png)
![Upgrade done, host reboot](screenshots/42-host-upgrade-reboot.png)

Updating the host wasn't wrong, and the new kernel needed a reboot to take
effect, so rebooting was the right call. What I learned:

| Where | Correct command |
|---|---|
| **Proxmox host** | `apt update && apt full-upgrade`. Proxmox warns that plain `apt upgrade` can leave a host partially upgraded, because PVE is a rolling release. |
| **Inside a container** (CT console) | `apt update && apt upgrade -y` |
| **The app itself** | `update` in the container's console (community-scripts helper) |

Rebooting the host also restarted **every** container. All of them were
running again afterwards (the Proxmox tile shows 4/4 LXC), but it's worth
confirming **Options → Start at boot = Yes** on each one.

---

## Part 4: First look

The default dashboard: placeholder groups, plus Reddit and YouTube bookmarks.

![Default dashboard](screenshots/43-default-dashboard.png)

---

## Part 5: A read-only Proxmox API token

The Proxmox widget needs API access. **I didn't give it my root password.**
Instead I created a token that can only *read*.

**Datacenter → Permissions → API Tokens → Add**

![API Tokens page](screenshots/44-api-tokens-page.png)

User `root@pam`, Token ID `homepage`, **Privilege Separation ✅**. With
privilege separation on, the token gets *only* the permissions I assign to
it, not everything root can do.

![Add token](screenshots/45-add-token.png)

The secret is shown **once**. I copied it straight into the `.env` file
(Part 6).

![Token secret (redacted)](screenshots/46-token-secret.png)

Then I gave it a role. **Permissions → Add → API Token Permission**:
Path `/`, Token `root@pam!homepage`, Role **PVEAuditor**.

![Add API token permission](screenshots/47-add-api-token-permission.png)
![PVEAuditor on /](screenshots/48-pveauditor-role.png)

**PVEAuditor** can view status and stats but **can't change anything**: no
start/stop, no config edits, no deletes. If this token ever leaked, the worst
case is someone reading my CPU usage.

> **Uptime Kuma status page:** the Uptime Kuma widget reads a *status page*,
> not the admin API. In Uptime Kuma: **Status Pages → New Status Page**, slug
> **`lab`**, add all monitors, Save.

---

## Part 6: Configuration

Back up the defaults first:

```bash
cp -r /opt/homepage/config /opt/homepage/config.bak
```

![Backup](screenshots/49-backup-config.png)

Then replace each file: empty it, paste the new version in `nano`, and
save.

```bash
truncate -s 0 /opt/homepage/config/services.yaml
nano /opt/homepage/config/services.yaml
```

![Empty the file](screenshots/50-truncate-services.png)

| File | What it controls | Final version |
|---|---|---|
| `services.yaml` | The tiles and their live widgets | [config/services.yaml](config/services.yaml) |
| `widgets.yaml` | Top bar: greeting, date/time, weather, search | [config/widgets.yaml](config/widgets.yaml) |
| `bookmarks.yaml` | Bottom link groups | [config/bookmarks.yaml](config/bookmarks.yaml) |
| `settings.yaml` | Title, layout, background, look | [config/settings.yaml](config/settings.yaml) |
| `.env` | **Secrets only** (never committed) | [config/env.example](config/env.example) |

![services.yaml (first version)](screenshots/51-services-yaml-v1.png)
![bookmarks.yaml](screenshots/52-bookmarks-yaml.png)
![widgets.yaml](screenshots/53-widgets-yaml.png)

### Keeping secrets out of the config

The YAML files never contain a password. They use placeholders:

```yaml
username: "{{HOMEPAGE_VAR_PROXMOX_TOKEN_ID}}"
password: "{{HOMEPAGE_VAR_PROXMOX_TOKEN_SECRET}}"
```

Homepage fills them in at startup from `/opt/homepage/.env`. The existing
`HOMEPAGE_ALLOWED_HOSTS` line stays as it is: it's Homepage's built-in
protection against requests using unexpected host names.

![The .env file before](screenshots/54-env-before.png)
![Secrets added at the end (redacted)](screenshots/55-env-secrets-added.png)

```
HOMEPAGE_VAR_PROXMOX_TOKEN_ID=root@pam!homepage
HOMEPAGE_VAR_PROXMOX_TOKEN_SECRET=<token secret>
HOMEPAGE_VAR_ADGUARD_USER=<adguard user>
HOMEPAGE_VAR_ADGUARD_PASS='<adguard password>'   # single quotes if it has $ or #
HOMEPAGE_VAR_LATITUDE=29.76      # city-center coordinates, not my home
HOMEPAGE_VAR_LONGITUDE=-95.37
```

Because of this split, **the YAML files can go on GitHub as-is**. The
`config/` folder in this repo is the real, running config.

Then I locked the file down. The service runs as `root`, so only root needs
to read it:

```bash
systemctl show -p User homepage     # → User=root
chmod 600 /opt/homepage/.env
systemctl restart homepage
systemctl status homepage --no-pager
```

![chmod 600](screenshots/56-chmod-600-env.png)
![Service running](screenshots/57-service-running.png)

### Why the Tailscale tile has no live stats
The Tailscale widget needs an API key with full control of the tailnet, and
it expires within 90 days. Storing that on a dashboard isn't worth a few
numbers, so the tile just shows a ping status dot.

---

## Part 7: Layout and look

The first layout had big half-empty tiles: Router and Tailscale have no
stats, but they shared rows with tiles that do. I regrouped them:

```
Lab Core:         [ Proxmox ]  [ AdGuard ]  [ Uptime Kuma ]        ← live stats
Network & Links:  [ Router ] [ Tailscale ] [ Portfolio ] [ Homelab ] ← status dots
```

![services.yaml, regrouped](screenshots/67-services-yaml-v2.png)

**Background and "frosted glass" tiles:** a personal wallpaper sits behind
the dashboard (not shown in these screenshots), dimmed with `opacity: 40`.
`cardBlur: md` blurs it behind each tile. Homepage can't combine card blur
with the background's own blur/brightness filters, so those are removed and
opacity does the dimming.

![settings.yaml, final](screenshots/68-settings-yaml-final.png)

---

## Challenges & Fixes

### 1. Stale defaults survived the lab wipe
I wiped the containers but not the Proxmox host. The helper script's saved
defaults live **on the host** (`/usr/local/community-scripts/defaults/`), so
they survived, still pointing DNS at `192.168.50.24`, an address that no
longer exists. A future rebuild using them would have had no DNS and failed
to download anything. **View Diff → Update Defaults** fixed it.
**Lesson:** when you rebuild, check what the old environment left behind.

### 2. AdGuard widget: a generic "HTTP Error"

![AdGuard tile: API Error](screenshots/58-adguard-api-error.png)

The tile gave no detail, just "see logs". The cause turned out to be
**a wrong AdGuard password** in `.env`. Two ways to find that quickly: the
Homepage log (`grep -i adguard /opt/homepage/config/logs/homepage.log`), or
a `curl` with the same credentials against AdGuard's API:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -u 'USER:PASS' http://192.168.50.101/control/status
# 200 = login OK, 401 = wrong username/password
```

**Watch out:** Homepage retries every few seconds, and AdGuard has
brute-force protection, so it can **lock out** the dashboard's IP. Fix the
credentials first, *then* restart AdGuard (`systemctl restart AdGuardHome`
in CT 101) to clear any block, then restart Homepage.

![AdGuard stats working](screenshots/59-adguard-fixed.png)

**Lesson:** generic UI errors hide the cause. The logs and a direct API test
show it.

### 3. `Widgets.yaml` ≠ `widgets.yaml`

![nano Widgets.yaml](screenshots/60-widgets-capital-w.png)

I typed the file name with a capital **W**. Linux file names are
**case-sensitive**, so nano created a brand-new file that Homepage never
reads, and the top bar didn't change. Editing `widgets.yaml` (lowercase)
fixed it. Clean up the stray copy with `rm /opt/homepage/config/Widgets.yaml`.

### 4. A fixed theme hid the night-mode switch

![settings.yaml with theme: dark](screenshots/61-settings-fixed-theme.png)

`theme: dark` and `color: slate` look fine, but a fixed theme **removes** the
light/dark toggle and the color picker, and `hideVersion` hid the version.
I removed all three. The black GitHub logo was also invisible on the dark
background, so I switched it and Tailscale to white Simple Icons
(`si-github-#ffffff`).

```bash
sed -i '/^theme: dark$/d; /^color: slate$/d; /^hideVersion: true$/d' /opt/homepage/config/settings.yaml
sed -i 's/icon: github.svg/icon: si-github-#ffffff/; s/icon: tailscale.svg/icon: si-tailscale-#ffffff/' \
  /opt/homepage/config/services.yaml /opt/homepage/config/bookmarks.yaml
```

![The fix](screenshots/62-remove-theme-white-icons.png)

### 5. The background that was really a web page
First attempt: the Reddit post URL, which returned **403 Forbidden**. Second
attempt: the Imgur *page* URL. That download "worked" but saved only
7.4 KB.

![curl: 403, then a 7.4 KB file](screenshots/65-curl-403-and-html.png)

Checking the file's first bytes (its **magic bytes**) showed why:

```bash
head -c 4 background.jpg | od -An -tx1
#  3c 21 64 6f   →  "<!do", the start of an HTML page, not a JPEG
```

![Magic bytes: HTML, not an image](screenshots/66-magic-bytes-html.png)

A real JPEG starts with `ff d8 ff`, a PNG with `89 50 4e 47`. The fix was
the **direct image link** (`i.imgur.com/<id>.jpg`), not the page that
displays it. The file extension said `.jpg`, but the content was HTML.
Checking file signatures instead of trusting names is the same habit used
in malware triage.

There was also a self-inflicted step: earlier, I had commented out the whole
`background:` block while the picture wasn't ready, then forgot to undo it.

![Commenting out the background block](screenshots/63-comment-out-background.png)
![Background block still commented](screenshots/64-background-commented.png)

### 6. Ran updates on the host instead of the container
Covered in Part 3: the prompt said `root@proxmox`, not `root@homepage`.
**Always read the prompt before running a command.** It's the difference
between updating one container and rebooting the whole lab.

---

## Security notes & known gaps

| Done | Detail |
|---|---|
| ✅ Least-privilege API token | Read-only `PVEAuditor`, privilege separation on |
| ✅ Secrets out of config | `.env` only, `chmod 600`, never committed |
| ✅ Host-header allowlist | `HOMEPAGE_ALLOWED_HOSTS` limited to `localhost:3000` and `192.168.50.103:3000` |
| ✅ No inbound exposure | LAN only, plus Tailscale for remote access |

| Known gap | Plan |
|---|---|
| **No login on the dashboard yet** | Homepage v2 has optional built-in authentication (`HOMEPAGE_AUTH_ENABLED`, `HOMEPAGE_AUTH_SECRET`, `HOMEPAGE_AUTH_PASSWORD` in `.env`). Next step: enable it with a freshly generated secret (`openssl rand -base64 32`). |
| **Plain HTTP** | Any login would cross the LAN unencrypted. Nginx Proxy Manager (next phase) adds HTTPS. |
| **Homepage isn't monitored yet** | Add an Uptime Kuma HTTP monitor for `http://192.168.50.103:3000` |

---

## Lessons learned

1. **Least privilege applies to API access too.** A dashboard that only
   *reads* gets a token that can only read.
2. **Keep secrets and config apart.** Config can be shared and versioned;
   secrets can't. Placeholders plus `.env` give you both.
3. **Logs over UI errors.** "HTTP Error" was useless; one log line and one
   `curl` found the cause.
4. **Trust content, not names.** A `.jpg` that starts with `<!do` is a web
   page. Magic bytes don't lie.
5. **Linux is case-sensitive, and the prompt tells you where you are.** Two
   small mistakes, two easy fixes, both worth remembering.
6. **Rebuilds inherit leftovers.** Old defaults on the host still pointed at
   an IP that no longer existed.

---

*Screenshots are redacted: the Proxmox token ID and secret, the AdGuard
credentials, Homepage's pre-generated auth secret, and the timezone are
covered. My personal wallpaper is left out of the final screenshots. The
lab's `192.168.50.x` addresses are left visible. They're private (RFC 1918)
and not reachable from the internet. Image metadata is stripped.*

**Phase 1 (core services) is complete.** *Next: Nginx Proxy Manager +
Vaultwarden (HTTPS and a password manager), then OPNsense and the segmented
lab networks.*
