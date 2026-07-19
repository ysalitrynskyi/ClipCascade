# <img src="https://github.com/user-attachments/assets/710bb1c3-0eda-48cf-819a-e066bde3a3ec" alt="ClipCascade Logo" width="34" /> ClipCascade — security-hardened fork

**ClipCascade** automatically syncs your clipboard across your devices, end-to-end encrypted, with no manual step.

This is a fork of [Sathvik-Rao/ClipCascade](https://github.com/Sathvik-Rao/ClipCascade) that applies a security baseline and publishes a
prebuilt multi-arch server image. The changes are proposed upstream in
[PR #163](https://github.com/Sathvik-Rao/ClipCascade/pull/163).

> **Why this fork exists.** Reviewing the upstream project for self-hosting turned up a set of
> defaults that are risky on a private network — a baked-in database password, a wildcard
> WebSocket origin, a forwarded-header path a client could steer, and containers published on
> every interface. Fixing those is what this fork is. Everything here is offered upstream; if it
> lands, this fork stops being necessary.

---

## What is different from upstream

| | Upstream | Here |
|---|---|---|
| Database password | Baked-in default | Required; the server refuses to start without it |
| `CC_ALLOWED_ORIGINS=*` | Accepted | Refused at startup — a wildcard lets any site open an authenticated WebSocket with a visitor's cookie and read their clipboard |
| `X-Forwarded-For` | Trusted from any peer | Trusted only from `CC_TRUSTED_PROXY_CIDRS`, canonicalised, so a client cannot pick the address brute-force lockout keys on |
| Container user | root | uid 10001 |
| Published port | `0.0.0.0` | `CC_BIND_ADDRESS`, defaulting to loopback |
| CSP | Inline scripts allowed | `script-src 'self'`, no inline scripts |
| Clipboard protocol | v1, no replay protection | v2 — device id, counter and timestamp bound inside the ciphertext |
| Client secrets | Plaintext on disk | OS keyring (desktop), Keychain/Keystore (mobile) |
| Android capture | Needed `READ_LOGS` / overlay | Share sheet, QS tile, notification action — no such permission |
| Server image | Docker Hub, amd64 | GHCR, `linux/amd64` + `linux/arm64` |

Full detail: [`SECURITY_SELF_HOST.md`](SECURITY_SELF_HOST.md). What is deliberately **not** done:
[`HARDENING_NEXT_STEPS.md`](HARDENING_NEXT_STEPS.md).

---

## Quick start (server)

A prebuilt multi-arch image is published to GHCR, so there is no build step.

```bash
git clone https://github.com/ysalitrynskyi/ClipCascade.git
cd ClipCascade/ClipCascade_Server/docker-compose

cp .env.tailscale-ghcr.example .env
# edit .env — at minimum:
#   CC_BIND_ADDRESS      this host's address (tailscale ip -4, or a LAN IP)
#   CC_ALLOWED_ORIGINS   the exact origin(s) you will open, comma separated
#   CC_SERVER_DB_PASSWORD  "<file password> <user password>" — the space is part of it
#   CC_INITIAL_ADMIN_PASSWORD  first boot only

docker compose -f docker-compose.tailscale-ghcr.yml --env-file .env pull
docker compose -f docker-compose.tailscale-ghcr.yml --env-file .env up -d
curl -sS http://<CC_BIND_ADDRESS>:8080/health   # -> OK
```

`ghcr.io/ysalitrynskyi/clipcascade:latest` runs on `linux/amd64` and `linux/arm64`, so x86 servers, Ampere VMs
and Raspberry Pi hosts all work.

Other topologies (Postgres, external broker, plain loopback) are in
[`ClipCascade_Server/docker-compose/`](ClipCascade_Server/docker-compose). All of them default to a
loopback bind; set `CC_BIND_ADDRESS` to expose the server deliberately rather than by accident.

**Already running the upstream image?** Read
[`docs/MIGRATE_EXISTING_DOCKER.md`](docs/MIGRATE_EXISTING_DOCKER.md) first. Two things bite: the
volume must be `chown`ed to uid 10001, and `CC_SERVER_DB_PASSWORD` must match the password the H2
files were created with or the database will not open.

### Running on Tailscale

Recommended, and what this fork is tuned for: bind the server to its tailnet address and never
open a port to the internet. See [`docs/TAILSCALE.md`](docs/TAILSCALE.md).

---

## Clients

> **Clients must be built from this fork.** Protocol v2 is enforced end to end, and the fix that
> makes it work over the default transport is client-side — so an upstream client build will not
> sync against this server, and vice versa. This fork does **not** publish prebuilt client
> binaries; build from source as below.

### Desktop (Windows, macOS, Linux)

The desktop client is Python and runs directly from source:

```bash
cd ClipCascade_Desktop/src
python -m pip install -r requirements_linux.txt   # or requirements_win.txt / requirements_mac.txt
python main.py
```

Secrets are stored in the OS keyring, so a keyring backend must be available — on a headless Linux
box install `gnome-keyring` or `kwallet` first. Without one the client refuses to write secrets to
disk rather than storing them in plaintext.

Linux needs a few system packages and has some well-known pitfalls; the dependency and
troubleshooting steps below are unchanged from upstream and still apply.

### Android

```bash
cd ClipCascade_Mobile/src
npm ci
cp android/gradle.properties.example android/gradle.properties
cd android && ./gradlew assembleRelease     # or assembleDebug
```

The APK lands in `android/app/build/outputs/apk/`. Release builds need a signing key — see
[`docs/SIGNING.md`](docs/SIGNING.md).

### iOS

Buildable from `ClipCascade_Mobile/src/ios`, but note the open item in
[`HARDENING_NEXT_STEPS.md`](HARDENING_NEXT_STEPS.md): the GCM nonce fix that makes desktop→iOS
decryption work is reasoned from Apple's documented contract and has not been confirmed on a
device.

---

## 📸 Screenshots

| 🪟 Desktop ([Windows](https://github.com/Sathvik-Rao/ClipCascade?tab=readme-ov-file#-windows-desktop-application)) | 🍏 Desktop ([macOS](https://github.com/Sathvik-Rao/ClipCascade?tab=readme-ov-file#-macos-desktop-application)) | 🤖📱 Mobile ([Android](https://github.com/Sathvik-Rao/ClipCascade?tab=readme-ov-file#-android-mobile-application)) | 🐧🖱️ Desktop ([Linux_GUI](https://github.com/Sathvik-Rao/ClipCascade?tab=readme-ov-file#%EF%B8%8F-linux-desktop-application-gui--%EF%B8%8F-linux-terminal-based-application-cli)) | 🐧⌨️ Desktop ([Linux_CLI](https://github.com/Sathvik-Rao/ClipCascade?tab=readme-ov-file#%EF%B8%8F-linux-desktop-application-gui--%EF%B8%8F-linux-terminal-based-application-cli)) | 
|-----------------------|--------------------|--------------------|--------------------|--------------------|
| <img src="https://github.com/user-attachments/assets/369d5db5-685c-4284-946d-b6a0e1f4fef9" alt="Desktop (Windows) - 1" width="360" /> | <img src="https://github.com/user-attachments/assets/2c0a7f4d-652c-4f4c-97e9-ee9b9d66f03f" alt="Desktop (macOS) - 1" width="360" /> | <img src="https://github.com/user-attachments/assets/a5606f3c-6d8a-434f-8f1d-03d6276e03c0" alt="Mobile (Android) - 1" width="360" /> | <img src="https://github.com/user-attachments/assets/f1acd9f4-27ee-4eb0-8696-a786a21551ed" alt="Desktop (Linux_GUI) - 1" width="360" /> | <img src="https://github.com/user-attachments/assets/f3f7c3a9-0299-4f0d-9494-5d9a102a243f" alt="Desktop (Linux_CLI) - 1" width="360" /> |
| <img src="https://github.com/user-attachments/assets/3d51539b-69d0-4b0d-8854-e262638333bd" alt="Desktop (Windows) - 2" width="240" /> | <img src="https://github.com/user-attachments/assets/3d473d8d-601e-4c78-bb7f-0684d39aef67" alt="Desktop (macOS) - 2" width="240" /> | <img src="https://github.com/user-attachments/assets/607135ff-498f-45ae-b60e-18da525b6b19" alt="Mobile (Android) - 2" width="240" /> | <img src="https://github.com/user-attachments/assets/394ab014-ae40-475d-8109-d95c9a69645b" alt="Desktop (Linux_GUI) - 2" width="240" /> | <img src="https://github.com/user-attachments/assets/daf0a4ac-4dcc-4547-9171-7bb0546f6712" alt="Desktop (Linux_CLI) - 2" width="240" /> |



## ✨ Features  

- **🚀 Instant Clipboard Sync** – Clipboard content updates in real time across all connected devices. Just copy, and it’s there!  
- **🔒 Secure Authentication** – Ensures only authorized users can sync clipboard data.  
- **🛡️ End-to-End Encryption** – Protects clipboard content with advanced cryptographic security and hashing techniques.  
- **🔄 Dual Sync Modes:**  
  - **☁️ Server-Based Sync** – Reliable cloud-based synchronization via a centralized server.  
  - **🔗 Peer-to-Peer Sync** – Direct device-to-device connection for ultra-low latency and minimal server dependency.  
- **💻 Cross-Platform Compatibility** – Works seamlessly on Windows, macOS, Linux, and Android.  
- **📄📷📁 Universal Clipboard** – Syncs text, images, and files effortlessly across devices.  
- **📦 Self-Hosting Option** – Deploy your own secure instance using a Docker image or standalone JAR file.  
- **👥 Multi-User Support** – Isolates clipboard data per user while enabling seamless syncing between personal devices.  
- **🌐 Web-Based Dashboard** – Track clipboard activity and manage settings through an intuitive interface.  
- **⚙️ Customizable Preferences** – Fine-tune sync settings for performance, security, and usability.  
- **🔔 Smart Update Notifications** – Stay informed about new features, security patches, and enhancements.  

   
<div align="center">
<table>
  <tr>
    <th>Type</th>
    <th>Windows</th>
    <th>MacOS</th>
    <th>Linux GUI</th>
    <th>Linux CLI</th>
    <th>Android</th>
  </tr>
  <tr>
    <td><strong>Text</strong></td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
  </tr>
  <tr>
    <td><strong>Image</strong></td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
  </tr>
  <tr>
    <td><strong>Files</strong></td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
    <td>✔</td>
  </tr>
</table>
</div>


---

## Linux desktop: dependencies and troubleshooting

#### Step 1: Check for updates and install required packages

##### Debian/Ubuntu:
```
sudo apt update
sudo apt install -y python3 python3-pip python3-gi xclip wl-clipboard dunst
```

##### Fedora:
```
sudo dnf check-update
sudo dnf install -y python3 python3-pip python3-gobject xclip wl-clipboard dunst
```

##### Arch:
```
sudo pacman -Syu --noconfirm python python-pip python-gobject xclip wl-clipboard dunst xdg-utils
```


#### Step 2: Install GTK 3.0 for clipboard monitoring and GUI support

##### Debian/Ubuntu:
```
sudo apt install -y python3-gi-cairo gir1.2-gtk-3.0 gir1.2-gdk-3.0
```

##### Fedora:
```
sudo dnf install -y libappindicator-gtk3
```

##### Arch:
```
sudo pacman -S --noconfirm python-gobject gtk3
```

#### Step 3: Install GNOME tray support extension (if tray icon is unavailable)
Install the [GNOME tray support extension](https://extensions.gnome.org/extension/615/appindicator-support/).


#### Step 4: Install Python Dependencies

##### Debian/Ubuntu/Fedora:
```
sudo pip3 install -r requirements.txt
```

##### Arch:
```
sudo pip install -r requirements.txt
```

#### Step 4.1: Fix `externally-managed-environment` Error (if applicable)

If you encounter the `error: externally-managed-environment`, install the required Python packages manually.

##### Debian/Ubuntu:
```
sudo apt install -y python3-xxhash python3-pyperclip python3-requests python3-websocket python3-pycryptodome python3-tk python3-pystray python3-pyfiglet python3-bs4 python3-plyer python3-aiortc
```

##### Fedora:
```
sudo dnf install -y python3-xxhash python3-pyperclip python3-requests python3-websocket-client python3-pycryptodomex python3-tkinter python3-pystray python3-pyfiglet python3-beautifulsoup4 python3-plyer python3-aiortc
```

##### Arch:
```
sudo pacman -S --noconfirm python-xxhash python-pyperclip python-requests python-websocket-client python-pycryptodome tk python-pystray python-pyfiglet python-beautifulsoup4 python-plyer python-aiortc
```
  - If you encounter the `error: target not found: python-plyer`, install via `yay -S --noconfirm python-plyer`
  - If you encounter the `error: target not found: python-aiortc`, install via `yay -S --noconfirm python-aiortc`


#### Step 4.2: Fix `Package libavformat was not found in the pkg-config search path` Error (if applicable)

##### Debian/Ubuntu:
```
sudo apt install -y libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libavfilter-dev libswscale-dev libswresample-dev pkg-config
```

##### Fedora:
```
sudo dnf install -y ffmpeg ffmpeg-devel
```

##### Arch:
```
sudo pacman -S --noconfirm ffmpeg
```


#### Step 5: Run the Application

Start ClipCascade by running (use sudo if needed):
   - When prompted, enter your **server's IP address, port number, or domain name**.
   - If encryption is enabled, ensure it is **enabled on all devices**.
   - In the **Extra Config** section, you can set a local clipboard size limit. By default, no limit is enforced (note: large file transfers may cause temporary unresponsiveness).
```
python3 main.py
```

Linux supports optional launch arguments to manually tune behavior when needed:
- `--gui <true|false>`: force **GUI** (tray / GTK login) with `true`, or **terminal CLI** with `false`, independent of the auto-selected default.
- `--xmode <true|false>`: force **X11 / XWayland-style** clipboard handling (`xclip`, GTK clipboard when applicable) with `true`, or **Wayland-style** handling (`wl-clipboard`) with `false`.
- `--polling <seconds>`: override clipboard polling interval (must be a positive number).

When flags are omitted, **display-server detection** sets the default `--xmode` behavior, and the default **GUI vs CLI** follows that same classification (GUI for X11, XWayland, and unknown; CLI for native Wayland and Hyprland). Use `--gui` / `--xmode` if your session variables do not match how you want the app to run.

Examples:
```bash
python3 main.py --gui true
python3 main.py --gui false --xmode false --polling 3
python3 main.py --polling 1.5
```

If these flags are not provided, ClipCascade automatically detects your environment and selects the most suitable clipboard path and interface.


#### Step 5.1: Fix 'No module named `Crypto`' Error (if applicable)

If you encounter the `No module named 'Crypto'` error, create a symbolic link for the Cryptodome library:
[see more](https://github.com/openthread/openthread/issues/1137#issuecomment-140879139)


##### Debian/Ubuntu:
```
sudo ln -s /usr/lib/python3/dist-packages/Cryptodome /usr/lib/python3/dist-packages/Crypto
```

##### Fedora:
```
sudo ln -s /usr/lib/python3/site-packages/Cryptodome /usr/lib/python3/site-packages/Crypto
```

##### Arch:
```
sudo ln -s /usr/lib/python3.*/site-packages/Cryptodome /usr/lib/python3.*/site-packages/Crypto
```

#### Step 5.2: Fix 'ModuleNotFoundError: No module named `tkinter`' Error (if applicable)

If you encounter the `No module named 'tkinter'` error:

##### Debian/Ubuntu:
```
sudo apt install -y python3-tkinter
```

##### Fedora:
```
sudo dnf install -y python3-tkinter
```

##### Arch:
```
sudo pacman -S --noconfirm tk
```

#### Step 5.3:  Fix '`gtk_widget_get_scale_factor: assertion 'GTK_IS_WIDGET (widget)' failed`' Error (if applicable)

This error occurs due to a missing tray icon extension in GNOME. To resolve it, you can install the extension from [here](https://extensions.gnome.org/extension/615/appindicator-support)


#### Step 5.4:  Fix '`g-exec-error-quark`' Error (if applicable)

##### Debian/Ubuntu:
```
sudo apt install dbus-x11
```

##### Fedora:
```
sudo dnf install dbus-x11
```

##### Arch:
```
sudo pacman -S --noconfirm dbus
```

#### Step 6: Run the Application in the Background (if needed)

To run ClipCascade as a background process (if the GUI setup is functional), use the following command:
```
nohup python3 main.py &> /dev/null &
```
If root privileges are required, use:
```
sudo nohup python3 main.py &> /dev/null &
```


#### Step 7: Add ClipCascade to Startup Script (if needed)

To ensure ClipCascade starts automatically when your system boots, modify the file paths as necessary and add the appropriate command to your startup script.

##### Example:
##### GUI
```
cd /path/to/clipcascade/src/ && nohup python3 main.py &> /dev/null &
```
If root privileges are required, use:
```
cd /path/to/clipcascade/src/ && sudo nohup python3 main.py &> /dev/null &
```

##### CLI
```
cd /path/to/clipcascade/src/ && python3 main.py
```
If root privileges are required, use:
```
cd /path/to/clipcascade/src/ && sudo python3 main.py
```

[➡️ Explore Advanced Details](https://github.com/Sathvik-Rao/ClipCascade?tab=readme-ov-file#%EF%B8%8F-advanced-details)


## ⚙️ Advanced Details


### 🗄️ Server Configuration

#### Security-relevant settings in this fork

These either behave differently from upstream or are new here. The full variable reference
follows further down.

| Variable | Default | Notes |
|---|---|---|
| `CC_SERVER_DB_PASSWORD` | *(none)* | **Required** for an on-disk H2 database; the server refuses to start without it. Format is `<file password> <user password>` — the space is part of the value, because the datasource uses `CIPHER=AES`. Upstream shipped a baked-in default, which also meant the "encrypted" file had a publicly known key. |
| `CC_ALLOWED_ORIGINS` | `http://localhost:8080` | Exact origins, comma separated. **`*` refuses to start**: it is applied verbatim to both WebSocket endpoints, so any page a logged-in user visits could open a socket with their cookie and read their clipboard. |
| `CC_BIND_ADDRESS` | `127.0.0.1` | Host address the container publishes on. Loopback by default so a server is never exposed on every interface by accident — set it to a tailnet or LAN address to reach it from other machines. |
| `CC_TRUSTED_PROXY_CIDRS` | *(unset)* | Only set this behind a reverse proxy you control, and keep it as narrow as possible — ideally the single proxy host, e.g. `172.18.0.2/32`. When unset, forwarded headers are ignored entirely and the socket peer is authoritative. **Never include a range containing clients**: everything inside it is trusted to declare who it is forwarding for, so a client in that range can choose the address brute-force lockout keys on. |
| `CC_FORWARDED_HEADER` | `X-Forwarded-For` | Which forwarding header to read, and only from a trusted proxy. Change it only if your proxy uses a different one, and make sure that proxy overwrites it on every request. |
| `CC_SESSION_TIMEOUT` | `1440m` (24h) | Upstream defaulted to roughly a year. Set it back to `525960m` if you would rather not sign in daily. |
| `CC_UPDATE_CHECK_ENABLED` | `true` | Set `false` to stop the server contacting GitHub for version checks — worth doing on an airgapped or private-mesh install. |

#### Important Security Notice:
**Set a strong initial admin password before first startup, then change it immediately after logging in** to prevent unauthorized access.

#### Initial Admin Credentials:
- **Username:** `admin` by default, configurable with `CC_INITIAL_ADMIN_USERNAME`
- **Password:** set `CC_INITIAL_ADMIN_PASSWORD` before first startup

#### Health Check Endpoint  
- **Purpose:** Verifies if the server is running and operational.  
- **Endpoint:** `/health`  
- **Response:** Returns `OK` with a status code `200` when the server is up and running.

#### Built-in Update Checker
- The server features a built-in update checker, prominently displayed on the homepage, keeping you informed about the latest enhancements and security fixes. This ensures your server stays up to date.

  <img src="https://github.com/user-attachments/assets/8184e4ad-d711-4fda-9382-eb3a252bc07b" alt="server_update" />
  

#### Environment Variables:

<table>
<thead>
<tr>
<th>Environment Variable</th>
<th>Extended Description</th>
<th>Default Value</th>
</tr>
</thead>
<tbody>

<!-- 1 -->
<tr>
<td>CC_MAX_MESSAGE_SIZE_IN_MiB</td>
<td>
Defines the maximum message size (in MiB) that the server can handle.
<br><br>
<strong>Note:</strong><br>
- Android typically supports larger clipboard sizes for images and files but limits text to ~1 MiB.<br>
- Desktop supports larger clipboard sizes across text, images, and files.
<br><br>
<strong>Additional Notes:</strong><br>
- Clients can set their own limits via the "Extra Config" on the login page.<br>
- If <code>CC_P2P_ENABLED</code> is <code>true</code>, this setting is ignored.
</td>
<td>1</td>
</tr>

<!-- 2 -->
<tr>
<td>CC_MAX_MESSAGE_SIZE_IN_BYTES</td>
<td>
Provides finer control over message size by specifying a limit in bytes.
<br><br>
<strong>Note:</strong> If set above zero, it overrides <code>CC_MAX_MESSAGE_SIZE_IN_MiB</code>.<br>
Ignored if <code>CC_P2P_ENABLED</code> is <code>true</code>.
</td>
<td>0</td>
</tr>

<!-- 3 -->
<tr>
<td>CC_P2P_ENABLED</td>
<td>
Toggles the Peer-to-Peer (P2P) feature, allowing direct device-to-device communication.
<br><br>
<strong>Advantages:</strong><br>
- Reduces server load.<br>
- Removes size restrictions, enabling unlimited data transfer.
<br><br>
<strong>Important Notes:</strong><br>
- If enabled, <code>CC_MAX_MESSAGE_SIZE_IN_MiB</code> and <code>CC_MAX_MESSAGE_SIZE_IN_BYTES</code> are ignored.<br>
- Some network configurations may not support P2P.
</td>
<td>false</td>
</tr>

<!-- 4 -->
<tr>
<td>CC_P2P_STUN_URL</td>
<td>
Defines the STUN server URL used for P2P communication, helping devices discover each other across NAT.
<br><br>
<strong>Note:</strong> Required when <code>CC_P2P_ENABLED</code> is <code>true</code>.<br>
You can use a public STUN server or host your own.
</td>
<td>stun:stun.l.google.com:19302</td>
</tr>

<!-- 5 -->
<tr>
<td>CC_ALLOWED_ORIGINS</td>
<td>
Specifies which domain is permitted to access the WebSocket server (CORS policy).
<br><br>
<strong>Security Note:</strong><br>
- Set this to the exact origin that serves the web app (for example, <code>https://clipcascade.example.com</code>).<br>
- Avoid <code>*</code> for browser-accessible deployments.
</td>
<td>http://localhost:8080</td>
</tr>

<!-- 6 -->
<tr>
<td>CC_SIGNUP_ENABLED</td>
<td>
Determines whether new users can sign up.
<br><br>
<strong>Default:</strong> <code>false</code> (public signups are disabled).
</td>
<td>false</td>
</tr>

<!-- 7 -->
<tr>
<td>CC_INITIAL_ADMIN_USERNAME</td>
<td>
Sets the initial administrator username used only when the user database is empty.
</td>
<td>admin</td>
</tr>

<!-- 8 -->
<tr>
<td>CC_INITIAL_ADMIN_PASSWORD</td>
<td>
Sets the initial administrator password used only when the user database is empty.
<br><br>
<strong>Required:</strong> A new server refuses to start until this value is set.
</td>
<td>(required)</td>
</tr>

<!-- 9 -->
<tr>
<td>CC_MAX_USER_ACCOUNTS</td>
<td>
Defines the maximum number of user accounts allowed on the server.
<br><br>
<strong>Note:</strong> <code>-1</code> means no limit.
</td>
<td>-1</td>
</tr>

<!-- 10 -->
<tr>
<td>CC_ACCOUNT_PURGE_TIMEOUT_SECONDS</td>
<td>
Specifies the duration (in seconds) after which inactive accounts are deleted.
<br><br>
<strong>Example:</strong> <code>63115200</code> (equivalent to 2 years).<br>
<strong>Note:</strong> <code>-1</code> disables automatic purging.
</td>
<td>-1</td>
</tr>

<!-- 11 -->
<tr>
<td>CC_PORT</td>
<td>
Defines the internal port where the ClipCascade server listens for connections.
<br><br>
<strong>Default:</strong> 8080, but can be changed if necessary.
</td>
<td>8080</td>
</tr>

<!-- 12 -->
<tr>
<td>CC_SESSION_TIMEOUT</td>
<td>
Specifies the duration before user sessions expire, using minute-based formatting (<code>[number]m</code>).
<br><br>
<strong>Default:</strong> <code>1440m</code> (1 day).
</td>
<td>1440m</td>
</tr>

<!-- 11 -->
<tr>
<td>CC_MAX_UNIQUE_IP_ATTEMPTS</td>
<td>
Sets the maximum number of failed login attempts from different IP addresses before an account is blocked.
</td>
<td>15</td>
</tr>

<!-- 12 -->
<tr>
<td>CC_MAX_ATTEMPTS_PER_IP</td>
<td>
Limits the number of failed login attempts allowed per IP before temporarily blocking that IP.
</td>
<td>30</td>
</tr>

<!-- 13 -->
<tr>
<td>CC_LOCK_TIMEOUT_SECONDS</td>
<td>
Defines the initial lockout duration (in seconds) after too many failed login attempts.
</td>
<td>60</td>
</tr>

<!-- 14 -->
<tr>
<td>CC_LOCK_TIMEOUT_SCALING_FACTOR</td>
<td>
Determines how the lockout time increases with each consecutive failed attempt.
<br><br>
<strong>Examples:</strong><br>
- Factor 1: 60, 120, 180…<br>
- Factor 2: 120, 240, 360…<br>
- Factor 3: 180, 360, 540…
</td>
<td>2</td>
</tr>

<!-- 15 -->
<tr>
<td>CC_BFA_CACHE_ENABLED</td>
<td>
Controls whether brute force attack (BFA) data is cached in memory and disk.
</td>
<td>false</td>
</tr>

<!-- 16 -->
<tr>
<td>CC_BFA_TRACKER_CACHE_MAX_JVM_ENTRIES</td>
<td>
Specifies the maximum number of entries in the BFA tracker cache, stored in JVM memory.
<br><br>
<strong>Note:</strong> Only used if <code>CC_BFA_CACHE_ENABLED</code> is <code>true</code>.
</td>
<td>50</td>
</tr>

<!-- 17 -->
<tr>
<td>CC_BFA_TRACKER_CACHE_RAM_PERCENTAGE</td>
<td>
Defines the percentage of the BFA tracker cache allocated to off-heap RAM.
<br><br>
<strong>Note:</strong> Only used if <code>CC_BFA_CACHE_ENABLED</code> is <code>true</code>.
</td>
<td>0</td>
</tr>

<!-- 18 -->
<tr>
<td>CC_BFA_TRACKER_CACHE_DISK_PERCENTAGE</td>
<td>
Defines the percentage of the BFA tracker cache allocated to disk.
<br><br>
<strong>Note:</strong> Only used if <code>CC_BFA_CACHE_ENABLED</code> is <code>true</code>.
</td>
<td>40</td>
</tr>

<!-- 19 -->
<tr>
<td>CC_SERVER_DB_USERNAME</td>
<td>
Specifies the username for the database connection.
</td>
<td>clipcascade</td>
</tr>

<!-- 20 -->
<tr>
<td>CC_SERVER_DB_PASSWORD</td>
<td>
Defines the password used for encrypting the user database.
<br><br>
<strong>Note:</strong><br>
- (H2) Replace <code>&lt;file password&gt; &lt;user password&gt;</code> with secure values.<br>
- Once set, you must use the same password whenever you migrate the database.
</td>
<td>QjuGlhE3uwylBBANMkX1 o2MdEoFgbU5XkFvTftky</td>
</tr>

<!-- 21 -->
<tr>
<td>CC_SERVER_DB_URL</td>
<td>
Defines the JDBC URL used to connect to the database.
<br><br>
<strong>Examples:</strong><br>
- PostgreSQL: <code>jdbc:postgresql://localhost:5432/clipcascade</code>
</td>
<td>jdbc:h2:file:./database/clipcascade;CIPHER=AES;MODE=PostgreSQL</td>
</tr>

<!-- 22 -->
<tr>
<td>CC_SERVER_DB_DRIVER</td>
<td>
Specifies the JDBC driver class used by the database connection.
<br><br>
<strong>Example:</strong> <code>org.postgresql.Driver</code> for PostgreSQL.
</td>
<td>org.h2.Driver</td>
</tr>

<!-- 23 -->
<tr>
<td>CC_SERVER_DB_HIBERNATE_DIALECT</td>
<td>
Sets the Hibernate dialect for the chosen database.
<br><br>
<strong>Example:</strong> <code>org.hibernate.dialect.PostgreSQLDialect</code> for PostgreSQL.
</td>
<td>org.hibernate.dialect.H2Dialect</td>
</tr>

<!-- 24 -->
<tr>
<td>CC_SERVER_LOGGING_LEVEL</td>
<td>
Sets the logging verbosity level (TRACE, DEBUG, INFO).
</td>
<td>INFO</td>
</tr>

<!-- 25 -->
<tr>
<td>CC_SERVER_LOG_HISTORY_MAX_DAYS</td>
<td>
Specifies how many days of logs to retain before they are rotated or removed.
</td>
<td>30</td>
</tr>

<!-- 26 -->
<tr>
<td>CC_SERVER_LOG_MAX_CAPACITY</td>
<td>
Defines the maximum total size of logs to keep before older files are purged.
</td>
<td>1GB</td>
</tr>

<!-- 27 -->
<tr>
<td>CC_LOG_BRUTE_FORCE_TRACKER_ENABLED</td>
<td>
Enables detailed logging of each login attempt in the Brute Force Attack (BFA) tracker.
<br>Useful for diagnosing repeated login failures.
</td>
<td>false</td>
</tr>

<!-- 28 -->
<tr>
<td>CC_EXTERNAL_BROKER_ENABLED</td>
<td>
Determines whether an external STOMP broker is used for message handling.
</td>
<td>false</td>
</tr>

<!-- 29 -->
<tr>
<td>CC_BROKER_HOST</td>
<td>
Specifies the STOMP broker host for external message handling.
<br><br>
<strong>Note:</strong> Only used if <code>CC_EXTERNAL_BROKER_ENABLED</code> is <code>true</code>.
</td>
<td>localhost</td>
</tr>

<!-- 30 -->
<tr>
<td>CC_BROKER_PORT</td>
<td>
Specifies the STOMP broker port for external message handling.
<br><br>
<strong>Note:</strong> Only used if <code>CC_EXTERNAL_BROKER_ENABLED</code> is <code>true</code>.
</td>
<td>61613</td>
</tr>

<!-- 31 -->
<tr>
<td>CC_BROKER_USERNAME</td>
<td>
Defines the STOMP broker username for external message handling.
<br><br>
<strong>Note:</strong> Only used if <code>CC_EXTERNAL_BROKER_ENABLED</code> is <code>true</code>.
</td>
<td>admin</td>
</tr>

<!-- 32 -->
<tr>
<td>CC_BROKER_PASSWORD</td>
<td>
Defines the STOMP broker password for external message handling.
<br><br>
<strong>Note:</strong> Only used if <code>CC_EXTERNAL_BROKER_ENABLED</code> is <code>true</code>.
</td>
<td>admin</td>
</tr>

<!-- 33 -->
<tr>
  <td>CC_MAX_WS_GLOBAL_CONNECTIONS</td>
  <td>
    Maximum number of global WebSocket connections allowed.
    <br><br>
    <strong>Supported Mode:</strong> Only applies when <code>CC_P2P_ENABLED</code> is <code>true</code>.
    <br><br>
    <strong>Note:</strong><br>
    - This value represents the total number of active WebSocket connections allowed across all users.<br>
    - <code>-1</code> means unlimited.
  </td>
  <td>-1</td>
</tr>

<!-- 34 -->
<tr>
  <td>CC_MAX_WS_CONNECTIONS_PER_USER</td>
  <td>
    Maximum number of WebSocket connections allowed per user.
    <br><br>
    <strong>Supported Mode:</strong> Only applies when <code>CC_P2P_ENABLED</code> is <code>true</code>.
    <br><br>
    <strong>Note:</strong><br>
    - This value limits how many simultaneous WebSocket connections a single user can have.<br>
    - <code>-1</code> means unlimited.
  </td>
  <td>-1</td>
</tr>

</tbody>
</table>

* * * * * * *

### 🖥️📱 Client Apps
- Logs (`clipcascade_log.log`) are stored in the installation directory on Windows and Linux, and in `<current user>/Library/Application Support/ClipCascade/` on macOS. These logs allow you to review application activity and are automatically reset each time the application is reopened, preventing indefinite growth.
- The `DATA` file stores settings and user details, enabling the app to retain this information across both restarts and updates.
- On Linux and macOS, a `ClipCascade.lock` file is created while the program is running. This file ensures that only a single instance of ClipCascade can be opened at a time.
- All apps include a built-in update check feature, conveniently displayed on the homepage or taskbar. This ensures you can quickly check for updates within the app, keeping you up to date with the latest enhancements and security fixes.

   <table>
    <tr>
        <td align="center"><strong>Desktop</strong></td>
        <td align="center"><strong>Mobile</strong></td>
    </tr>
    <tr>
        <td><img src="https://github.com/user-attachments/assets/92583d05-769e-4883-b427-7b4a41815610" alt="desktop_update" /></td>
        <td><img src="https://github.com/user-attachments/assets/4b5b1d34-4f0f-4770-805c-b212d85aaa2b" alt="android_update" /></td>
    </tr>
   </table>
  
#### Extra Config/Advanced Settings (Desktop/Mobile):
- **Maximum Clipboard Size Local Limit (in bytes)**: If the app crashes or stops unexpectedly, it may be due to receiving clipboard content exceeding the platform's maximum size limit. You can set a local size limit by specifying a value in bytes (e.g., 512 KiB = 524288 bytes) to test different thresholds suitable for your device. This local limit works alongside the server-specified limit to ensure smoother operation without crashes. For example, on Android (particularly on the Pixel 6a as of 2024), the platform limit(for text) is typically less than 1 MiB. Since the server limit cannot go below 1 MiB, setting the local limit to around 900,000 bytes on the Pixel 6a can help prevent crashes.
- **Store Password Locally (not recommended)**: Enable this option if you frequently encounter session logouts. While the app stores session cookies for an extended period, a server restart may prompt a re-login. If re-entering the password becomes tedious, you can use this option to store your password locally for convenience.
   > Note: This option will only work if encryption is disabled, as encryption requires the raw password to generate a password hash.
- **Enable Image Sharing and Enable File Sharing**: Enabling these options allows the app to send images or files. However, the app will continue to receive images and files even if these options are disabled.
- **Enable Notification**: Turn on this option to receive notifications about WebSocket disconnections and reconnections.
- **Enable Encryption (recommended)**: Enabling this option activates end-to-end encryption for clipboard data. This ensures that all clipboard content is encrypted before leaving your device. Refer to the section below on E2E encryption for detailed instructions on how it works and how to configure the `salt` and `hash rounds`.

  #### Desktop (Specific):
  - **Default File Download Location**: When this path is set, the app will save files directly to the specified location without prompting the user each time the "Download Files" button is clicked.
  - **SSL CA bundle**: Path to a PEM file containing your root CA (or full chain) used for HTTPS/WSS verification. Leave it empty to use default public CA/OS trust. Use this field when your ClipCascade server certificate is signed by a private/internal CA (for example, corporate PKI).
  
  #### Android (Specific):
  - **Run on System Startup**: Enable this option to allow the app to automatically start on system reboot. Prefer Share sheet / Quick Settings tile / notification "Share clipboard now" for capture (READ_LOGS overlay capture was removed).
  - **Enable WebSocket Status Notification**: Receive alerts when the WebSocket connection is lost or restored, ensuring you're informed about any connection disruptions.
    
    <img src="https://github.com/user-attachments/assets/6a8b903c-ee52-444c-a14e-bed70e31dcee" alt="periodic_check_notification" width="250" />

  - **Enable Periodic Checks**: Enabling this option performs periodic checks to ensure clipboard monitoring and the foreground service are running. It verifies the service status when monitoring starts and then checks every 15 minutes in the background. If the service is not running, a notification is displayed. Clicking the notification will restart the service.
    
    <img src="https://github.com/user-attachments/assets/7341b960-5e60-4af6-b627-2183088de262" alt="periodic_check_notification" width="250" />

* * * * * * *

### 🔒 End-to-End Encryption Configuration for Clipboard Data

When encryption is enabled, clipboard data is encrypted directly on the client devices, ensuring true end-to-end encryption. The server does not store the encryption key, offering maximum security for your data.

Enabling the encryption option is all that's needed to activate encryption by default. However, advanced users have the option to further customize the encryption process by adjusting the following parameters:

- **Salt**: An optional string used as an additional input in the hashing process. For example, you could use a unique string like `"myCustomSalt123"` to enhance encryption security.
  
- **Hash Rounds**: An integer that specifies the number of hashing iterations. Increasing the hash rounds strengthens encryption by making it more computationally demanding. You can increase the default value of 664,937 to a higher number, such as 1,000,000, to boost security.

It is crucial to ensure the same **salt** and **hash rounds** are used across all client devices to maintain compatibility.

You can adjust these settings in the **Extra Config** section on the login page for users who require enhanced encryption options.

<img src="https://github.com/user-attachments/assets/59252f1c-c149-43c2-b0a2-c09ca075a5c1" alt="e2e_p2s" />

> Note: In a peer-to-peer architecture, clipboard data is broadcasted to all connected devices directly without the need for a server. The encryption mechanism remains unchanged, ensuring the same level of security across all devices.

* * * * * * *

### 📋 Clipboard Functioning
- **Text and Images**: These are directly copied to the clipboard, enabling seamless sharing across devices.  
- **Files**: When files are received, a notification icon appears in the system tray (on desktop platforms). Since the clipboard does not store files, only their file paths are retained.  
  
   <table>
    <tr>
        <td align="center"><strong>Desktop</strong></td>
        <td align="center"><strong>Mobile</strong></td>
    </tr>
    <tr>
        <td><img src="https://github.com/user-attachments/assets/c4f06a97-1fc5-47e6-9ff5-d4dc3ff41c68" alt="desktop_tray_icon" /></td>
        <td><img src="https://github.com/user-attachments/assets/7708d786-3c95-4892-8815-0792fc2fd465" alt="android_notification" width="150" /></td>
    </tr>
    <tr>
	<td><img src="https://github.com/user-attachments/assets/eca756ad-76de-4fb9-ba14-619e92172758" alt="desktop_tray_options" width="150" /></td>
        <td><img src="https://github.com/user-attachments/assets/c06e8584-fbed-4ae1-b3b3-d12664115178" alt="android_home_screen" width="150" /></td>
    </tr>
   </table>


- **Clipboard Monitoring**:

   | **Platform**              | **Implementation Details**                                                                                                                                              |
   |---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
   | **Windows**               | Uses `win32gui`, `win32api`, `win32con`, and `win32clipboard` to capture clipboard changes in an event-driven manner.                                         |
   | **macOS**                 | Utilizes the `pasteboard` to monitor the clipboard with polling every 0.3 seconds. Instead of checking clipboard content directly, it compares a counter to detect changes efficiently. |
   | **Linux (X11)**           | Integrates `Gtk.Clipboard` to capture clipboard changes in an event-driven manner. **Default UI:** GUI (tray).                                                         |
   | **Linux (XWayland)**      | Same X11-style stack as above: `Gtk.Clipboard` when possible, otherwise `xclip` polling (~0.3s). **Default UI:** GUI—not CLI. Detection treats XWayland as a Wayland session that also has `DISPLAY` and `WAYLAND_DISPLAY` set. |
   | **Linux (Unknown)**     | Treated like the X11-style path for defaults (GTK / `xclip` as above). **Default UI:** GUI. |
   | **Linux (native Wayland, Hyprland)** | Tries **`wl-paste --watch` first** (**event-driven** where the compositor supports it: clipboard changes wake the watcher instead of a fixed timer). If `--watch` is not supported or exits immediately, falls back to **polling** `wl-paste -l` (default every 3 seconds; `--polling`). **Default UI:** CLI (terminal). Override with `--gui` / `--xmode`. |
   | **Android**               | Uses `ClipboardManager` to capture clipboard changes in an event-driven manner.                                                                         |


  <table border="1">
  <tr>
    <th rowspan="2">Type</th>
    <th colspan="5" align="center">Send</th>
    <th colspan="5" align="center">Receive</th>
  </tr>
  <tr>
    <th>Windows</th>
    <th>MacOS</th>
    <th>Linux GUI</th>
    <th>Linux CLI</th>
    <th>Android</th>
    <th>Windows</th>
    <th>MacOS</th>
    <th>Linux GUI</th>
    <th>Linux CLI</th>
    <th>Android</th>
  </tr>
  <tr>
    <td>Text</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱ / share</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
  </tr>
  <tr>
    <td>Image</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱ / share</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
  </tr>
  <tr>
    <td>Files</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>✱</td>
    <td>share</td>
    <td>click</td>
    <td>click</td>
    <td>click</td>
    <td>click</td>
    <td>click</td>
  </tr>
  </table>


## ⇄ Reverse Proxy Setup

Below is a screenshot demonstrating how to configure a reverse proxy using **Cloudflare Tunnels**. Similar configurations can be applied with other providers as well.

<img src="https://github.com/user-attachments/assets/0f45879f-307a-4f5f-9f26-ca5d3de7b1cf" alt="Reverse Proxy web" width="600" />

<img src="https://github.com/user-attachments/assets/5cdca092-8d6b-4b3e-ac2b-fb10390a4ca2" alt="login form desktop" width="400" />

### Note:
For other providers, you may need to configure HTTP to HTTPS redirection manually by adding a permanent redirection rule.

Example: Caddy Configuration
```
http://clipcascade.sample.com {
	redir https://clipcascade.sample.com{uri} permanent
}
```

### Note:
Additionally, it might be helpful to mention that the server uses WebSockets (ws/wss) for live clipboard broadcasting. In most cases, no extra configuration is needed for WebSockets since they typically rely on an HTTP switching protocol. Most providers will support WebSocket connections out of the box, without requiring additional setup. Example: `ws://localhost:8080/clipsocket`, `ws://localhost:8080/p2psignaling`.


## 🔓 SSO Proxy Bypass Rules

When fronting ClipCascade with an SSO proxy (Auth0, Keycloak, etc.), you’ll need to exempt both the unauthenticated HTTP URLs _and_ the raw-WebSocket handshake endpoints. Add these rules to your proxy’s “bypass” or “whitelist” list:

#### HTTP(S) endpoints
```
/login
/logout
/signup
/captcha
/help
/donate
/health
/ping
/assets/**
```

#### WebSocket handshake endpoints
```
/p2psignaling   # P2P mode
/clipsocket     # P2S mode
```

## 🔧 Usage

1. **Login:** Use your credentials to log into ClipCascade.
2. **Sync:** Copy any text or content to your clipboard, and it will automatically sync across your connected devices.
3. **Monitor:** Open the web-based monitoring page to see your clipboard history in real-time.

## Contributing

Fixes that are not fork-specific belong upstream at [Sathvik-Rao/ClipCascade](https://github.com/Sathvik-Rao/ClipCascade) —
that is where the project lives and where everyone benefits. This fork carries the security
baseline while [PR #163](https://github.com/Sathvik-Rao/ClipCascade/pull/163) is open.

Issues specific to the hardening or the published image: use this fork's tracker.

Before proposing changes here, run the suites:

```bash
python3 scripts/check-versions.py
cd ClipCascade_Desktop/src && python -m unittest discover -s tests
cd ClipCascade_Mobile/src   && npx jest --watchAll=false && npx eslint .
cd ClipCascade_Server/ClipCascade_Backend && ./mvnw test
```

## 📦 Versioning

ClipCascade uses **Semantic Versioning (SemVer)** for releases:

- **🔴 Major (X)**: Incremented for releases that introduce **backward incompatible changes**.
- **🟠 Minor (Y)**: Incremented for **backward compatible functionality** added.
- **🟢 Patch (Z)**: Incremented for **backward compatible bug fixes** or minor improvements.

### Version Format

**X.Y.Z**  
Where:
- **X** is the major version.
- **Y** is the minor version.
- **Z** is the patch version.
  
Example versioning:
- **1.0.0**: Initial release.
- **1.1.0**: Added new features, backward compatible.
- **2.0.0**: Major changes, **not backward compatible**.


## License

GPL-3.0, inherited from upstream. See [`LICENSE`](LICENSE).

ClipCascade is the work of [Sathvik Rao](https://github.com/Sathvik-Rao) and its contributors;
this fork only adds the security baseline described above.

## Support

For upstream questions and the community server, see
[Sathvik-Rao/ClipCascade](https://github.com/Sathvik-Rao/ClipCascade).
For this fork, open an issue here.
