# CSE412 Project 2 — Complete Command Reference
## From Zero to 100% Tested

> Run every block in order. Do not skip steps.
> All commands run inside **WSL2 Ubuntu 24.04** unless stated otherwise.

---

## PART 0 — Install Host Dependencies

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  gawk wget git diffstat unzip texinfo gcc build-essential \
  chrpath socat cpio python3 python3-pip python3-pexpect \
  xz-utils debianutils iputils-ping python3-git python3-jinja2 \
  libegl1-mesa libsdl1.2-dev pylint xterm python3-subunit \
  mesa-common-dev zstd liblz4-tool file lz4 \
  iproute2 net-tools
```

---

## PART 1 — Clone All Required Repositories

```bash
mkdir -p ~/yocto && cd ~/yocto
```

```bash
git clone git://git.yoctoproject.org/poky -b kirkstone
```

```bash
git clone https://github.com/openembedded/meta-openembedded.git -b kirkstone
```

```bash
git clone https://github.com/meta-qt5/meta-qt5.git -b kirkstone
```

```bash
git clone git://git.yoctoproject.org/meta-raspberrypi -b kirkstone
```

Verify all four folders exist:

```bash
ls ~/yocto
# Expected: poky  meta-openembedded  meta-qt5  meta-raspberrypi
```

---

## PART 2 — Initialize the Build Environment

```bash
cd ~/yocto/poky
source oe-init-build-env build
```

> You are now inside `~/yocto/poky/build/`. This shell session has `bitbake` available. Every time you open a new terminal you must re-run `source oe-init-build-env build` from `~/yocto/poky`.

---

## PART 3 — Configure `local.conf`

```bash
cat >> ~/yocto/poky/build/conf/local.conf << 'EOF'

MACHINE ??= "qemux86-64"
DISTRO ?= "poky"
PACKAGE_CLASSES ?= "package_rpm"
BB_NUMBER_THREADS ?= "${@oe.utils.cpu_count()}"
PARALLEL_MAKE ?= "-j ${@oe.utils.cpu_count()}"
EOF
```

---

## PART 4 — Configure `bblayers.conf`

Replace the entire bblayers.conf:

```bash
cat > ~/yocto/poky/build/conf/bblayers.conf << 'EOF'
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
EOF
```

---

## PART 5 — Create the `meta-chat` Layer Structure

```bash
cd ~/yocto/poky
source oe-init-build-env build
cd ~/yocto
bitbake-layers create-layer meta-chat
```

Create all required directories:

```bash
mkdir -p ~/yocto/meta-chat/recipes-chat/chat-server/files
mkdir -p ~/yocto/meta-chat/recipes-chat/chat-client/files
mkdir -p ~/yocto/meta-chat/recipes-chat/images
```

---

## PART 6 — Write `layer.conf`

```bash
cat > ~/yocto/meta-chat/conf/layer.conf << 'EOF'
BBPATH .= ":${LAYERDIR}"

BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-chat"
BBFILE_PATTERN_meta-chat = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-chat = "6"

LAYERDEPENDS_meta-chat = "core"
LAYERSERIES_COMPAT_meta-chat = "kirkstone"
EOF
```

---

## PART 7 — Write Server Source Files

### `server.h`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-server/files/server.h << 'EOF'
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
EOF
```

### `server.cpp`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-server/files/server.cpp << 'EOF'
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

    setWindowTitle("Qt Chat -- Server");
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
    qDebug().noquote() << line;
}
EOF
```

### `main-server.cpp`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-server/files/main-server.cpp << 'EOF'
#include <QApplication>
#include "server.h"

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    ChatServer window;
    window.show();
    return app.exec();
}
EOF
```

### `chat-server.pro`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-server/files/chat-server.pro << 'EOF'
TEMPLATE = app
TARGET   = chat-server
QT      += widgets network
SOURCES += main-server.cpp server.cpp
HEADERS += server.h
EOF
```

---

## PART 8 — Write the Systemd Service File

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-server/files/chat-server.service << 'EOF'
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
EOF
```

---

## PART 9 — Write Client Source Files

