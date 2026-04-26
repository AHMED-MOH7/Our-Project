# CSE412 — Embedded OS: Yocto Project Implementation Plan

**Environment:** WSL2 — Ubuntu 24.04  
**Target board:** Raspberry Pi (via meta-raspberrypi, kirkstone branch)  
**Qt layer:** meta-qt5 (kirkstone branch)  
**Yocto release:** Kirkstone (LTS)

---

## Prerequisites & WSL Setup

```bash
# Install required host packages (inside WSL)
sudo apt update && sudo apt install -y \
  gawk wget git-core diffstat unzip texinfo gcc-multilib \
  build-essential chrpath socat cpio python3 python3-pip \
  python3-pexpect xz-utils debianutils iputils-ping python3-git \
  python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3 xterm \
  python3-subunit mesa-common-dev zstd liblz4-tool file

# Recommended: increase WSL memory in %USERPROFILE%\.wslconfig
# [wsl2]
# memory=8GB
# swap=4GB
# processors=4
```

---

## Phase 1 — Download & Build Yocto (7 marks)

### Step 1 — Clone Poky (Yocto core)

```bash
mkdir -p ~/yocto && cd ~/yocto
git clone git://git.yoctoproject.org/poky -b kirkstone
cd poky
```

### Step 2 — Add meta-raspberrypi layer

```bash
cd ~/yocto
git clone git://git.yoctoproject.org/meta-raspberrypi -b kirkstone
```

### Step 3 — Add meta-qt5 layer

```bash
cd ~/yocto
git clone https://github.com/meta-qt5/meta-qt5.git -b kirkstone
```

### Step 4 — Initialize the build environment

```bash
cd ~/yocto/poky
source oe-init-build-env build
# This creates ~/yocto/poky/build/ and sets shell env vars
```

### Step 5 — Configure bblayers.conf

Edit `~/yocto/poky/build/conf/bblayers.conf`:

```
BBLAYERS ?= " \
  /home/<user>/yocto/poky/meta \
  /home/<user>/yocto/poky/meta-poky \
  /home/<user>/yocto/poky/meta-yocto-bsp \
  /home/<user>/yocto/meta-raspberrypi \
  /home/<user>/yocto/meta-qt5 \
  "
```

> Replace `<user>` with your WSL username (`echo $USER`).

### Step 6 — Configure local.conf

Edit `~/yocto/poky/build/conf/local.conf` — key settings:

```
# Target machine
MACHINE = "raspberrypi4-64"

# Parallel build tuning (match your CPU core count)
BB_NUMBER_THREADS = "4"
PARALLEL_MAKE = "-j 4"

# Enable Qt5 and OpenGL
DISTRO_FEATURES:append = " opengl"
IMAGE_INSTALL:append = " qtbase qtdeclarative"

# Enable SSH for deployment convenience
IMAGE_FEATURES += "ssh-server-openssh"

# Set preferred Qt version
PREFERRED_VERSION_qtbase = "5.15%"
```

### Step 7 — Write the Qt Chat Application (C++ / Qt5 Sockets)

#### File layout

```
~/yocto/qt-chat/
├── server/
│   ├── main.cpp
│   ├── server.cpp
│   ├── server.h
│   └── server.pro
└── client/
    ├── main.cpp
    ├── client.cpp
    ├── client.h
    └── client.pro
```

#### server/server.h

```cpp
#pragma once
#include <QTcpServer>
#include <QTcpSocket>

class ChatServer : public QTcpServer {
    Q_OBJECT
public:
    explicit ChatServer(QObject *parent = nullptr);
    void startServer(quint16 port);
protected:
    void incomingConnection(qintptr socketDescriptor) override;
private slots:
    void onReadyRead();
    void onClientDisconnected();
private:
    QList<QTcpSocket*> m_clients;
};
```

#### server/server.cpp

```cpp
#include "server.h"
#include <QDebug>

ChatServer::ChatServer(QObject *parent) : QTcpServer(parent) {}

void ChatServer::startServer(quint16 port) {
    if (!listen(QHostAddress::Any, port))
        qFatal("Cannot listen on port %d", port);
    qDebug() << "Server listening on port" << port;
}

void ChatServer::incomingConnection(qintptr descriptor) {
    auto *socket = new QTcpSocket(this);
    socket->setSocketDescriptor(descriptor);
    m_clients << socket;
    connect(socket, &QTcpSocket::readyRead, this, &ChatServer::onReadyRead);
    connect(socket, &QTcpSocket::disconnected, this, &ChatServer::onClientDisconnected);
    qDebug() << "Client connected:" << socket->peerAddress().toString();
}

void ChatServer::onReadyRead() {
    auto *sender = qobject_cast<QTcpSocket*>(QObject::sender());
    QByteArray data = sender->readAll();
    qDebug() << "Received:" << data;
    for (auto *client : m_clients)
        if (client != sender)
            client->write(data);
}

void ChatServer::onClientDisconnected() {
    auto *socket = qobject_cast<QTcpSocket*>(QObject::sender());
    m_clients.removeAll(socket);
    socket->deleteLater();
}
```

