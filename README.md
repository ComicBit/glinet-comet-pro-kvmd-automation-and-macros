# GL.iNet Comet Pro KVMD Automation Guide (API, ATX, WebTerm, SSH, Macros) 🧠🔌

This guide is a practical, copy-pasteable walkthrough for automating a **GL.iNet Comet Pro** (KVMD-based KVM). It covers:

* KVMD **HTTPS API** for HID (keyboard and mouse), text typing, and ATX power control
* The **nginx rewrites** needed to expose missing `/api/...` endpoints
* **SSH access** (Dropbear) for reliable “run macro script” automation
* The built-in **Web Terminal** (“webterm”, `ttyd`) and how it’s wired

The goal: trigger robust multi-step KVM macros from **iOS Shortcuts** or any client that can run an SSH command.

---

## What this setup gives you ✅

You can automate things like:

* Type text into the target machine (BIOS, OS, login prompts)
* Press single keys, shortcuts, sequences, or spam keys (example: `Delete` during reboot)
* Move/click mouse (absolute/relative/wheel)
* Press ATX buttons (power, long power press, reset) through the Comet Pro ATX board
* Run “macros” from iOS Shortcuts by SSH-ing into the Comet Pro and executing scripts

---

## Architecture overview 🧩

The Comet Pro runs:

* **KVMD** (backend API)
* **nginx** (frontend reverse proxy)
* Several services wired via **UNIX sockets**, typically:

  * KVMD backend: `/run/kvmd/kvmd.sock`
  * Web terminal (`ttyd`): `/run/kvmd/ttyd.sock`

nginx exposes “friendly” `/api/...` routes and rewrites them to KVMD internal routes like `/hid/...`.

---

## Security warnings 🔒

* Do **not** expose the Comet Pro directly to the public internet.
* Prefer LAN-only access or a VPN (Tailscale, WireGuard).
* Use strong credentials.
* Prefer key-based SSH auth for admin machines.

---

## 1) Authentication basics (important) ✅

### KVMD API credentials

For the Comet Pro KVMD API, Basic Auth uses:

* **Username:** `admin`
* **Password:** the **same UI password** you set during the device’s first-time setup

In examples below we’ll refer to these as:

* `KVM_USER=admin`
* `KVM_PASS=<your-ui-password>`

### TLS / self-signed certs

The Comet Pro typically uses a self-signed TLS cert, so for `curl` you will usually need:

* `-k` / `--insecure`

---

## 2) Access methods: Web UI, WebTerm, SSH 🧭

### Web UI

Open the Comet Pro in a browser, login, and confirm:

* video capture works
* keyboard works
* ATX buttons work

### Web terminal (“webterm”)

The web terminal is an **embedded `ttyd`** instance proxied by nginx. It’s useful for debugging, but not the best automation primitive.

Relevant wiring (on the device):

* nginx server location: `/usr/share/kvmd/extras/webterm/nginx.ctx-server.conf`
* nginx upstream: `/usr/share/kvmd/extras/webterm/nginx.ctx-http.conf`
* upstream socket: `/run/kvmd/ttyd.sock`

This typically means a websocket endpoint under:

* `wss://<host>/extras/webterm/ttyd/ws`

### SSH (recommended for automation)

Comet Pro commonly runs **Dropbear** SSH server on port 22.

SSH is the cleanest way to run macros:

* iOS Shortcuts can run a single SSH command
* you avoid handling cookies/websockets/terminal state

---

## 3) Fixing SSH so automation is sane 🔑

### 3.1 Set a known root password

If you don’t know the current root password, use WebTerm to set it:

```sh
passwd
```

Use a strong password.

### 3.2 Add SSH keys (recommended)

On your Mac/Linux:

```sh
ssh-keygen -t ed25519 -C "comet-pro"
```

Copy to the Comet Pro:

```sh
ssh root@<host> "mkdir -p /root/.ssh && chmod 700 /root/.ssh"
scp ~/.ssh/id_ed25519.pub root@<host>:/root/.ssh/authorized_keys
ssh root@<host> "chmod 600 /root/.ssh/authorized_keys"
```

If iOS Shortcuts is your main client, you might still use password auth there, but keys are strongly recommended for your admin machines.

---

## 4) KVMD HID API (keyboard, typing, mouse) ⌨️🖱️

### Critical gotcha: many HID endpoints read from URL query params

On this Comet Pro build, handlers like `send_key` read `req.query.get("key")`.

That means **JSON body won’t work**, and you’ll get errors like:

* `None argument is not a valid Keyboard key`

✅ Working format (query string):

