# GL.iNet Comet Pro KVMD Automation Guide (API, ATX, WebTerm, SSH, Macros) 🧠🔌

This guide is a practical, copy-pasteable walkthrough for automating a **GL.iNet Comet Pro** (KVMD-based KVM).

It focuses on **what actually works** on the Comet Pro firmware today and how to wire it together cleanly for automation (especially from **iOS Shortcuts**).

> **Compatibility note**
> Some of this may also work on other GL.iNet KVMD-based devices, but this is **untested**. Endpoints, paths, and nginx configs may differ across models and firmware versions.

---

## What this setup gives you ✅

You can automate things like:

* Type text into the target machine (BIOS, OS, login prompts)
* Press single keys, shortcuts, sequences, or spam keys (example: `Delete` during reboot)
* Press ATX buttons (power, long power press, reset)
* Trigger complex, multi-step macros over **SSH**

---

## Architecture overview 🧩

The Comet Pro runs:

* **KVMD** (backend API)
* **nginx** (frontend reverse proxy)
* Several services wired via **UNIX sockets**, typically:

  * KVMD backend: `/run/kvmd/kvmd.sock`
  * Web terminal (`ttyd`): `/run/kvmd/ttyd.sock`

By default, **not all KVMD endpoints are exposed under `/api/...`**. Some useful internal routes exist only as `/hid/...` and must be explicitly proxied by nginx.

---

## Security and persistence warnings 🔒⚠️

* Do **not** expose the Comet Pro directly to the public internet.
* Prefer LAN-only access or a VPN (Tailscale, WireGuard).
* Use strong credentials.
* Prefer key-based SSH auth.
* **nginx changes are firmware-dependent and may be lost on updates.**

  * Keep a backup of modified config files.
  * Expect to re-apply changes after firmware upgrades.

---

## 1) Authentication basics (important) ✅

### KVMD API credentials

For the Comet Pro KVMD API, Basic Auth uses:

* **Username:** `admin`
* **Password:** the **same UI password** set during first-time setup

### TLS / self-signed certs

The Comet Pro uses a self-signed TLS certificate. For `curl`, you will usually need:

* `-k` / `--insecure`

---

## 2) First step: expose missing KVMD endpoints (required) 🛠️

### Why this is necessary

Several KVMD endpoints exist internally but are **not mapped under `/api/...` by default**. For example:

* `/hid/events/send_key`
* `/hid/events/send_shortcut`

The web UI can reach them internally, but external automation cannot until nginx rewrites are added.

> **Important:** Do this **before** trying any HID automation examples.

---

### 2.1 Where nginx configs live

Common locations on the Comet Pro:

* `/etc/kvmd/nginx/`
* Server context:

  * `/etc/kvmd/nginx/kvmd.ctx-server.conf`
  * or `/etc/kvmd/nginx/gl.ctx-server.conf`
* Upstream definition:

  * `/etc/kvmd/nginx/kvmd.ctx-http.conf`

Example upstream (usually already present):

```nginx
upstream kvmd {
    server unix:/run/kvmd/kvmd.sock fail_timeout=0s max_fails=0;
}
```

---

### 2.2 Expose required HID endpoints

Add the following `location` blocks:

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

Restart nginx (method depends on firmware):

```sh
/etc/init.d/nginx restart
```

or:

```sh
systemctl restart nginx
```

---

## 3) Access methods: Web UI, WebTerm, SSH 🧭

### Web UI

Use the Web UI to confirm:

* video capture works
* keyboard works
* ATX buttons work

### Web terminal (webterm)

The Web Terminal is an embedded **`ttyd`** instance proxied by nginx.

It is useful for debugging but **not recommended for automation**.

Key wiring:

* nginx config: `/usr/share/kvmd/extras/webterm/nginx.ctx-server.conf`
* upstream socket: `/run/kvmd/ttyd.sock`

Endpoint:

* `https://<host>/extras/webterm/ttyd/`

### SSH (recommended)

Comet Pro runs **Dropbear** on port 22.

SSH is the cleanest automation primitive:

* iOS Shortcuts can run one SSH command
* No cookies, WebSockets, or terminal state

---

## 4) Fix SSH for automation 🔑

### 4.1 Set a known root password

From WebTerm:

```sh
passwd
```

### 4.2 Add SSH keys (recommended)

