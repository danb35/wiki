---
title: Add iMessage Bridge to Synapse
description: Installing the Mautrix iMessage bridge to Synapse
published: true
date: 2026-09-04T18:47:34.317Z
tags: 
editor: markdown
dateCreated: 2023-01-07T01:30:02.650Z
---

# Setting Up an iMessage↔Matrix Bridge

Bridges iMessage to Matrix using [mautrix/imessage](https://github.com/mautrix/imessage) with the BlueBubbles connector, which offloads the actual iMessage-hooking work to [BlueBubbles Server](https://bluebubbles.app/) rather than depending on private-framework tricks maintained by the bridge project itself.

## Architecture: two hosts, not necessarily one

This setup has two logically separate roles:

- **The BlueBubbles host** — must run macOS, since it runs Messages.app and hooks into it directly. This is the one piece that can't be avoided.
- **The bridge host** — runs `mautrix-imessage` and `mautrix-wsproxy`. These are plain Go binaries that talk to BlueBubbles over its local REST API; they have no macOS-specific dependency when using the `bluebubbles` platform. This can be the *same* Mac as the BlueBubbles host (simplest — everything talks over `localhost`), or a completely separate Linux, Windows, or macOS machine, as long as it can reach the BlueBubbles host's REST API (default port 1234) over the network.

The rest of this guide assumes the simplest case — both roles on the same Mac — but calls out where it matters if you split them.

## Prerequisites

### The BlueBubbles host

- A Mac — physical hardware or a VM — dedicated to this role. It needs to stay powered on continuously and be signed into Messages.app with the Apple ID / iCloud account whose iMessages you want to bridge.
- System Integrity Protection (SIP) disabled, **only if** you want reactions/tapbacks and other rich features via BlueBubbles' Private API. A stock Mac with SIP enabled can still bridge basic text messages through BlueBubbles without Private API. AMFI does **not** need to be disabled — that was a requirement of an older, now-unmaintained connector (Barcelona), not of BlueBubbles.

  **To disable SIP** (requires physical access — it cannot be done remotely or from a normal login session):
  1. Restart the Mac and enter Recovery Mode:
     - **Apple Silicon:** hold the power button as the Mac starts up until "Loading startup options" appears, then choose **Options**.
     - **Intel:** hold **Cmd+R** immediately at startup.
  2. From the menu bar in Recovery Mode, open **Utilities → Terminal**.
  3. Run:
     ```bash
     csrutil disable
     ```
  4. On Apple Silicon, if prompted by Startup Security Utility, you may also need to select a reduced security level — this is a separate Apple Silicon boot-security gate layered on top of SIP.
  5. Restart normally, then verify:
     ```bash
     csrutil status
     ```
     should report SIP as disabled.

### The bridge host

(If combining with the BlueBubbles host, this is the same machine — the requirements below just add to the ones above.)

- Xcode Command Line Tools if on macOS (`xcode-select --install`), or a C toolchain (`build-essential` on Debian/Ubuntu, etc.) if on Linux — needed to compile with cgo.
- Go 1.22 or newer. No package manager required; download directly from [go.dev/dl](https://go.dev/dl/).
- SSH enabled for remote administration — optional, but strongly recommended on macOS, since without it you'll need physical or screen-sharing access for the GUI steps in this guide. (Enable via System Settings → General → Sharing → Remote Login.)

### The Matrix homeserver

- A Synapse (or other appservice-capable homeserver) you administer.
- A network path between the homeserver and the bridge host, in both directions:
  - The homeserver needs to reach the bridge's `wsproxy` component to deliver events.
  - The bridge needs outbound access to the homeserver's client API.
  - If both are on the same network, this is trivial. If not (e.g. a cloud-hosted homeserver and a bridge host at home), you'll need a tunnel. An SSH reverse tunnel (documented below) is simple and has no dependency on third-party infrastructure. A mesh VPN (ZeroTier, Tailscale, etc.) is an alternative, but is another moving part that can fail independently — see Changelog.

## Setting up the BlueBubbles host

Download from [bluebubbles.app/downloads/server](https://bluebubbles.app/downloads/server/) and install to `/Applications`.

Launch it and walk through setup. Skip the Google/Firebase account step — it's not needed, since the Matrix bridge talks to BlueBubbles' local REST API directly rather than through Firebase push.

Grant Full Disk Access when prompted.

Set a server password (Settings → Server Address / Password) and note it down — you'll need it for the bridge config below.

If you disabled SIP, enable Private API in Settings → Private API to get reactions/tapbacks.

## Setting up the bridge host

### 1. Build libolm

GitHub's `matrix-org/olm` mirror is a stub (kept empty for US export-control reasons around cryptography). Get the real source from GitLab:

```bash
git clone https://gitlab.matrix.org/matrix-org/olm.git
cd olm && mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$HOME/local \
  -DBUILD_SHARED_LIBS=OFF -DCMAKE_POLICY_VERSION_MINIMUM=3.5
cmake --build . -j4
cmake --install .
```

If you don't have `cmake`, a portable build works fine — download one from [cmake.org/download](https://cmake.org/download/), no system-wide install needed. This works the same way on Linux as on macOS.

### 2. Build mautrix-imessage

```bash
git clone https://github.com/mautrix/imessage.git
cd imessage
CGO_CFLAGS="-I$HOME/local/include" CGO_LDFLAGS="-L$HOME/local/lib" ./build.sh -o mautrix-imessage
```

### 3. Build mautrix-wsproxy

wsproxy relays appservice transactions between the homeserver and the bridge. See [docs.mau.fi/bridges/go/setup.html](https://docs.mau.fi/bridges/go/setup.html) for its source and build instructions.

## Configuring the bridge

- Copy `example-config.yaml` to `config.yaml`.
- Under `imessage:`, set:
  ```yaml
  platform: bluebubbles
  bluebubbles_url: http://<blueBubbles-host-address>:1234
  bluebubbles_password: <the password you set in BlueBubbles Server>
  ```
  (`<bluebubbles-host-address>` is `localhost` if the bridge and BlueBubbles are on the same machine.)
- Under `homeserver:`, set `address` to your homeserver's public URL and `domain` to its server name.
- Generate the appservice registration:
  ```bash
  ./mautrix-imessage --config config.yaml --generate-registration
  ```

## Connecting the bridge to your homeserver

### Registering the appservice

Copy the generated `registration.yaml` to your homeserver and add its path to Synapse's `app_service_config_files` in `homeserver.yaml`, then restart Synapse. If you're using a deployment tool like matrix-docker-ansible-deploy, check whether it has native mautrix-imessage support before wiring this in manually — as of this writing it does not, and the practical approach is to mount the registration file in as an extra argument and reference it directly in `app_service_config_files`.

### Connectivity: SSH reverse tunnel

The homeserver needs to reach `mautrix-wsproxy` (default port 29331) on the bridge host. If they're not on the same network, an SSH reverse tunnel from the bridge host to the homeserver is a simple way to achieve this:

1. On the bridge host, generate a dedicated key restricted to port-forwarding only:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_wsproxy_tunnel -N ""
   ```
2. On the homeserver, add its public key to some low-privilege account's `authorized_keys`, prefixed with:
   ```
   restrict,port-forwarding,permitopen="localhost:29331" ssh-ed25519 AAAA...
   ```
   This key can only forward ports — it can't open a shell, even if it's ever compromised.
3. If your homeserver runs Synapse in a Docker container on an isolated bridge network (common with matrix-docker-ansible-deploy and similar), the container can't reach the host's loopback interface. Bind the tunnel to that Docker network's gateway IP instead of `localhost` — find it with `docker network inspect <network-name>`. This requires adding `GatewayPorts clientspecified` to the homeserver's `sshd_config`.
4. If the homeserver has a restrictive host firewall (common on cloud VPS default images, which often only allow inbound SSH), add an explicit allow rule for the Docker bridge subnet on the wsproxy port.
5. Run the tunnel persistently (see "Running the bridge persistently" below for concrete examples on both macOS and Linux).
6. In your appservice registration's `url:` field, point at wherever the tunnel lands on the homeserver side (the Docker bridge gateway IP + port from step 3, or `localhost` if you're not using Docker).

## Running the bridge persistently

You need `mautrix-imessage` and `mautrix-wsproxy` running continuously, restarting automatically on crash or reboot — plus the SSH tunnel from the previous section, if you're using one. How you do this depends on what the bridge host runs.

### macOS: launchd

Create one plist per process in `~/Library/LaunchAgents/`, then `launchctl load` each.

`~/Library/LaunchAgents/mautrix-imessage.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.github.mautrix-imessage</string>
    <key>WorkingDirectory</key>
    <string>/Users/you/src/mautrix-imessage</string>
    <key>ProgramArguments</key>
    <array>
        <string>./mautrix-imessage</string>
        <string>--config</string>
        <string>config.yaml</string>
    </array>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
</dict>
</plist>
```

`~/Library/LaunchAgents/mautrix-wsproxy.plist` follows the same shape, pointing at the wsproxy binary and its own working directory.

The SSH tunnel's launchd job (label `com.familybrown.wsproxy-tunnel` in the example below — pick your own) uses `/usr/bin/ssh` directly as the program, since there's no long-running binary of your own to wrap:
```xml
<key>ProgramArguments</key>
<array>
  <string>/usr/bin/ssh</string>
  <string>-N</string><string>-R</string>
  <string>&lt;docker-bridge-gateway-ip&gt;:29331:localhost:29331</string>
  <string>-o</string><string>ServerAliveInterval=15</string>
  <string>-o</string><string>ServerAliveCountMax=3</string>
  <string>-o</string><string>ExitOnForwardFailure=yes</string>
  <string>-i</string><string>/Users/you/.ssh/id_ed25519_wsproxy_tunnel</string>
  <string>user@your-homeserver</string>
</array>
<key>KeepAlive</key><true/>
```

After creating or editing a plist:
```bash
launchctl load ~/Library/LaunchAgents/<name>.plist
```
(If you've previously loaded it and are updating it, `launchctl unload` first, or use `launchctl bootout`/`bootstrap` — the newer verbs, more reliable on recent macOS than `load`/`unload` for picking up a changed file.)

### Linux: systemd

If you've split the bridge host onto Linux, use a user or system systemd unit instead. Example for `mautrix-imessage` (`/etc/systemd/system/mautrix-imessage.service` or `~/.config/systemd/user/mautrix-imessage.service`):

```ini
[Unit]
Description=mautrix-imessage bridge
After=network-online.target

[Service]
WorkingDirectory=/home/you/mautrix-imessage
ExecStart=/home/you/mautrix-imessage/mautrix-imessage --config config.yaml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`mautrix-wsproxy` follows the same shape. Enable and start with `systemctl enable --now mautrix-imessage.service` (add `--user` and use `systemctl --user` if using a user unit).

## Verifying it works

- Check BlueBubbles Server's own status: `curl http://localhost:1234/api/v1/server/info?password=<password>` should return `private_api: true` and `helper_connected: true`.
- Tail the bridge's logs for `Startup sync complete` and successful websocket pings.
- Send a message from Matrix and confirm it arrives in iMessage, and vice versa.

## Troubleshooting

- **Contact lookups failing, or reactions not bridging:** confirm BlueBubbles' Private API is enabled and SIP is actually disabled (`csrutil status`).
- **Bridge can't reach the homeserver:** check that the tunnel/VPN link is actually up before assuming it's a bridge config problem.
- **Homeserver can't reach the bridge:** check the homeserver's own firewall — cloud VPS images often default to allowing only SSH inbound, which silently blocks a newly-added listener.

## Changelog

- **2026-09-04:** Migrated from the Barcelona connector to BlueBubbles. Barcelona had gone unmaintained (no releases since Aug 2023) and had a live regression in its contacts API; BlueBubbles is actively maintained and decouples the iMessage-hooking problem from the Matrix-bridging problem. AMFI no longer needs to be disabled, since that was specific to Barcelona.
- **2026-09-04:** Replaced a ZeroTier mesh VPN link between the bridge host and homeserver with an SSH reverse tunnel, after the ZeroTier link failed and could not be restored — the two nodes stopped negotiating any peer path despite both showing authorized/active in ZeroTier Central and running matching client versions. Root cause was never identified, likely a home-router NAT issue. The SSH tunnel has fewer moving parts and no dependency on third-party relay infrastructure.