```sh
curl -k -u "admin:<UI_PASSWORD>" -X POST \
  "https://<host>/api/hid/events/send_key?key=ArrowUp&state=1&finish=1"
```

---

### 4.1 Type text

**Endpoint:**

* `POST /api/hid/print`

**Body:** raw text

Example:

```sh
curl -k -u "admin:<UI_PASSWORD>" -X POST "https://<host>/api/hid/print" \
  --data-binary "cmd"
```

---

### 4.2 Send a single key event

**Endpoint:**

* `POST /api/hid/events/send_key`

**Query params:**

* `key=<KeyboardEvent.code>`
* `state=1` (press) or `state=0` (release)
* `finish=1` (optional, helps finalize sequences)

Tap a key (press then release):

```sh
curl -k -u "admin:<UI_PASSWORD>" -X POST \
  "https://<host>/api/hid/events/send_key?key=Enter&state=1"

curl -k -u "admin:<UI_PASSWORD>" -X POST \
  "https://<host>/api/hid/events/send_key?key=Enter&state=0&finish=1"
```

---

### 4.3 Send a shortcut (optional)

If you exposed `send_shortcut` via nginx (see below), you can call:

* `POST /api/hid/events/send_shortcut?keys=...`

Safest form is repeating params:

```sh
curl -k -u "admin:<UI_PASSWORD>" -X POST \
  "https://<host>/api/hid/events/send_shortcut?keys=ControlLeft&keys=AltLeft&keys=Delete"
```

---

### 4.4 Mouse (overview)

KVMD supports mouse endpoints (button, absolute move, relative move, wheel). Names can vary by build. The practical way is to inspect the server or web UI requests and expose those endpoints the same way you exposed HID keys.

---

## 5) Supported keys legend (Comet Pro build) 🎛️

This is the **full list of supported keys** dumped from the device:

```text
Total keys: 115
KeyA
KeyB
KeyC
KeyD
KeyE
KeyF
KeyG
KeyH
KeyI
KeyJ
KeyK
KeyL
KeyM
KeyN
KeyO
KeyP
KeyQ
KeyR
KeyS
KeyT
KeyU
KeyV
KeyW
KeyX
KeyY
KeyZ
Digit1
Digit2
Digit3
Digit4
Digit5
Digit6
Digit7
Digit8
Digit9
Digit0
Enter
Escape
Backspace
Tab
Space
Minus
Equal
BracketLeft
BracketRight
Backslash
Semicolon
Quote
Backquote
Comma
Period
Slash
CapsLock
F1
F2
F3
F4
F5
F6
F7
F8
F9
F10
F11
F12
PrintScreen
Insert
Home
PageUp
Delete
End
PageDown
ArrowRight
ArrowLeft
ArrowDown
ArrowUp
ControlLeft
ShiftLeft
AltLeft
MetaLeft
ControlRight
ShiftRight
AltRight
MetaRight
Pause
ScrollLock
NumLock
ContextMenu
NumpadDivide
NumpadMultiply
NumpadSubtract
NumpadAdd
NumpadEnter
Numpad1
Numpad2
Numpad3
Numpad4
Numpad5
Numpad6
Numpad7
Numpad8
Numpad9
Numpad0
NumpadDecimal
Power
IntlBackslash
IntlYen
IntlRo
KanaMode
Convert
NonConvert
AudioVolumeMute
AudioVolumeUp
AudioVolumeDown
F20
```

---

## 6) ATX API (Comet Pro power board) ⚡️

### 6.1 Read ATX state

```sh
curl -k -u "admin:<UI_PASSWORD>" "https://<host>/api/atx"
```

### 6.2 The endpoint the web UI actually uses: `/api/atx/click`

This is the reliable control route on Comet Pro:

* `POST /api/atx/click?button=<button>`

Observed working example:

```sh
curl -k -u "admin:<UI_PASSWORD>" -X POST \
  "https://<host>/api/atx/click?button=power"
```

Common button values typically include:

* `power` (short press)
* `power_long` (long press)
* `reset`

### 6.3 `/api/atx/power?action=...` (device-dependent)

Some builds expose an action-based endpoint, but on this Comet Pro build it was not reliable for actual physical action. Prefer `/api/atx/click`.

---

## 7) Exposing missing endpoints under `/api/...` (nginx) 🛠️

### Why you need this

Some endpoints exist internally at KVMD paths like:

* `/hid/events/send_key`
* `/hid/events/send_shortcut`

…but are not mapped under `/api/...` by default.

### Where configs live

