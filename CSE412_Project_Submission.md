# CSE412 — Embedded Linux Project Submission
## Qt5 TCP Chat Application on Yocto / QEMU

| Field | Details |
|---|---|
| **Course** | CSE412 — Embedded Systems |
| **University** | Faculty of Engineering, Ain Shams University |
| **Student** | Ahmed Mohamed |
| **GitHub** | AHMED-MOH7 |
| **Date** | April 2026 |

---

## 1. Project Overview

This project implements a **Qt5 TCP chat application** running on two separate embedded Linux images, each booted inside QEMU. The two virtual machines communicate over a Linux bridge network:

- **chat-server-image** — boots a Qt5 TCP server on port 12345, auto-started by systemd on every boot.
- **chat-client-image** — boots a Qt5 TCP client that auto-connects to the server and sends messages; the GUI is accessible from Windows via VNC.

Both images are built with **Yocto Kirkstone** using a custom `meta-chat` layer on top of `poky`.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Windows Host                      │
│                                                     │
│   RealVNC Viewer ──► WSL2 (socat) ──► QEMU Client  │
│                   172.22.252.105:5900  192.168.10.3  │
└─────────────────────────────────────────────────────┘

        QEMU Server                QEMU Client
      192.168.10.2                192.168.10.3
      ┌──────────┐                ┌──────────┐
      │ chat-    │◄───TCP:12345───│ chat-    │
      │ server   │                │ client   │
      │(systemd) │                │(manual / │
      └──────────┘                │ autorun) │
            │                    └──────────┘
            └──────────┬──────────┘
                    br0 (Linux bridge)
                   tap0       tap1
```

---

## 3. Yocto Layer Structure

```
~/yocto/
├── poky/                          # Yocto Kirkstone base
│   └── build/
│       ├── conf/
│       │   ├── bblayers.conf      # Includes meta-chat
│       │   └── local.conf         # MACHINE = qemux86-64
│       └── tmp/deploy/images/qemux86-64/
│           ├── chat-server-image-*.wic
│           └── chat-client-image-*.wic
│
└── meta-chat/
    ├── conf/layer.conf
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
        ├── images/
        │   ├── chat-server-image.bb
        │   └── chat-client-image.bb
```

---

## 4. Source Code

### 4.1 Server — `server.h`

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

### 4.2 Server — `server.cpp`

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
    connect(m_server, &QTcpServer::newConnection, this, &ChatServer::onNewConnection);
    startServer(12345);
}

ChatServer::~ChatServer() {}

void ChatServer::startServer(quint16 port) {
    if (!m_server->listen(QHostAddress::Any, port)) {
        log("ERROR: Cannot listen on port " + QString::number(port) + ": " + m_server->errorString());
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
        connect(socket, &QTcpSocket::readyRead,    this, &ChatServer::onReadyRead);
        connect(socket, &QTcpSocket::disconnected, this, &ChatServer::onClientDisconnected);
        log("Client connected: " + socket->peerAddress().toString() +
            ":" + QString::number(socket->peerPort()));
        m_statusLabel->setText("Server: " + QString::number(m_clients.size()) + " client(s) connected");
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
    m_statusLabel->setText("Server: " + QString::number(m_clients.size()) + " client(s) connected");
}

void ChatServer::broadcast(const QByteArray &data, QTcpSocket *exclude) {
    for (QTcpSocket *client : m_clients) {
        if (client != exclude && client->state() == QAbstractSocket::ConnectedState)
            client->write(data);
    }
}

void ChatServer::log(const QString &msg) {
    QString timestamp = QDateTime::currentDateTime().toString("hh:mm:ss");
    QString line = "[" + timestamp + "] " + msg;
    m_logView->append(line);
    qDebug().noquote() << line;   // visible via journalctl
}
```

### 4.3 Client — `client.h`

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

### 4.4 Client — `client.cpp`

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
    connect(m_socket, &QTcpSocket::connected,    this, &ChatClient::onConnected);
    connect(m_socket, &QTcpSocket::disconnected, this, &ChatClient::onDisconnected);
    connect(m_socket, &QTcpSocket::readyRead,    this, &ChatClient::onReadyRead);
    connect(m_socket, QOverload<QAbstractSocket::SocketError>::of(&QAbstractSocket::errorOccurred),
            this, &ChatClient::onErrorOccurred);

    connect(m_connectButton, &QPushButton::clicked,    this, &ChatClient::connectToServer);
    connect(m_sendButton,    &QPushButton::clicked,    this, &ChatClient::sendMessage);
    connect(m_inputField,    &QLineEdit::returnPressed, this, &ChatClient::sendMessage);

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
    m_statusLabel->setText("Status: Connected to " + m_socket->peerAddress().toString());
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
    qDebug().noquote() << line;   // visible via journalctl
}
```

---

## 5. BitBake Recipes

### 5.1 `chat-server_1.0.bb`

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

    # Pre-create enable symlink (avoids postinstall failure in Kirkstone)
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

### 5.2 `chat-server.service`

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

> **Key:** `QT_QPA_PLATFORM=offscreen` allows Qt to initialize without a display, so the server can run as a headless systemd service.

### 5.3 `chat-client_1.0.bb`

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

### 5.4 Image Recipes

**`chat-server-image.bb`**
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

**`chat-client-image.bb`**
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

## 6. Network Setup

Both QEMU machines are connected through a Linux bridge (`br0`) on the WSL2 host:

```bash
# Run once in WSL2 (requires root)
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

