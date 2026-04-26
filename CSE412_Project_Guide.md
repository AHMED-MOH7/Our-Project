# CSE412 — Step-by-Step Project Guide
## Qt5 TCP Chat on Yocto + QEMU

This guide walks through every step needed to build, run, and demonstrate the project from scratch.

---

## Prerequisites

- WSL2 with Ubuntu 24.04
- Yocto Kirkstone (`poky`) already cloned at `~/yocto/poky`
- About 50 GB of free disk space (Yocto build is large)
- RealVNC Viewer installed on Windows (for the GUI step)

---

## Part 1 — Create the `meta-chat` Layer

### Step 1.1 — Scaffold the layer

```bash
cd ~/yocto/poky
source oe-init-build-env build   # always from ~/yocto/poky, NOT from inside build/

cd ~/yocto
bitbake-layers create-layer meta-chat
```

### Step 1.2 — Create the directory structure

```bash
mkdir -p ~/yocto/meta-chat/recipes-chat/chat-server/files
mkdir -p ~/yocto/meta-chat/recipes-chat/chat-client/files
mkdir -p ~/yocto/meta-chat/recipes-chat/images
```

### Step 1.3 — Register the layer

Edit `~/yocto/poky/build/conf/bblayers.conf` and add the `meta-chat` path:

```
BBLAYERS ?= " \
  /home/hp/yocto/poky/meta \
  /home/hp/yocto/poky/meta-poky \
  /home/hp/yocto/poky/meta-yocto-bsp \
  /home/hp/yocto/meta-chat \
  "
```

### Step 1.4 — Configure `local.conf`

Ensure `~/yocto/poky/build/conf/local.conf` contains:

```
MACHINE ??= "qemux86-64"
```

---

## Part 2 — Write the Source Code

### Step 2.1 — Server header (`server.h`)

Create `~/yocto/meta-chat/recipes-chat/chat-server/files/server.h`:

```cpp
#pragma once
#include <QMainWindow>
#include <QTcpServer>
#include <QTcpSocket>
#include <QTextEdit>
#include <QLabel>
#include <QList>

class ChatServer : public QMainWindow {
    Q_OBJECT
public:
    explicit ChatServer(QWidget *parent = nullptr);
    ~ChatServer();

private slots:
    void onNewConnection();
    void onReadyRead();
    void onClientDisconnected();

private:
    void startServer(quint16 port);
    void broadcast(const QByteArray &data, QTcpSocket *exclude);
    void log(const QString &msg);

    QTcpServer  *m_server;
    QList<QTcpSocket*> m_clients;
    QTextEdit   *m_logView;
    QLabel      *m_statusLabel;
};
```

### Step 2.2 — Server implementation (`server.cpp`)

Create `~/yocto/meta-chat/recipes-chat/chat-server/files/server.cpp` — full source in submission doc.

### Step 2.3 — Server main (`main-server.cpp`)

```cpp
#include <QApplication>
#include "server.h"

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    ChatServer window;
    window.show();
    return app.exec();
}
```

### Step 2.4 — Server qmake project (`chat-server.pro`)

```
TEMPLATE = app
TARGET   = chat-server
QT      += widgets network
SOURCES += main-server.cpp server.cpp
HEADERS += server.h
```

### Step 2.5 — Client files

Mirror the same pattern for the client:
- `client.h`, `client.cpp`, `main-client.cpp`, `chat-client.pro`
- `main-client.cpp` is identical to main-server.cpp but instantiates `ChatClient`.
- `chat-client.pro` uses `TARGET = chat-client` and references client sources.

Full source code is in the submission document.

---

## Part 3 — Write the Systemd Service

Create `~/yocto/meta-chat/recipes-chat/chat-server/files/chat-server.service`:

```ini
[Unit]
Description=Qt5 TCP Chat Server
After=network.target

[Service]
Type=simple
User=root
Environment=QT_QPA_PLATFORM=offscreen
ExecStart=/usr/bin/chat-server
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

> `QT_QPA_PLATFORM=offscreen` is **critical** — without it, Qt fails to find a display and the server process exits immediately.

---

## Part 4 — Write the BitBake Recipes

### Step 4.1 — Server recipe (`chat-server_1.0.bb`)

Create `~/yocto/meta-chat/recipes-chat/chat-server/chat-server_1.0.bb`:

```bitbake
SUMMARY = "Qt5 TCP Chat Server"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