* `/etc/kvmd/nginx/`
* common server ctx: `/etc/kvmd/nginx/kvmd.ctx-server.conf` and/or `/etc/kvmd/nginx/gl.ctx-server.conf`
* upstream definition: `/etc/kvmd/nginx/kvmd.ctx-http.conf`

Example upstream:

```nginx
upstream kvmd {
    server unix:/run/kvmd/kvmd.sock fail_timeout=0s max_fails=0;
}
```

### Example: expose `send_key` and `send_shortcut`

Add blocks like:

```nginx
location /api/hid/events/send_key {
    rewrite ^/api/hid/events/send_key$ /hid/events/send_key break;
    rewrite ^/api/hid/events/send_key\?(.*)$ /hid/events/send_key?$1 break;
    proxy_pass http://kvmd;
    include /etc/kvmd/nginx/loc-proxy.conf;
    include /etc/kvmd/nginx/loc-login.conf;
    include /etc/kvmd/nginx/loc-nocache.conf;
}

location /api/hid/events/send_shortcut {
    rewrite ^/api/hid/events/send_shortcut$ /hid/events/send_shortcut break;
    rewrite ^/api/hid/events/send_shortcut\?(.*)$ /hid/events/send_shortcut?$1 break;
    proxy_pass http://kvmd;
    include /etc/kvmd/nginx/loc-proxy.conf;
    include /etc/kvmd/nginx/loc-login.conf;
    include /etc/kvmd/nginx/loc-nocache.conf;
}
```

Restart nginx (command depends on firmware):

```sh
/etc/init.d/nginx restart
```

or:

```sh
systemctl restart nginx
```

---

## 8) Recommended automation pattern (what you should actually do) 🤖

### 8.1 Put macros on the Comet Pro

```sh
mkdir -p /root/macros
chmod 700 /root/macros
```

### 8.2 Use a generic script template

Create `/root/macros/example.sh`:

```sh
#!/bin/sh
set -eu

BASE="${BASE:-https://127.0.0.1}"
KVM_USER="${KVM_USER:-admin}"
KVM_PASS="${KVM_PASS:-CHANGE_ME}"

CURL="curl -fsS -k -u ${KVM_USER}:${KVM_PASS}"

hid_print() {
  printf "%s" "$1" | $CURL -X POST "$BASE/api/hid/print" --data-binary @- >/dev/null
}

tap_key() {
  key="$1"
  $CURL -X POST "$BASE/api/hid/events/send_key?key=$key&state=1" >/dev/null
  sleep 0.03
  $CURL -X POST "$BASE/api/hid/events/send_key?key=$key&state=0&finish=1" >/dev/null
}

countdown() {
  label="$1"
  seconds="$2"
  while [ "$seconds" -gt 0 ]; do
    echo "[*] $label: ${seconds}s remaining"
    sleep 1
    seconds=$((seconds - 1))
  done
}

echo "[*] Example macro starting"
hid_print "hello from kvmd"
tap_key "Enter"
countdown "Waiting" 3
echo "[+] Done"
```

Enable it:

```sh
chmod +x /root/macros/example.sh
```

### 8.3 Trigger from iOS Shortcuts

Use the Shortcuts action: **Run Script over SSH**

Command example:

```sh
KVM_PASS='<your-ui-password>' /root/macros/example.sh
```

This avoids browser state, cookies, and websockets.

---

## 9) Troubleshooting 🧯

### 404 Not Found on `/api/hid/events/send_key`

nginx does not expose it yet.

* Add nginx `location` rewrite blocks.
* Restart nginx.

### 405 Method Not Allowed

You used `GET`. Use `POST`.

### “None argument is not a valid Keyboard key”

Your request did not include `key` in the query string.

Use:

* `...?key=ArrowUp&state=1`

### ATX returns ok but does nothing

Use:

* `/api/atx/click?button=...`

Not all builds implement action-based `/api/atx/power?action=...` reliably.

---

## Appendix: Quick command cheatsheet 📌

### Type text

```sh
curl -k -u "admin:<UI_PASSWORD>" -X POST "https://<host>/api/hid/print" \
  --data-binary "cmd"
```

### Tap a key

```sh
curl -k -u "admin:<UI_PASSWORD>" -X POST \
  "https://<host>/api/hid/events/send_key?key=Enter&state=1"

curl -k -u "admin:<UI_PASSWORD>" -X POST \
  "https://<host>/api/hid/events/send_key?key=Enter&state=0&finish=1"
```

### ATX short press power

```sh
curl -k -u "admin:<UI_PASSWORD>" -X POST \
  "https://<host>/api/atx/click?button=power"
```