### `client.h`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-client/files/client.h << 'EOF'
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
EOF
```

### `client.cpp`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-client/files/client.cpp << 'EOF'
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

    setWindowTitle("Qt Chat -- Client");
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

    QStringList args = QCoreApplication::arguments();
    if (args.size() >= 2)
        m_hostField->setText(args.at(1));

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
    qDebug().noquote() << line;
}
EOF
```

### `main-client.cpp`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-client/files/main-client.cpp << 'EOF'
#include <QApplication>
#include "client.h"

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    ChatClient window;
    window.show();
    return app.exec();
}
EOF
```

### `chat-client.pro`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-client/files/chat-client.pro << 'EOF'
TEMPLATE = app
TARGET   = chat-client
QT      += widgets network
SOURCES += main-client.cpp client.cpp
HEADERS += client.h
EOF
```

---

## PART 10 — Write BitBake Recipes

### `chat-server_1.0.bb`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-server/chat-server_1.0.bb << 'EOF'
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
EOF
```

### `chat-client_1.0.bb`

```bash
cat > ~/yocto/meta-chat/recipes-chat/chat-client/chat-client_1.0.bb << 'EOF'
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
EOF
```

---

## PART 11 — Write Image Recipes

### `chat-server-image.bb`

```bash
cat > ~/yocto/meta-chat/recipes-chat/images/chat-server-image.bb << 'EOF'
require recipes-core/images/core-image-minimal.bb

IMAGE_INSTALL:append = " \
    chat-server \
    qtbase \
    qtbase-plugins \
    openssh \
"

IMAGE_FEATURES:append = " ssh-server-openssh"
EOF
```

### `chat-client-image.bb`

```bash
cat > ~/yocto/meta-chat/recipes-chat/images/chat-client-image.bb << 'EOF'
require recipes-core/images/core-image-minimal.bb

IMAGE_INSTALL:append = " \
    chat-client \
    qtbase \
    qtbase-plugins \
    openssh \
"

IMAGE_FEATURES:append = " ssh-server-openssh"
EOF
```

---

## PART 12 — Verify the Layer Structure

```bash
find ~/yocto/meta-chat -type f | sort
```

Expected output:
```
/home/hp/yocto/meta-chat/conf/layer.conf
/home/hp/yocto/meta-chat/recipes-chat/chat-client/chat-client_1.0.bb
/home/hp/yocto/meta-chat/recipes-chat/chat-client/files/chat-client.pro
/home/hp/yocto/meta-chat/recipes-chat/chat-client/files/client.cpp
/home/hp/yocto/meta-chat/recipes-chat/chat-client/files/client.h
/home/hp/yocto/meta-chat/recipes-chat/chat-client/files/main-client.cpp
/home/hp/yocto/meta-chat/recipes-chat/chat-server/chat-server_1.0.bb
/home/hp/yocto/meta-chat/recipes-chat/chat-server/files/chat-server.pro
/home/hp/yocto/meta-chat/recipes-chat/chat-server/files/chat-server.service
/home/hp/yocto/meta-chat/recipes-chat/chat-server/files/main-server.cpp
/home/hp/yocto/meta-chat/recipes-chat/chat-server/files/server.cpp
/home/hp/yocto/meta-chat/recipes-chat/chat-server/files/server.h
/home/hp/yocto/meta-chat/recipes-chat/images/chat-client-image.bb
/home/hp/yocto/meta-chat/recipes-chat/images/chat-server-image.bb
```

---

## PART 13 — Verify Layers Are Recognized

```bash
cd ~/yocto/poky
source oe-init-build-env build
bitbake-layers show-layers
```

Confirm `meta-chat` appears in the list.

---

## PART 14 — Build the Images

> This step takes **1–3 hours** on first run. Subsequent builds are much faster due to caching.

```bash
cd ~/yocto/poky
source oe-init-build-env build

# Clean any stale cache for our recipes
bitbake -c cleansstate chat-server chat-client

# Build server image
bitbake chat-server-image

