# Tailscale Subnet Router + Exit Node on Proxmox (LXC)

[← Back to lab overview](../README.md)

Part of my **Home SOC Lab v2** rebuild. This component gives me secure remote
access to the whole lab from anywhere, **without opening a single port on my
router**. Tailscale runs in a lightweight Debian LXC container that acts as
both a **subnet router** (reach internal lab IPs by address) and an
**exit node** (route my traffic through home on untrusted Wi-Fi).

> **Why it's the first thing I rebuilt:** every later step (firewall, SIEM, AD)
> is easier when I can manage the host remotely. It also keeps zero inbound
> ports exposed to the internet, so the attack surface stays closed.

**What this shows:** Proxmox LXC provisioning · Linux administration ·
VPN / zero-trust remote access · least-privilege container design ·
troubleshooting from error output and vendor docs

---

## Goal

| Item | Value |
|---|---|
| Container | Unprivileged LXC, Debian 12 (Bookworm) |
| CT ID / hostname | `100` / `TailScale` |
| Resources | 2 vCPU · 512 MB RAM · 512 MB swap · 8 GB disk (`local-lvm`) |
| Network | `eth0` on `vmbr0`, static **192.168.50.100/24**, gw 192.168.50.1 |
| Tailscale role | Subnet router (`192.168.50.0/24`) + exit node |
| Tailscale SSH | Enabled |
| Key expiry | Disabled (node won't drop offline; see the trade-off note below) |
| Proxmox VE | 9.2 |

**Architecture position:** the Tailscale CT sits on the home LAN
(`192.168.50.0/24`) alongside the other service containers. It's the remote
way in to the lab. The OPNsense-segmented lab networks will sit behind it
later in the rebuild.

---

## Part 1: Download the Debian 12 template

Proxmox → `local` storage → **CT Templates** → **Templates**, search `deb`,
pick **debian-12-standard**, then **Download**.

![Open the template downloader](screenshots/01-download-template-button.png)
![Select Debian 12 Bookworm](screenshots/02-select-debian12-template.png)
![Template download complete: TASK OK](screenshots/03-template-download-complete.png)

> I deliberately used **Debian 12**, not 13. It's the mature, widely
> documented base for Tailscale-in-LXC setups, and this is a set-and-forget
> appliance. Newest isn't the point.

---

## Part 2: Create the container

Top-right **Create CT**, then step through the wizard.

![Create CT button](screenshots/04-create-ct-button.png)

**General**: CT ID `100`, hostname `TailScale`, **Unprivileged** ✅,
**Nesting** ✅, set a root password.

![General tab](screenshots/05-ct-general.png)

**Template**: the Debian 12 image from Part 1.

![Template tab](screenshots/06-ct-template.png)

**Disks**: 8 GB on `local-lvm`.

![Disks tab](screenshots/07-ct-disks.png)

**CPU**: 2 cores.

![CPU tab](screenshots/08-ct-cpu.png)

**Memory**: 512 MB RAM + 512 MB swap (plenty for Tailscale).

![Memory tab](screenshots/09-ct-memory.png)

**Network**: `eth0` on `vmbr0`, static **192.168.50.100/24**, gateway
`192.168.50.1`. A static IP matters here: this node is a router, so its
address shouldn't move.

![Network tab](screenshots/10-ct-network.png)

**Confirm** → **Finish**.

![Confirm tab](screenshots/11-ct-confirm.png)
![CT 100 created: TASK OK](screenshots/12-ct-create-task-ok.png)

---

## Part 3: Install Tailscale

Start the CT and open its **Console**. First login is `root` + the password
you set.

![First console login](screenshots/13-first-console-login.png)

Update the base image and install `curl` (the minimal template doesn't ship
with it):

```bash
apt update && apt upgrade -y
apt install curl -y
```

![apt update && upgrade](screenshots/14-apt-update-upgrade.png)

> The screenshot shows `apt upgrade all`. That's a typo on my part:
> `apt upgrade` takes no `all` argument. Use the command in the code block.

![apt install curl](screenshots/15-apt-install-curl.png)

Install Tailscale with the official one-liner from
`tailscale.com/download/linux`:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

![Install command on the Tailscale site](screenshots/16-tailscale-install-command.png)
![Running the install script](screenshots/17-run-install-script.png)
![Install complete](screenshots/18-install-complete.png)

Bring it up with Tailscale SSH enabled:

```bash
tailscale up --ssh
```

It prints a one-time auth URL. Open it, sign in, approve the device.

![Authentication link](screenshots/19-auth-link.png)
![Tailscale login](screenshots/20-login-page.png)
![Connect device](screenshots/21-connect-device.png)
![Login successful](screenshots/22-login-successful.png)

The node now shows up in the admin console as connected.

![Machine connected in the admin console](screenshots/23-machines-connected.png)

---

## Part 4: Turn it into a subnet router + exit node

This is where the real work (and all the troubleshooting) happened. See
**Challenges & Fixes** below for the full story. The short version:

1. Enable IP forwarding inside the container.
2. Give the unprivileged LXC access to `/dev/net/tun` (host-side).
3. Re-run `tailscale up` advertising the route and exit node.

Final working command (inside the CT):

```bash
tailscale up --accept-dns=false --advertise-exit-node \
  --advertise-routes=192.168.50.0/24 --ssh
```

- `--advertise-routes=192.168.50.0/24`: offer the home LAN to the tailnet
- `--advertise-exit-node`: offer this node as an internet exit
- `--accept-dns=false`: keep the container on the LAN's DNS instead of
  letting Tailscale manage its resolver
- `--ssh`: keep Tailscale SSH on (it was already set; see Challenge #2)

![Advertise command succeeds; tailscale status](screenshots/33-advertise-success-status.png)

Then in the **admin console → the node → ⋯ → Edit route settings**, enable the
subnet route and the exit node. These options only appear *after* the node
advertises them.

![Open Edit route settings](screenshots/34-open-route-settings.png)
![Enable subnet route + exit node](screenshots/35-enable-routes-exitnode.png)

Finally, **⋯ → Disable key expiry** so the node never silently drops off the
tailnet. The **Expiry disabled** badge on the node (visible above) confirms it.

> **Security trade-off:** with key expiry disabled, a stolen node key stays
> valid until I revoke it. I accept that for a server I physically control,
> and I can remove the node from the admin console at any time. I wouldn't
> disable expiry on a laptop or phone.

---

## Challenges & Fixes

The clean walkthrough above hides four real problems I hit. These are the
useful part.

### 1. Admin console: "This machine does not expose any routes" / exit node greyed out

![Routes not exposed](screenshots/24-routes-not-exposed.png)

**Cause:** I first ran `tailscale up --ssh` with no advertise flags. The admin
console can only toggle what the node actually *offers*, and it was offering
nothing, so the checkbox and exit-node toggle were disabled.

**Fix:** advertise the route and exit node from the CLI first (see #2–#4),
then the toggles appear in the console.

---

### 2. `tailscale up` error: "requires mentioning all non-default flags"

![Must mention all non-default flags](screenshots/25-advertise-flags-error.png)

**Cause:** `--ssh` was already set from the first run. Re-running
`tailscale up` to add routes requires restating *every* non-default flag.
Otherwise Tailscale refuses, so that it doesn't silently drop a setting.

**Fix:** put them all in one command (Tailscale even prints the exact line):

```bash
tailscale up --accept-dns=false --advertise-exit-node \
  --advertise-routes=192.168.50.0/24 --ssh
```

---

### 3. Warning: "IP forwarding is disabled, subnet routing/exit nodes will not work"

That same output also warned that IP forwarding was off.

**Cause:** a subnet router / exit node has to forward packets between
interfaces, which the kernel blocks by default.

**Fix (inside the CT):** enable IPv4 + IPv6 forwarding, then reload.

```bash
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
```

> **Gotcha within the gotcha:** I first pasted Tailscale's documented
> commands, which use `sudo` and `/etc/sysctl.d/99-tailscale.conf`. The
> minimal Debian 12 LXC has **no `sudo`**. The console session is already
> `root`, so `sudo` returns `command not found`. Dropping `sudo` fixes it
> (either file path works).

![IP forwarding enabled (no sudo needed)](screenshots/26-ip-forwarding-fix.png)

---

### 4. The big one: unprivileged LXC has no `/dev/net/tun`

Tailscale needs a TUN device to route traffic in kernel mode, which subnet
routing and exit nodes depend on. An **unprivileged** container doesn't get
`/dev/net/tun` by default. Proxmox *helper-script* containers add it
automatically. Building the CT by hand does not, which is why I hit it.

I confirmed the fix against Tailscale's own "Tailscale in LXC containers" doc:

![Searching the Tailscale LXC docs](screenshots/27-tun-docs-search.png)
![Tailscale docs: Proxmox TUN access](screenshots/28-tun-docs-proxmox.png)

**Fix (on the Proxmox HOST, not inside the CT):** edit the container config and
add the TUN device.

```bash
vi /etc/pve/lxc/100.conf
```

![Editing /etc/pve/lxc/100.conf](screenshots/29-edit-conf-ctid.png)
![Config before the change](screenshots/30-conf-before.png)

Add these two lines:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

![Config after adding the TUN lines](screenshots/31-conf-after-tun-lines.png)

Restart the container so the change takes effect:

```bash
pct stop 100 && pct start 100
```

![Starting the CT](screenshots/32-start-ct.png)

To check it worked, `ls -l /dev/net/tun` inside the CT should show a
character device `10, 200`.

> **Easier on newer Proxmox:** I used the config-file method from the docs'
> Proxmox 7 section, and it still works on 9.2. Current Proxmox can do the
> same thing from the GUI (**CT → Resources → Add → Device Passthrough →
> `/dev/net/tun`**) or with `pct set 100 --dev0 /dev/net/tun`. The GUI
> method is what I'd use next time.

---

### Minor: UDP GRO forwarding warning

`tailscale up` also warns that UDP GRO forwarding is suboptimally configured on
`eth0`. It's a **throughput** note, not a failure: subnet routing and the exit
node work without it. Tailscale's performance docs fix it with:

```bash
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
ethtool -K $NETDEV rx-udp-gro-forwarding on rx-gro-list off
```

That setting doesn't persist across reboots by default, and I haven't tested
whether an unprivileged LXC is allowed to change it. Left as-is for now.

---

## Proof it works

- `tailscale status` shows the node connected, with the old (wiped-lab) nodes
  correctly listed as offline.
- Admin console shows the node **Connected**, with **Expiry disabled**, the
  `192.168.50.0/24` route **approved**, and **Use as exit node** enabled.
- From a phone/laptop on the tailnet I can reach `192.168.50.x` lab services by
  IP, and switch my internet egress through home via the exit node.

![Status + advertise success](screenshots/33-advertise-success-status.png)

---

## Lessons learned

1. **Build-by-hand ≠ helper-script.** The helper scripts quietly fix TUN and
   forwarding. Doing it manually taught me *why* those settings exist.
2. **Unprivileged containers trade convenience for safety.** The TUN device
   and IP forwarding both had to be granted explicitly, which is exactly the
   least-privilege behavior you want.
3. **`tailscale up` is declarative.** It won't let you change one setting
   without restating the rest, which prevents silent misconfiguration.
4. **Read the error output.** Every fix here came from a message Tailscale
   printed, or from its own docs, not from guessing.
5. **Zero inbound ports.** Remote access to the entire lab, nothing exposed to
   the internet. This is the model I'd push for any small business too.

---

*Screenshots are redacted: tailnet name, Tailscale (100.x) addresses, account
emails, other devices on the tailnet, the one-time auth link, the container's
MAC address and SSH host fingerprints are covered. The lab's `192.168.50.x`
addresses are left visible. They're private (RFC 1918) and not reachable from
the internet.*

*Next in the build: [AdGuard Home](../02-adguard-home/) (DNS + ad blocking), then the
rest of the home-LAN service containers, then OPNsense and the segmented lab networks.*
