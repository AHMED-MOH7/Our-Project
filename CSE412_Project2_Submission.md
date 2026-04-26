# CSE412 — Project 2 Submission
## Embedded Linux: Qt5 TCP Chat Application on Yocto

| | |
|---|---|
| **Course** | CSE412 — Embedded Systems |
| **University** | Faculty of Engineering, Ain Shams University |
| **Student** | Ahmed Mohamed |
| **GitHub** | AHMED-MOH7 |
| **Date** | April 2026 |

---

# Phase 1 — Download & Build Yocto Project (7 Marks)

## 1.1 Setting Up the Yocto / OpenEmbedded Environment

### Clone Poky (Kirkstone)

```bash
mkdir -p ~/yocto && cd ~/yocto
git clone git://git.yoctoproject.org/poky -b kirkstone
```

### Initialize the Build Environment

```bash
cd ~/yocto/poky
source oe-init-build-env build
```

This creates `~/yocto/poky/build/` and sets `$BUILDDIR`. All `bitbake` commands must be run from this shell session.

---

## 1.2 Configure the Project and Choose a Target

Edit `~/yocto/poky/build/conf/local.conf`:

```
MACHINE ??= "qemux86-64"
DISTRO ?= "poky"
PACKAGE_CLASSES ?= "package_rpm"
BB_NUMBER_THREADS ?= "${@oe.utils.cpu_count()}"
PARALLEL_MAKE ?= "-j ${@oe.utils.cpu_count()}"
```

---

## 1.3 Add the Raspberry Pi Layer

```bash
cd ~/yocto
git clone git://git.yoctoproject.org/meta-raspberrypi -b kirkstone
```

Add it to `~/yocto/poky/build/conf/bblayers.conf`:

```
BBLAYERS ?= " \
  /home/hp/yocto/poky/meta \
  /home/hp/yocto/poky/meta-poky \
  /home/hp/yocto/poky/meta-yocto-bsp \
  /home/hp/yocto/meta-raspberrypi \
  "
```

> To build for Raspberry Pi, change `MACHINE = "raspberrypi4-64"` in `local.conf`. For this project we use `qemux86-64` for QEMU deployment.

---

## 1.4 Add the Qt5 Layer

```bash
cd ~/yocto
git clone https://github.com/meta-qt5/meta-qt5.git -b kirkstone
```

Add it to `bblayers.conf` (Qt5 depends on `meta-oe`, so `meta-openembedded` is also required):

```bash
cd ~/yocto
git clone https://github.com/openembedded/meta-openembedded.git -b kirkstone
```

Final `bblayers.conf`:

```
BBLAYERS ?= " \
  /home/hp/yocto/poky/meta \
  /home/hp/yocto/poky/meta-poky \
  /home/hp/yocto/poky/meta-yocto-bsp \
  /home/hp/yocto/meta-openembedded/meta-oe \
  /home/hp/yocto/meta-openembedded/meta-python \
  /home/hp/yocto/meta-qt5 \
  /home/hp/yocto/meta-raspberrypi \
  /home/hp/yocto/meta-chat \
  "
```

---

## 1.5 Qt Server/Client Chat Application Source Code

### Application Architecture

```
chat-server (192.168.10.2:12345)          chat-client (192.168.10.3)
┌─────────────────────────────┐           ┌─────────────────────────────┐
│  QTcpServer (port 12345)    │◄──TCP────►│  QTcpSocket                 │
│  Accepts multiple clients   │           │  Connects to 192.168.10.2   │
│  Broadcasts messages        │           │  Qt GUI + VNC platform      │
│  Headless via offscreen     │           │  Auto-sends test message    │
└─────────────────────────────┘           └─────────────────────────────┘
```

---

### `server.h`

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

    QTcpServer         *m_server;
    QList<QTcpSocket*>  m_clients;
    QTextEdit          *m_logView;
    QLabel             *m_statusLabel;
};
```

---

### `server.cpp`

```cpp
#include "server.h"
#include <QHostAddress>
#include <QVBoxLayout>
#include <QWidget>
#include <QDateTime>
#include <QDebug>

ChatServer::ChatServer(QWidget *parent) : QMainWindow(parent) {
    QWidget *central = new QWidget(this);
    setCentralWidget(central);
    QVBoxLayout *layout = new QVBoxLayout(central);

    m_statusLabel = new QLabel("Server: starting...", this);
    layout->addWidget(m_statusLabel);

    m_logView = new QTextEdit(this);
    m_logView->setReadOnly(true);
    layout->addWidget(m_logView);

    setWindowTitle("Qt Chat — Server");
    resize(500, 400);

    m_server = new QTcpServer(this);
    connect(m_server, &QTcpServer::newConnection,
            this, &ChatServer::onNewConnection);
    startServer(12345);
}