#### server/main.cpp

```cpp
#include <QCoreApplication>
#include "server.h"

int main(int argc, char *argv[]) {
    QCoreApplication app(argc, argv);
    ChatServer server;
    server.startServer(12345);
    return app.exec();
}
```

#### server/server.pro

```
QT += core network
TARGET = chat-server
SOURCES += main.cpp server.cpp
HEADERS += server.h
```

#### client/client.h

```cpp
#pragma once
#include <QTcpSocket>

class ChatClient : public QObject {
    Q_OBJECT
public:
    explicit ChatClient(QObject *parent = nullptr);
    void connectToServer(const QString &host, quint16 port);
    void sendMessage(const QString &msg);
private slots:
    void onReadyRead();
    void onConnected();
private:
    QTcpSocket *m_socket;
};
```

#### client/client.cpp

```cpp
#include "client.h"
#include <QDebug>
#include <QTextStream>

ChatClient::ChatClient(QObject *parent) : QObject(parent) {
    m_socket = new QTcpSocket(this);
    connect(m_socket, &QTcpSocket::readyRead, this, &ChatClient::onReadyRead);
    connect(m_socket, &QTcpSocket::connected, this, &ChatClient::onConnected);
}

void ChatClient::connectToServer(const QString &host, quint16 port) {
    m_socket->connectToHost(host, port);
}

void ChatClient::sendMessage(const QString &msg) {
    m_socket->write(msg.toUtf8());
}

void ChatClient::onConnected() {
    qDebug() << "Connected to server";
}

void ChatClient::onReadyRead() {
    qDebug() << "Message:" << m_socket->readAll();
}
```

#### client/main.cpp

```cpp
#include <QCoreApplication>
#include "client.h"

int main(int argc, char *argv[]) {
    QCoreApplication app(argc, argv);
    ChatClient client;
    client.connectToServer("192.168.7.1", 12345);   // QEMU host IP
    client.sendMessage("Hello from client!\n");
    return app.exec();
}
```

#### client/client.pro

```
QT += core network
TARGET = chat-client
SOURCES += main.cpp client.cpp
HEADERS += client.h
```

---

## Phase 2 — Build Custom Yocto Layer (8 marks)

### Step 1 — Create the custom layer skeleton

```bash
cd ~/yocto/poky
source oe-init-build-env build
bitbake-layers create-layer ../../meta-chat
# Results in ~/yocto/meta-chat/
```

### Step 2 — Layer directory structure

```
meta-chat/
├── conf/
│   └── layer.conf
├── recipes-chat/
│   ├── chat-server/
│   │   ├── chat-server_1.0.bb
│   │   └── files/
│   │       ├── server/          ← server source files
│   │       └── chat-server.service   ← systemd unit (bonus)
│   └── chat-client/
│       ├── chat-client_1.0.bb
│       └── files/
│           └── client/          ← client source files
└── recipes-images/
    ├── images/
    │   ├── chat-server-image.bb
    │   └── chat-client-image.bb
```

### Step 3 — conf/layer.conf

```
BBPATH .= ":${LAYERDIR}"
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb"
BBFILE_COLLECTIONS += "chat"
BBFILE_PATTERN_chat = "^${LAYERDIR}/"
BBFILE_PRIORITY_chat = "6"
LAYERSERIES_COMPAT_chat = "kirkstone"
```

### Step 4 — chat-server_1.0.bb

```bitbake
SUMMARY = "Qt5 TCP Chat Server"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

DEPENDS = "qtbase"

SRC_URI = "file://server/"
S = "${WORKDIR}/server"

inherit qmake5 systemd

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/chat-server ${D}${bindir}/chat-server
}

# BONUS: systemd auto-start
SYSTEMD_SERVICE:${PN} = "chat-server.service"
FILES:${PN} += "${systemd_system_unitdir}/chat-server.service"
```

### Step 5 — chat-client_1.0.bb

```bitbake
SUMMARY = "Qt5 TCP Chat Client"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

DEPENDS = "qtbase"

SRC_URI = "file://client/"
S = "${WORKDIR}/client"

inherit qmake5

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/chat-client ${D}${bindir}/chat-client
}
```