DEPENDS = "qtbase qtbase-native"

SRC_URI = " \
    file://server.cpp \
    file://server.h \
    file://main-server.cpp \
    file://chat-server.pro \
    file://chat-server.service \
"

S = "${WORKDIR}"

inherit qmake5

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/chat-server ${D}${bindir}/chat-server

    install -d ${D}/lib/systemd/system
    install -m 0644 ${WORKDIR}/chat-server.service \
        ${D}/lib/systemd/system/chat-server.service

    install -d ${D}/lib/systemd/system/multi-user.target.wants
    ln -sf ../chat-server.service \
        ${D}/lib/systemd/system/multi-user.target.wants/chat-server.service
}

FILES:${PN} += " \
    ${bindir}/chat-server \
    /lib/systemd/system/chat-server.service \
    /lib/systemd/system/multi-user.target.wants/chat-server.service \
"
```

> **Do NOT use `inherit systemd`** — in Kirkstone it generates a postinstall script that calls `systemctl enable` during rootfs creation, which always fails with exit code 1. Instead, pre-create the symlink in `do_install`.

### Step 4.2 — Client recipe (`chat-client_1.0.bb`)

```bitbake
SUMMARY = "Qt5 TCP Chat Client"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

DEPENDS = "qtbase qtbase-native"

SRC_URI = " \
    file://client.cpp \
    file://client.h \
    file://main-client.cpp \
    file://chat-client.pro \
"

S = "${WORKDIR}"

inherit qmake5

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/chat-client ${D}${bindir}/chat-client
}

FILES:${PN} += "${bindir}/chat-client"
```

### Step 4.3 — Image recipes

**`~/yocto/meta-chat/recipes-chat/images/chat-server-image.bb`**:
```bitbake
require recipes-core/images/core-image-minimal.bb

IMAGE_INSTALL:append = " \
    chat-server \
    qtbase \
    qtbase-plugins \
    openssh \
"

IMAGE_FEATURES:append = " ssh-server-openssh"
```

**`~/yocto/meta-chat/recipes-chat/images/chat-client-image.bb`**:
```bitbake
require recipes-core/images/core-image-minimal.bb

IMAGE_INSTALL:append = " \
    chat-client \
    qtbase \
    qtbase-plugins \
    openssh \
"

IMAGE_FEATURES:append = " ssh-server-openssh"
```

---

## Part 5 — Build the Images

```bash
cd ~/yocto/poky
source oe-init-build-env build

# Clean previous artifacts if rebuilding
bitbake -c cleansstate chat-server chat-client