ChatServer::~ChatServer() {}

void ChatServer::startServer(quint16 port) {
    if (!m_server->listen(QHostAddress::Any, port)) {
        log("ERROR: Cannot listen on port " + QString::number(port) +
            ": " + m_server->errorString());
        m_statusLabel->setText("Server: FAILED to start");
        return;
    }
    m_statusLabel->setText("Server: listening on port " + QString::number(port));
    log("Server started on port " + QString::number(port));
}

void ChatServer::onNewConnection() {
    while (m_server->hasPendingConnections()) {
        QTcpSocket *socket = m_server->nextPendingConnection();
        m_clients << socket;
        connect(socket, &QTcpSocket::readyRead,
                this, &ChatServer::onReadyRead);
        connect(socket, &QTcpSocket::disconnected,
                this, &ChatServer::onClientDisconnected);
        log("Client connected: " + socket->peerAddress().toString() +
            ":" + QString::number(socket->peerPort()));
        m_statusLabel->setText("Server: " +
            QString::number(m_clients.size()) + " client(s) connected");
    }
}

void ChatServer::onReadyRead() {
    QTcpSocket *senderSocket = qobject_cast<QTcpSocket*>(sender());
    if (!senderSocket) return;
    while (senderSocket->canReadLine()) {
        QByteArray data = senderSocket->readLine();
        QString msg = QString::fromUtf8(data).trimmed();
        log("[" + senderSocket->peerAddress().toString() + "] " + msg);
        broadcast(data, senderSocket);
    }
}

void ChatServer::onClientDisconnected() {
    QTcpSocket *socket = qobject_cast<QTcpSocket*>(sender());
    if (!socket) return;
    log("Client disconnected: " + socket->peerAddress().toString());
    m_clients.removeAll(socket);
    socket->deleteLater();
    m_statusLabel->setText("Server: " +
        QString::number(m_clients.size()) + " client(s) connected");
}

void ChatServer::broadcast(const QByteArray &data, QTcpSocket *exclude) {
    for (QTcpSocket *client : m_clients) {
        if (client != exclude &&
            client->state() == QAbstractSocket::ConnectedState)
            client->write(data);
    }
}

void ChatServer::log(const QString &msg) {
    QString timestamp = QDateTime::currentDateTime().toString("hh:mm:ss");
    QString line = "[" + timestamp + "] " + msg;
    m_logView->append(line);
    qDebug().noquote() << line;   // captured by journalctl
}
```

---

### `main-server.cpp`

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

---

### `chat-server.pro`

```
TEMPLATE = app
TARGET   = chat-server
QT      += widgets network
SOURCES += main-server.cpp server.cpp
HEADERS += server.h
```

---

### `client.h`

```cpp
#pragma once
#include <QMainWindow>
#include <QTcpSocket>
#include <QTextEdit>
#include <QLineEdit>
#include <QPushButton>
#include <QLabel>

class ChatClient : public QMainWindow {
    Q_OBJECT
public:
    explicit ChatClient(QWidget *parent = nullptr);
    ~ChatClient();

private slots:
    void onConnected();
    void onDisconnected();
    void onReadyRead();
    void onErrorOccurred(QAbstractSocket::SocketError error);
    void sendMessage();
    void connectToServer();

private:
    void log(const QString &msg);

    QTcpSocket  *m_socket;
    QTextEdit   *m_logView;
    QLineEdit   *m_inputField;
    QLineEdit   *m_hostField;
    QPushButton *m_sendButton;
    QPushButton *m_connectButton;
    QLabel      *m_statusLabel;
};
```

---

### `client.cpp`

```cpp
#include "client.h"
#include <QHostAddress>
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QWidget>
#include <QDateTime>
#include <QTimer>
#include <QCoreApplication>
#include <QDebug>