# Build client image
bitbake chat-client-image
```

Confirm both images were created:

```bash
ls ~/yocto/poky/build/tmp/deploy/images/qemux86-64/*.ext4
```

You should see two `.ext4` files — one for each image.

---

## PART 15 — Set Up the Network Bridge

> Run these commands in **WSL2** with a new terminal (not inside QEMU).
> You need root. Run once — the bridge persists until you reboot WSL2.

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

Verify the bridge:

```bash
ip link show br0
ip link show tap0
ip link show tap1
```

All three should show `state UP`.

---

## PART 16 — Start the Server QEMU (Terminal 1)

Open a dedicated terminal for the server VM and run:

```bash
cd ~/yocto/poky
source oe-init-build-env build

runqemu chat-server-image nographic \
  qemuparams="-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
              -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:02"
```

Wait for the login prompt, then log in:

```
qemux86-64 login: root
```

---

## PART 17 — Configure Network Inside Server VM

Inside the **server VM** terminal:

```bash
ip addr add 192.168.10.2/24 dev eth0
ip link set eth0 up
```

Verify the IP was assigned:

```bash
ip addr show eth0
# Should show: inet 192.168.10.2/24
```

---

## PART 18 — Verify Server Is Running

Still inside the **server VM**:

```bash
# Check systemd service status
systemctl status chat-server
```

Expected: `Active: active (running)`

```bash
# Check the port is listening (port 12345 = 0x3039 in hex)
grep 3039 /proc/net/tcp6
```

Expected: a line with state `0A` (LISTEN)

```bash
# Watch live logs
journalctl -fu chat-server -o cat | grep -v "QFont\|Qt no longer\|This plugin"
```

Expected:
```
[HH:MM:SS] Server started on port 12345
```

Leave this log command running to see client connections arrive.

---

## PART 19 — Start the Client QEMU (Terminal 2)

Open a **second terminal** for the client VM:

```bash
cd ~/yocto/poky
source oe-init-build-env build

runqemu chat-client-image nographic \
  qemuparams="-netdev tap,id=net0,ifname=tap1,script=no,downscript=no \
              -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:03"
```

Log in:

```
qemux86-64 login: root
```

---

## PART 20 — Configure Network Inside Client VM

Inside the **client VM** terminal:

```bash
ip addr add 192.168.10.3/24 dev eth0
ip link set eth0 up
```

Verify:

```bash
ip addr show eth0
# Should show: inet 192.168.10.3/24
```

---

## PART 21 — Test Connectivity Between the Two VMs

Inside the **client VM**, ping the server:

```bash
ping -c 4 192.168.10.2
```

Expected: 4 packets transmitted, 4 received, 0% packet loss.

> If ping fails, go back to Part 15 and re-run the bridge setup commands.

---

## PART 22 — Run the Chat Client

Inside the **client VM**:

```bash
QT_QPA_PLATFORM=offscreen chat-client 192.168.10.2
```

You will immediately see in the **client VM** terminal:

```
[HH:MM:SS] Connecting to 192.168.10.2:12345...
[HH:MM:SS] Connected to server.
[HH:MM:SS] [Me] Hello from chat-client on QEMU!
```

---

## PART 23 — Verify Communication on the Server

Switch to **Terminal 1** (server VM with `journalctl` running).

You should now see:

```
[HH:MM:SS] Client connected: ::ffff:192.168.10.3:XXXXX
[HH:MM:SS] [::ffff:192.168.10.3] Hello from chat-client on QEMU!
```

> This confirms end-to-end TCP communication between the two QEMU instances.

---

## PART 24 — (BONUS) Capture Traffic with Wireshark

Open a **third WSL2 terminal** (not inside QEMU):

```bash
sudo apt install tshark -y
sudo tshark -i br0 -f "tcp port 12345" -w /tmp/chat_capture.pcap
```

Let it run while the client sends messages. Press `Ctrl+C` to stop. Copy to Windows:

```bash
cp /tmp/chat_capture.pcap /mnt/c/Users/Hp/chat_capture.pcap
```

Open `C:\Users\Hp\chat_capture.pcap` in **Wireshark** on Windows to see the full TCP session.

---

## PART 25 — (BONUS) View the Qt GUI from Windows via VNC

### Step 1 — Start client with VNC (inside client VM)

```bash
QT_QPA_PLATFORM=vnc chat-client 192.168.10.2 &
```

Qt's built-in VNC server starts on port 5900 inside the client VM.

### Step 2 — Forward VNC port (new WSL2 terminal)

```bash
# Find your WSL2 IP
ip addr show eth0 | grep "inet "
```

Note the IP (e.g., `172.22.252.105`), then run:

```bash
socat TCP-LISTEN:5900,fork TCP:192.168.10.3:5900
```

### Step 3 — Connect from Windows

1. Open **RealVNC Viewer** on Windows
2. Connect to your WSL2 IP on port 5900 — e.g., `172.22.252.105:5900`
3. The Qt chat window appears
4. Type a message and press Enter — it shows up in the server journal

---

## PART 26 — Test Checklist

Use this checklist to confirm 100% completion:

```
[ ] Part 0  — Host dependencies installed
[ ] Part 1  — All 4 repos cloned (poky, meta-oe, meta-qt5, meta-raspberrypi)
[ ] Part 2  — oe-init-build-env sourced from ~/yocto/poky
[ ] Part 3  — local.conf has MACHINE = qemux86-64
[ ] Part 4  — bblayers.conf includes all 8 layers
[ ] Part 5  — meta-chat directory structure created
[ ] Part 6  — layer.conf written
[ ] Part 7  — server.h, server.cpp, main-server.cpp, chat-server.pro written
[ ] Part 8  — chat-server.service written
[ ] Part 9  — client.h, client.cpp, main-client.cpp, chat-client.pro written
[ ] Part 10 — chat-server_1.0.bb and chat-client_1.0.bb written
[ ] Part 11 — chat-server-image.bb and chat-client-image.bb written
[ ] Part 12 — find meta-chat shows 14 files
[ ] Part 13 — bitbake-layers show-layers lists meta-chat
[ ] Part 14 — both .ext4 images exist in deploy/images/qemux86-64/
[ ] Part 15 — br0, tap0, tap1 all show state UP
[ ] Part 16 — server QEMU booted, logged in as root
[ ] Part 17 — eth0 shows 192.168.10.2/24 in server VM
[ ] Part 18 — systemctl status chat-server shows active (running)
[ ] Part 18 — grep 3039 /proc/net/tcp6 shows LISTEN
[ ] Part 19 — client QEMU booted, logged in as root
[ ] Part 20 — eth0 shows 192.168.10.3/24 in client VM
[ ] Part 21 — ping 192.168.10.2 from client shows 0% packet loss
[ ] Part 22 — chat-client runs and prints "Connected to server"
[ ] Part 23 — server journal shows client IP and "Hello from chat-client"
[ ] Part 24 — (BONUS) .pcap file captured and opened in Wireshark
[ ] Part 25 — (BONUS) Qt GUI visible in RealVNC Viewer from Windows
```

---

## Quick Restart Guide

If you close all terminals and need to re-run the demo:

```bash
# Terminal 0 — WSL2: re-create bridge (lost on WSL2 restart)
sudo ip link add br0 type bridge
sudo ip addr add 192.168.10.1/24 dev br0
sudo ip link set br0 up
sudo ip tuntap add tap0 mode tap && sudo ip link set tap0 master br0 && sudo ip link set tap0 up
sudo ip tuntap add tap1 mode tap && sudo ip link set tap1 master br0 && sudo ip link set tap1 up

# Terminal 1 — Start server QEMU
cd ~/yocto/poky && source oe-init-build-env build
runqemu chat-server-image nographic qemuparams="-netdev tap,id=net0,ifname=tap0,script=no,downscript=no -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:02"
# Inside server VM:
ip addr add 192.168.10.2/24 dev eth0 && ip link set eth0 up
journalctl -fu chat-server -o cat | grep -v "QFont\|Qt no longer\|This plugin"

# Terminal 2 — Start client QEMU
cd ~/yocto/poky && source oe-init-build-env build
runqemu chat-client-image nographic qemuparams="-netdev tap,id=net0,ifname=tap1,script=no,downscript=no -device virtio-net-pci,netdev=net0,mac=52:54:00:12:34:03"
# Inside client VM:
ip addr add 192.168.10.3/24 dev eth0 && ip link set eth0 up
ping -c 2 192.168.10.2
QT_QPA_PLATFORM=offscreen chat-client 192.168.10.2
```