### Step 6 — Image recipes

**chat-server-image.bb**
```bitbake
require recipes-core/images/core-image-minimal.bb
IMAGE_INSTALL += "chat-server qtbase"
IMAGE_FEATURES += "ssh-server-openssh"
```

**chat-client-image.bb**
```bitbake
require recipes-core/images/core-image-minimal.bb
IMAGE_INSTALL += "chat-client qtbase"
IMAGE_FEATURES += "ssh-server-openssh"
```

### Step 7 — Add meta-chat to bblayers.conf

```
BBLAYERS ?= " \
  ...existing layers...
  /home/<user>/yocto/meta-chat \
  "
```

### Step 8 — Build both images

```bash
cd ~/yocto/poky
source oe-init-build-env build

bitbake chat-server-image
bitbake chat-client-image
```

Images land in `build/tmp/deploy/images/qemuarm64/` (or `qemux86-64` if you switch MACHINE for QEMU).

> **Tip:** Use `MACHINE = "qemux86-64"` in local.conf for faster QEMU builds during development, then switch to `raspberrypi4-64` for the final image.

---

## Phase 2 — Deploy on QEMU & Connect the Two Instances

### Launch server image

```bash
# Terminal 1
runqemu qemux86-64 chat-server-image nographic \
  qemuparams="-net nic -net tap,ifname=tap0,script=no,downscript=no"
```

### Launch client image

```bash
# Terminal 2
runqemu qemux86-64 chat-client-image nographic \
  qemuparams="-net nic -net tap,ifname=tap1,script=no,downscript=no"
```

### Network bridging (connect the two QEMU instances)

```bash
# On WSL host — create a bridge
sudo ip link add name br0 type bridge
sudo ip link set br0 up
sudo ip tuntap add tap0 mode tap
sudo ip tuntap add tap1 mode tap
sudo ip link set tap0 master br0
sudo ip link set tap1 master br0
sudo ip link set tap0 up
sudo ip link set tap1 up
sudo ip addr add 192.168.10.1/24 dev br0
```

Inside server QEMU: `ip addr add 192.168.10.2/24 dev eth0`  
Inside client QEMU: `ip addr add 192.168.10.3/24 dev eth0`  
Client connects to `192.168.10.2:12345`.

---

## BONUS Tasks

### Auto-start server via systemd

Create `meta-chat/recipes-chat/chat-server/files/chat-server.service`:

```ini
[Unit]
Description=Qt Chat Server
After=network.target

[Service]
ExecStart=/usr/bin/chat-server
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable in the recipe with `SYSTEMD_SERVICE:${PN} = "chat-server.service"` (already shown above).

### Wireshark traffic capture

```bash
# On WSL host — capture on the bridge interface
sudo tshark -i br0 -f "tcp port 12345" -w ~/chat_traffic.pcap
# Open chat_traffic.pcap in Wireshark on Windows
```

---

## Deliverables Checklist

| Item | Location |
|------|----------|
| `local.conf` | `build/conf/local.conf` |
| `bblayers.conf` | `build/conf/bblayers.conf` |
| Server source code | `meta-chat/recipes-chat/chat-server/files/server/` |
| Client source code | `meta-chat/recipes-chat/chat-client/files/client/` |
| Custom layer folder | `meta-chat/` |
| QEMU network setup script | `~/yocto/qemu-net.sh` |
| Wireshark capture (bonus) | `~/chat_traffic.pcap` |

---

## Timeline Estimate

| Day | Work |
|-----|------|
| 1 | WSL setup + clone poky, meta-raspberrypi, meta-qt5 |
| 2 | Configure bblayers.conf & local.conf; first `bitbake core-image-minimal` to verify toolchain |
| 3–4 | Write & locally test Qt server/client (compile on host with `qmake`) |
| 5 | Create meta-chat layer; write .bb recipes |
| 6 | `bitbake chat-server-image` + `bitbake chat-client-image` |
| 7 | QEMU deployment + network bridge; test end-to-end chat |
| 8 | Bonus (systemd auto-start, Wireshark); prepare slides & report |

---

## Useful Commands

```bash
# Check available machines
ls meta-raspberrypi/conf/machine/

# Inspect a recipe
bitbake -e chat-server | grep "^S ="

# Clean and rebuild a single recipe
bitbake -c cleanall chat-server && bitbake chat-server

# Open devshell for debugging
bitbake -c devshell chat-server

# Check layer dependencies
bitbake-layers show-layers
bitbake-layers show-recipes | grep chat
```