ChatClient::ChatClient(QWidget *parent) : QMainWindow(parent) {
    QWidget *central = new QWidget(this);
    setCentralWidget(central);
    QVBoxLayout *mainLayout = new QVBoxLayout(central);

    m_statusLabel = new QLabel("Status: Disconnected", this);
    mainLayout->addWidget(m_statusLabel);

    QHBoxLayout *connectRow = new QHBoxLayout();
    m_hostField = new QLineEdit("192.168.10.2", this);
    m_hostField->setPlaceholderText("Server IP");
    m_connectButton = new QPushButton("Connect", this);
    connectRow->addWidget(m_hostField);
    connectRow->addWidget(m_connectButton);
    mainLayout->addLayout(connectRow);

    m_logView = new QTextEdit(this);
    m_logView->setReadOnly(true);
    mainLayout->addWidget(m_logView);

    QHBoxLayout *inputRow = new QHBoxLayout();
    m_inputField = new QLineEdit(this);
    m_inputField->setPlaceholderText("Type a message...");
    m_inputField->setEnabled(false);
    m_sendButton = new QPushButton("Send", this);
    m_sendButton->setEnabled(false);
    inputRow->addWidget(m_inputField);
    inputRow->addWidget(m_sendButton);
    mainLayout->addLayout(inputRow);

    setWindowTitle("Qt Chat — Client");
    resize(500, 400);

    m_socket = new QTcpSocket(this);
    connect(m_socket, &QTcpSocket::connected,
            this, &ChatClient::onConnected);
    connect(m_socket, &QTcpSocket::disconnected,
            this, &ChatClient::onDisconnected);
    connect(m_socket, &QTcpSocket::readyRead,
            this, &ChatClient::onReadyRead);
    connect(m_socket,
            QOverload<QAbstractSocket::SocketError>::of(
                &QAbstractSocket::errorOccurred),
            this, &ChatClient::onErrorOccurred);

    connect(m_connectButton, &QPushButton::clicked,
            this, &ChatClient::connectToServer);
    connect(m_sendButton, &QPushButton::clicked,
            this, &ChatClient::sendMessage);
    connect(m_inputField, &QLineEdit::returnPressed,
            this, &ChatClient::sendMessage);

    // Accept server IP as command-line argument
    QStringList args = QCoreApplication::arguments();
    if (args.size() >= 2)
        m_hostField->setText(args.at(1));

    // Auto-connect 1 second after launch
    QTimer::singleShot(1000, this, &ChatClient::connectToServer);
}

ChatClient::~ChatClient() {}

void ChatClient::connectToServer() {
    QString host = m_hostField->text().trimmed();
    if (host.isEmpty()) host = "192.168.10.2";
    log("Connecting to " + host + ":12345...");
    m_socket->connectToHost(host, 12345);
}

void ChatClient::onConnected() {
    m_statusLabel->setText("Status: Connected to " +
        m_socket->peerAddress().toString());
    m_inputField->setEnabled(true);
    m_sendButton->setEnabled(true);
    m_connectButton->setEnabled(false);
    log("Connected to server.");

    // Auto-send proof message
    QTimer::singleShot(500, this, [this]() {
        QByteArray data = QByteArray("Hello from chat-client on QEMU!\n");
        m_socket->write(data);
        log("[Me] Hello from chat-client on QEMU!");
    });
}

void ChatClient::onDisconnected() {
    m_statusLabel->setText("Status: Disconnected");
    m_inputField->setEnabled(false);
    m_sendButton->setEnabled(false);
    m_connectButton->setEnabled(true);
    log("Disconnected from server.");
}

void ChatClient::onReadyRead() {
    while (m_socket->canReadLine()) {
        QString msg = QString::fromUtf8(m_socket->readLine()).trimmed();
        log("[Remote] " + msg);
    }
}

void ChatClient::onErrorOccurred(QAbstractSocket::SocketError error) {
    Q_UNUSED(error);
    log("Socket error: " + m_socket->errorString());
    m_connectButton->setEnabled(true);
}

void ChatClient::sendMessage() {
    QString text = m_inputField->text().trimmed();
    if (text.isEmpty()) return;
    if (m_socket->state() != QAbstractSocket::ConnectedState) {
        log("Not connected.");
        return;
    }
    QByteArray data = (text + "\n").toUtf8();
    m_socket->write(data);
    log("[Me] " + text);
    m_inputField->clear();
}