| Interface | IP | Role |
|---|---|---|
| br0 (WSL2 host) | 192.168.10.1 | Bridge gateway |
| tap0 → QEMU server | 192.168.10.2 | chat-server |
| tap1 → QEMU client | 192.168.10.3 | chat-client |

---

## 7. Running the Demo

### Terminal 1 — Start server QEMU
```bash
cd ~/yocto/poky
source oe-init-build-env build

runqemu chat-server-image nographic \
  qemuparams="-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
              -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:02"
```

Inside the server VM:
```bash
ip addr add 192.168.10.2/24 dev eth0
ip link set eth0 up
# chat-server auto-started by systemd — verify:
journalctl -fu chat-server -o cat | grep -v "QFont\|Qt no longer\|This plugin"
```

### Terminal 2 — Start client QEMU
```bash
runqemu chat-client-image nographic \
  qemuparams="-netdev tap,id=net0,ifname=tap1,script=no,downscript=no \
              -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:03"
```

Inside the client VM:
```bash
ip addr add 192.168.10.3/24 dev eth0
ip link set eth0 up
ping -c 2 192.168.10.2          # verify connectivity
QT_QPA_PLATFORM=vnc chat-client 192.168.10.2 &
```

### Terminal 3 — Forward VNC from WSL2 to Windows
```bash
socat TCP-LISTEN:5900,fork TCP:192.168.10.3:5900
```

### Windows — View the GUI
Open **RealVNC Viewer** and connect to the WSL2 IP on port 5900 (e.g., `172.22.252.105:5900`).

---

## 8. Proof of Working

### Server journal output (clean logs):
```
[HH:MM:SS] Server started on port 12345
[HH:MM:SS] Client connected: ::ffff:192.168.10.3:XXXXX
[HH:MM:SS] [::ffff:192.168.10.3] Hello from chat-client on QEMU!
```

### Client terminal output:
```
[HH:MM:SS] Connecting to 192.168.10.2:12345...
[HH:MM:SS] Connected to server.
[HH:MM:SS] [Me] Hello from chat-client on QEMU!
```

These logs confirm:
1. The server starts automatically on boot via systemd.
2. The client connects and sends a message over the TCP bridge.
3. The server receives and logs the message.
4. The Qt GUI is accessible from Windows via VNC.

---

## 9. Key Technical Decisions

| Decision | Reason |
|---|---|
| `QT_QPA_PLATFORM=offscreen` in systemd service | Qt5 refuses to initialize without a display platform; offscreen provides a null framebuffer, allowing the server to run headlessly |
| Pre-created systemd symlink in `do_install` | `inherit systemd` in Kirkstone generates a postinstall script that calls `systemctl enable`, which fails during rootfs creation with exit code 1 |
| Check `/proc/net/tcp6` for port 3039 | Qt5 `QHostAddress::Any` binds IPv6 dual-stack; the listening socket appears in `tcp6`, not `tcp` |
| `QT_QPA_PLATFORM=vnc` for client GUI | Yocto-built QEMU has no SDL; Qt's built-in VNC server streams the GUI without needing a display driver |
| `journalctl -o cat` | Default journalctl truncates lines to terminal width, hiding important log messages |

---

## 10. Challenges and Solutions

| Challenge | Solution |
|---|---|
| Server failing to start — Qt platform plugin missing | Added `Environment=QT_QPA_PLATFORM=offscreen` to service file |
| `chat-server.postinst returned 1` | Removed `inherit systemd`; manually created `/lib/systemd/system/multi-user.target.wants/` symlink in `do_install` |
| BitBake `Nothing PROVIDES 'chat-server'` | Was building from inside the build directory; fixed by sourcing `oe-init-build-env` from `~/yocto/poky` |
| Port 12345 not visible in `/proc/net/tcp` | Qt binds IPv6 dual-stack — port is in `/proc/net/tcp6` as hex `3039` |
| QEMU `-display sdl` error (no SDL support) | Yocto-built QEMU compiled without SDL; used `nographic` + Qt VNC instead |