# Build both images (takes ~1–2 hours first time)
bitbake chat-server-image chat-client-image
```

Confirm the images are ready:
```bash
ls ~/yocto/poky/build/tmp/deploy/images/qemux86-64/*.ext4
```

You should see `chat-server-image-qemux86-64-*.rootfs.ext4` and `chat-client-image-qemux86-64-*.rootfs.ext4`.

---

## Part 6 — Set Up the Network Bridge

Run these commands **once** in WSL2 (you'll need root):

```bash
# Create bridge
sudo ip link add br0 type bridge
sudo ip addr add 192.168.10.1/24 dev br0
sudo ip link set br0 up

# Create TAP interface for server VM
sudo ip tuntap add tap0 mode tap
sudo ip link set tap0 master br0
sudo ip link set tap0 up

# Create TAP interface for client VM
sudo ip tuntap add tap1 mode tap
sudo ip link set tap1 master br0
sudo ip link set tap1 up
```

Verify the bridge:
```bash
ip addr show br0    # should show 192.168.10.1/24
ip link show tap0   # should show UP
ip link show tap1   # should show UP
```

---

## Part 7 — Run the QEMU Machines

### Terminal 1 — Server QEMU

```bash
cd ~/yocto/poky
source oe-init-build-env build

runqemu chat-server-image nographic \
  qemuparams="-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
              -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:02"
```

Wait for the login prompt, then log in as `root` (no password).

Configure networking inside the server VM:
```bash
ip addr add 192.168.10.2/24 dev eth0
ip link set eth0 up
```

### Terminal 2 — Client QEMU

```bash
runqemu chat-client-image nographic \
  qemuparams="-netdev tap,id=net0,ifname=tap1,script=no,downscript=no \
              -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:03"
```

Log in as `root`, configure networking:
```bash
ip addr add 192.168.10.3/24 dev eth0
ip link set eth0 up
ping -c 2 192.168.10.2   # must succeed before continuing
```

---

## Part 8 — Verify the Chat is Working

### Check server is listening (in server VM)

```bash
# Port 12345 = 0x3039 — appears in tcp6 (IPv6 dual-stack)
grep 3039 /proc/net/tcp6
```

You should see a line with local address ending in `3039` and state `0A` (LISTEN).

### Watch server logs (in server VM)

```bash
journalctl -fu chat-server -o cat | grep -v "QFont\|Qt no longer\|This plugin"
```

Leave this running — you will see connections arrive here.

### Run the client (in client VM)

```bash
QT_QPA_PLATFORM=offscreen chat-client 192.168.10.2
```

The client terminal will print:
```
[HH:MM:SS] Connecting to 192.168.10.2:12345...
[HH:MM:SS] Connected to server.
[HH:MM:SS] [Me] Hello from chat-client on QEMU!
```

The server journal will show:
```
[HH:MM:SS] Client connected: ::ffff:192.168.10.3:XXXXX
[HH:MM:SS] [::ffff:192.168.10.3] Hello from chat-client on QEMU!
```

This proves TCP communication is working end-to-end.

---

## Part 9 — View the Qt GUI via VNC (from Windows)

### Step 9.1 — Start client with VNC platform (client VM)

```bash
QT_QPA_PLATFORM=vnc chat-client 192.168.10.2 &
```

Qt's built-in VNC server now listens on port 5900 inside the client VM.

### Step 9.2 — Forward the port from WSL2 (Terminal 3 in WSL2)

```bash
# Find your WSL2 IP first
ip addr show eth0 | grep "inet "   # e.g., 172.22.252.105

# Forward WSL2:5900 → QEMU client:5900
socat TCP-LISTEN:5900,fork TCP:192.168.10.3:5900
```

### Step 9.3 — Connect from Windows

1. Open **RealVNC Viewer**
2. Enter the WSL2 IP with port 5900: e.g., `172.22.252.105:5900`
3. Connect — the Qt chat GUI window will appear
4. Type a message and press Enter → it appears in the server journal

---

## Part 10 — Troubleshooting Reference

| Symptom | Diagnosis | Fix |
|---|---|---|
| `chat-server` service not starting | Qt can't find display platform | Add `Environment=QT_QPA_PLATFORM=offscreen` to service file |
| `chat-server.postinst returned 1` during build | `inherit systemd` calls `systemctl enable` in rootfs | Remove `inherit systemd`; pre-create the symlink in `do_install` |
| `Nothing PROVIDES 'chat-server'` in bitbake | Sourced `oe-init-build-env` from inside build dir | Run: `cd ~/yocto/poky && source oe-init-build-env build` |
| Port 12345 not in `/proc/net/tcp` | Qt binds IPv6 dual-stack | Check `/proc/net/tcp6` for hex `3039` |
| `runqemu` fails with SDL error | Yocto QEMU built without SDL | Always use `nographic` flag |
| VNC connection refused | socat not running, or wrong IP | Check socat is running in Terminal 3; verify WSL2 IP |
| journalctl output garbled/truncated | Default journalctl wraps to terminal width | Use `journalctl -o cat` |
| Client connects but no message on server | Race condition — network not ready | `ping 192.168.10.2` succeeds before running chat-client |

---

## Summary of What Each Part Proves

| Requirement | How it's demonstrated |
|---|---|
| Qt5 app built via Yocto | BitBake output: `chat-server` and `chat-client` binaries in rootfs |
| Server auto-starts on boot | `systemctl status chat-server` shows `active (running)` |
| TCP communication between VMs | Server journal shows client IP + received message |
| GUI visible from Windows | RealVNC Viewer renders the Qt window via VNC |
| Custom Yocto layer | `meta-chat` layer with proper `layer.conf`, recipes, and images |