void ChatClient::log(const QString &msg) {
    QString ts = QDateTime::currentDateTime().toString("hh:mm:ss");
    QString line = "[" + ts + "] " + msg;
    m_logView->append(line);
    qDebug().noquote() << line;   // captured by journalctl
}
```

---

### `main-client.cpp`

```cpp
#include <QApplication>
#include "client.h"

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    ChatClient window;
    window.show();
    return app.exec();
}
```

---

### `chat-client.pro`

```
TEMPLATE = app
TARGET   = chat-client
QT      += widgets network
SOURCES += main-client.cpp client.cpp
HEADERS += client.h
```

---

## 1.6 Phase 1 Deliverables

### `local.conf` (relevant section)

```
MACHINE ??= "qemux86-64"
DISTRO ?= "poky"
PACKAGE_CLASSES ?= "package_rpm"
BB_NUMBER_THREADS ?= "${@oe.utils.cpu_count()}"
PARALLEL_MAKE ?= "-j ${@oe.utils.cpu_count()}"
```

### `bblayers.conf`

```
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /home/hp/yocto/poky/meta \
  /home/hp/yocto/poky/meta-poky \
  /home/hp/yocto/poky/meta-yocto-bsp \
  /home/hp/yocto/meta-openembedded/meta-oe \
  /home/hp/yocto/meta-openembedded/meta-python \
  /home/hp/yocto/meta-qt5 \
  /home/hp/yocto/meta-raspberrypi \
  /home/hp/yocto/meta-chat \
  "
```

---

---

# Phase 2 — Build the Image for Each Application (8 Marks)

## 2.1 Create a New Yocto Layer — `meta-chat`

```bash
cd ~/yocto/poky
source oe-init-build-env build
cd ~/yocto
bitbake-layers create-layer meta-chat
```

### Layer Folder Structure

```
meta-chat/
├── conf/
│   └── layer.conf
└── recipes-chat/
    ├── chat-server/
    │   ├── chat-server_1.0.bb
    │   └── files/
    │       ├── server.cpp
    │       ├── server.h
    │       ├── main-server.cpp
    │       ├── chat-server.pro
    │       └── chat-server.service       ← systemd unit (BONUS)
    ├── chat-client/
    │   ├── chat-client_1.0.bb
    │   └── files/
    │       ├── client.cpp
    │       ├── client.h
    │       ├── main-client.cpp
    │       └── chat-client.pro
    └── images/
        ├── chat-server-image.bb
        └── chat-client-image.bb
```

### `conf/layer.conf`

```bitbake
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-chat"
BBFILE_PATTERN_meta-chat = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-chat = "6"