```sh
ssh-keygen -t ed25519 -C "comet-pro"
ssh root@<host> "mkdir -p /root/.ssh && chmod 700 /root/.ssh"
scp ~/.ssh/id_ed25519.pub root@<host>:/root/.ssh/authorized_keys
ssh root@<host> "chmod 600 /root/.ssh/authorized_keys"
```

---

## 5) KVMD HID API (keyboard + typing) ⌨️

### Critical gotcha: query-string parameters only

On this Comet Pro build, HID handlers like `send_key` read parameters from the **URL query string**, not JSON bodies.

If `key` is missing from the query, you will see:

* `None argument is not a valid Keyboard key`

---

### 5.1 Type text

* Endpoint: `POST /api/hid/print`
* Body: raw text

Example:

```sh
curl -k -u admin:PASS -X POST https://kvm/api/hid/print \
  --data-binary 'cmd'
```

---

### 5.2 Send a key event

* Endpoint: `POST /api/hid/events/send_key`
* Query params:

  * `key=<KeyName>`
  * `state=1` (down) or `state=0` (up)
  * optional `finish=1`

Example (tap Enter):

```sh
curl -k -u admin:PASS -X POST \
  'https://kvm/api/hid/events/send_key?key=Enter&state=1'
curl -k -u admin:PASS -X POST \
  'https://kvm/api/hid/events/send_key?key=Enter&state=0&finish=1'
```

---

### 5.3 Shortcuts (manual pattern)

Recommended sequence:

1. modifier down
2. main key tap
3. modifier up (reverse order)

Example (Ctrl+Alt+Delete):

```sh
curl -k -u admin:PASS -X POST \
  'https://kvm/api/hid/events/send_key?key=ControlLeft&state=1'
curl -k -u admin:PASS -X POST \
  'https://kvm/api/hid/events/send_key?key=AltLeft&state=1'

curl -k -u admin:PASS -X POST \
  'https://kvm/api/hid/events/send_key?key=Delete&state=1'
curl -k -u admin:PASS -X POST \
  'https://kvm/api/hid/events/send_key?key=Delete&state=0&finish=1'

curl -k -u admin:PASS -X POST \
  'https://kvm/api/hid/events/send_key?key=AltLeft&state=0&finish=1'
curl -k -u admin:PASS -X POST \
  'https://kvm/api/hid/events/send_key?key=ControlLeft&state=0&finish=1'
```

---

## 6) Supported keys (Comet Pro build) 🎛️

<details>
<summary><strong>Full supported key list</strong></summary>

```text
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

</details>

---

## 7) ATX API (Comet Pro power board) ⚡️

The Comet Pro exposes ATX controls under `/api/atx`. These map directly to the same actions used by the web UI.

### Read ATX state

```sh
curl -k -u admin:PASS https://kvm/api/atx
```

### Click ATX buttons

* `POST /api/atx/click?button=power`
* `POST /api/atx/click?button=power_long`
* `POST /api/atx/click?button=reset`

Example:

```sh
curl -k -u admin:PASS -X POST \
  'https://kvm/api/atx/click?button=power'
```

> Some firmware builds expose `/api/atx/power?action=...`, but on the Comet Pro this was unreliable. Prefer `/api/atx/click`.

---

## 8) Recommended automation pattern 🤖

For macros, store scripts under `/usr/share/kvmd/extras/scripts/` and call them via SSH.

### 8.1 Create a script directly (copy-paste)

This creates a minimal macro that types a string and presses Enter.

```sh
mkdir -p /usr/share/kvmd/extras/scripts
cat > /usr/share/kvmd/extras/scripts/hello-print-enter.sh <<'SH'
#!/bin/sh
set -eu

BASE="https://127.0.0.1"
KVM_USER="admin"
KVM_PASS="CHANGE_ME"

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

hid_print "hello from kvmd"
tap_key "Enter"
SH
chmod +x /usr/share/kvmd/extras/scripts/hello-print-enter.sh
echo OK
```

### 8.2 Trigger via SSH (iOS Shortcuts)

Because the password is stored inside the script, you can run it directly:

```sh
/usr/share/kvmd/extras/scripts/hello-print-enter.sh
```

---

## 9) Troubleshooting 🧯

* **404 on `/api/hid/...`**: nginx rewrite missing or lost after update
* **405 Method Not Allowed**: wrong HTTP method (must be `POST`)
* **Invalid key error**: `key` missing from query string
* **ATX does nothing**: use `/api/atx/click`, not action-based endpoints