LAYERDEPENDS_meta-chat = "core qtbase"
LAYERSERIES_COMPAT_meta-chat = "kirkstone"
```

---

## 2.2 Interface the Custom Layer to the Existing Yocto Project

The layer was added to `bblayers.conf` (shown in Phase 1 deliverables):

```
/home/hp/yocto/meta-chat \
```

Verify it is recognized:

```bash
bitbake-layers show-layers
```

Expected output includes:
```
layer                 path                                    priority
==========================================================================
meta-chat             /home/hp/yocto/meta-chat                6
```

---

## 2.3 Add the Client Application to the Layer

### `chat-client_1.0.bb`

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

---

## 2.4 Configure and Build the Client Image

### `chat-client-image.bb`

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

Build:

```bash
cd ~/yocto/poky
source oe-init-build-env build
bitbake chat-client-image
```

---

## 2.5 Configure and Build the Server Image

### `chat-server_1.0.bb`

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

    # BONUS: pre-create enable symlink so server auto-starts on boot
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

### `chat-server.service` (BONUS — auto-start on boot)

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

> `QT_QPA_PLATFORM=offscreen` allows Qt5 to initialize without a physical display, which is required for a headless systemd service.

### `chat-server-image.bb`

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

Build:

```bash
bitbake chat-server-image
```

---

## 2.6 Deploy Each Image on a QEMU Instance

Confirm images were built:

```bash
ls ~/yocto/poky/build/tmp/deploy/images/qemux86-64/*.ext4
# chat-server-image-qemux86-64-*.rootfs.ext4
# chat-client-image-qemux86-64-*.rootfs.ext4
```

### Terminal 1 — Server QEMU

```bash
cd ~/yocto/poky && source oe-init-build-env build

runqemu chat-server-image nographic \
  qemuparams="-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
              -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:02"
```

Log in as `root`, then configure the network:

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

Log in as `root`, configure the network:

```bash
ip addr add 192.168.10.3/24 dev eth0
ip link set eth0 up
```

---

## 2.7 Connect the Two QEMU Instances

Both VMs are connected through a Linux bridge (`br0`) on the WSL2 host.

### Bridge Setup (run once in WSL2)

```bash
sudo ip link add br0 type bridge
sudo ip addr add 192.168.10.1/24 dev br0
sudo ip link set br0 up

sudo ip tuntap add tap0 mode tap
sudo ip link set tap0 master br0
sudo ip link set tap0 up

sudo ip tuntap add tap1 mode tap
sudo ip link set tap1 master br0
sudo ip link set tap1 up
```

### Network Topology

```
WSL2 Host (br0 — 192.168.10.1)
        │
   ┌────┴────┐
  tap0      tap1
   │          │
QEMU Server  QEMU Client
192.168.10.2  192.168.10.3
```

### Verify Connectivity

From the client VM:
```bash
ping -c 3 192.168.10.2
```

---

## 2.8 Proof of Working Communication

### Server Side — Watch logs

```bash
# In server QEMU terminal
journalctl -fu chat-server -o cat | grep -v "QFont\|Qt no longer\|This plugin"
```

### Client Side — Connect and send

```bash
# In client QEMU terminal
QT_QPA_PLATFORM=offscreen chat-client 192.168.10.2
```

### Expected Output

**Server journal:**
```
[10:32:01] Server started on port 12345
[10:32:15] Client connected: ::ffff:192.168.10.3:54321
[10:32:15] [::ffff:192.168.10.3] Hello from chat-client on QEMU!
```

**Client terminal:**
```
[10:32:14] Connecting to 192.168.10.2:12345...
[10:32:15] Connected to server.
[10:32:15] [Me] Hello from chat-client on QEMU!
```

---

## BONUS 1 — Kernel Auto-starts the Server Application

This is implemented via systemd. The `chat-server.service` unit is placed under `multi-user.target.wants/` during `do_install`, so the server binary starts automatically on every boot without any manual intervention.

Verify inside the server VM:

```bash
systemctl status chat-server
```

Expected output:
```
● chat-server.service - Qt5 TCP Chat Server
     Loaded: loaded (/lib/systemd/system/chat-server.service; enabled)
     Active: active (running) since ...
```

The server is already listening by the time the user reaches a login prompt.

---

## BONUS 2 — Track Traffic with Wireshark

To capture the TCP traffic between the two QEMU instances from the WSL2 host:

```bash
# Install tshark (CLI Wireshark) in WSL2
sudo apt install tshark -y

# Capture on the bridge interface, filter for port 12345
sudo tshark -i br0 -f "tcp port 12345" -w /tmp/chat_capture.pcap
```

After the demo, copy the `.pcap` file to Windows and open it in Wireshark:

```bash
cp /tmp/chat_capture.pcap /mnt/c/Users/Hp/chat_capture.pcap
```

Open `C:\Users\Hp\chat_capture.pcap` in Wireshark on Windows.

What to observe in Wireshark:
- **TCP 3-way handshake**: SYN (client → server), SYN-ACK (server → client), ACK
- **TCP data segments**: containing the UTF-8 text `Hello from chat-client on QEMU!\n`
- **TCP FIN/RST**: when the connection closes

---

## Phase 2 Deliverable — Layer Folder

The complete `meta-chat` layer submitted for grading:

```
meta-chat/
├── conf/
│   └── layer.conf
└── recipes-chat/
    ├── chat-server/
    │   ├── chat-server_1.0.bb
    │   └── files/
    │       ├── server.cpp
    │       ├── server.h
    │       ├── main-server.cpp
    │       ├── chat-server.pro
    │       └── chat-server.service
    ├── chat-client/
    │   ├── chat-client_1.0.bb
    │   └── files/
    │       ├── client.cpp
    │       ├── client.h
    │       ├── main-client.cpp
    │       └── chat-client.pro
    └── images/
        ├── chat-server-image.bb
        └── chat-client-image.bb
```

---

## Summary Table

| Requirement | Status | Evidence |
|---|---|---|
| Yocto environment set up | Done | `poky` cloned, `oe-init-build-env` sourced |
| Target configured (qemux86-64) | Done | `local.conf` |
| meta-raspberrypi layer added | Done | `bblayers.conf` |
| meta-qt5 layer added | Done | `bblayers.conf` |
| Qt server/client application | Done | `server.cpp`, `client.cpp` |
| Socket communication (TCP) | Done | `QTcpServer` / `QTcpSocket` on port 12345 |
| Custom `meta-chat` layer created | Done | Layer folder structure |
| Layer interfaced to Yocto | Done | `bblayers.conf` |
| Client added to layer | Done | `chat-client_1.0.bb` |
| Client image built | Done | `chat-client-image.bb` + `bitbake` |
| Server image built | Done | `chat-server-image.bb` + `bitbake` |
| Deployed on QEMU instances | Done | `runqemu` with TAP networking |
| Two instances connected | Done | Linux bridge `br0` + tap0/tap1 |
| **BONUS:** Auto-start server | **Done** | `systemd` + pre-created enable symlink |
| **BONUS:** Wireshark traffic | **Done** | `tshark` capture on `br0` |
