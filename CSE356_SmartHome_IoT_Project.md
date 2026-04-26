# CSE356: Internet of Things — Project Report
**Ain Shams University | Faculty of Engineering | Computer and Systems Engineering Dept.**
**CHEP – Spring 2026**

---

# Smart Home Automation IoT System

**Project Title:** Smart Home Automation System
**Course:** CSE356 – Internet of Things
**Submitted to:** Dr. Islam Tharwat Abdel Halim

---

## Table of Contents

1. [Step 1: Purpose & Requirements](#step-1-purpose--requirements)
2. [Step 2: Process Specification](#step-2-process-specification)
3. [Step 3: Domain Model Specification](#step-3-domain-model-specification)
4. [Step 4: Information Model Specification](#step-4-information-model-specification)
5. [Step 5: Service Specifications](#step-5-service-specifications)
6. [Step 6: IoT Level Specification](#step-6-iot-level-specification)
7. [Step 7: Functional View Specification](#step-7-functional-view-specification)
8. [Step 8: Operational View Specification](#step-8-operational-view-specification)
9. [Step 9: Device & Component Integration](#step-9-device--component-integration)
10. [Step 10: Application Development](#step-10-application-development)

---

## Step 1: Purpose & Requirements

### 1.1 Purpose

The Smart Home Automation System (SHAS) is an IoT-based solution designed to enhance the comfort, safety, energy efficiency, and security of residential environments. The system enables homeowners to remotely monitor and control home appliances, lighting, climate, security cameras, and door locks through a unified mobile/web application. It also supports autonomous decision-making through rule-based automation and AI-driven recommendations.

### 1.2 Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-01 | The system shall allow users to remotely control lighting (on/off, dimming, color) via a mobile application. |
| FR-02 | The system shall monitor real-time temperature and humidity and automatically control the HVAC system. |
| FR-03 | The system shall detect motion and send push notifications to the homeowner. |
| FR-04 | The system shall control smart door locks with PIN, biometric, or remote unlock. |
| FR-05 | The system shall detect smoke/gas leakage and trigger alarms and automated ventilation. |
| FR-06 | The system shall monitor and report energy consumption per device. |
| FR-07 | The system shall support scheduled automation (e.g., turn off all lights at 11 PM). |
| FR-08 | The system shall stream live video from security cameras accessible via the app. |
| FR-09 | The system shall allow voice-command control via integration with Google Home / Amazon Alexa. |
| FR-10 | The system shall log all events and sensor readings in a cloud database. |

### 1.3 Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 | **Performance:** Sensor data shall be processed and acted upon within 500 ms under normal network conditions. |
| NFR-02 | **Reliability:** The system shall maintain 99.5% uptime with local fallback when cloud is unavailable. |
| NFR-03 | **Scalability:** The system shall support up to 200 devices per household without performance degradation. |
| NFR-04 | **Security:** All communications shall be encrypted using TLS 1.3; user authentication shall use OAuth 2.0 + MFA. |
| NFR-05 | **Usability:** The mobile app shall have an average task completion time under 3 seconds for common actions. |
| NFR-06 | **Interoperability:** The system shall support Zigbee, Z-Wave, Wi-Fi, and Bluetooth device protocols. |
| NFR-07 | **Privacy:** No video/audio data shall be stored in the cloud without explicit user consent. |
| NFR-08 | **Maintainability:** Over-the-air (OTA) firmware updates shall be supported for all IoT devices. |

---

## Step 2: Process Specification

### 2.1 Overview

The SHAS performs three categories of processes: **Sensing & Data Collection**, **Processing & Decision Making**, and **Actuation & Response**.

### 2.2 Main System Process Flowchart

```mermaid
flowchart TD
    A([System Start]) --> B[Initialize Sensors & Devices]
    B --> C[Connect to Home Gateway]
    C --> D{Connection\nSuccessful?}
    D -- No --> E[Switch to Local Mode]
    D -- Yes --> F[Connect to Cloud Platform]
    E --> G
    F --> G[Begin Continuous Sensor Polling]

    G --> H[Read Sensor Data\nTemp / Humidity / Motion / Smoke / Light]
    H --> I[Send Data to Gateway]
    I --> J{Rule Engine\nEvaluation}

    J -- Rule Triggered --> K[Execute Automation Action]
    J -- No Rule Triggered --> L[Store Data in Time-Series DB]
    K --> L

    L --> M{User\nCommand\nReceived?}
    M -- Yes --> N[Parse & Validate Command]
    N --> O[Send Actuation Signal to Device]
    O --> P[Confirm Action & Update UI]
    M -- No --> H

    P --> Q{Alert\nCondition?}
    Q -- Yes --> R[Send Push Notification to User]
    Q -- No --> H
    R --> H
```

### 2.3 Automation Rule Execution Process

```mermaid
flowchart LR
    A[Sensor Reading Received] --> B{Evaluate\nRule Conditions}
    B --> C{Temp > 28°C?}
    B --> D{Motion Detected\nat Night?}
    B --> E{Smoke > Threshold?}
    B --> F{Lux < 200?}

    C -- Yes --> C1[Turn ON AC\nSet to 22°C]
    D -- Yes --> D1[Turn ON\nSecurity Lights\nSend Alert]
    E -- Yes --> E1[Trigger Fire Alarm\nOpen Vents\nCall Emergency]
    F -- Yes --> F1[Turn ON\nAmbient Lights\nto 60%]

    C1 --> G[Log Action]
    D1 --> G
    E1 --> G
    F1 --> G
```

### 2.4 User Command Process

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant Cloud as Cloud Platform
    participant GW as Home Gateway
    participant Device as Smart Device

    User->>App: Issue Command (e.g., Turn on Living Room Light)
    App->>Cloud: POST /api/command {deviceId, action}
    Cloud->>Cloud: Authenticate & Authorize User
    Cloud->>GW: Forward Command via MQTT
    GW->>Device: Send Actuation Signal (Zigbee/Z-Wave)
    Device->>GW: Acknowledge Execution
    GW->>Cloud: Publish Status Update
    Cloud->>App: Push Notification / WebSocket Update
    App->>User: Display Updated State
```

---

## Step 3: Domain Model Specification

### 3.1 Description

The domain model identifies the core entities of the Smart Home Automation System and their relationships. The main entities are: **User**, **Home**, **Room**, **Device**, **Sensor**, **Actuator**, **Gateway**, **Cloud Platform**, **Automation Rule**, and **Alert**.

### 3.2 UML Class Diagram

```mermaid
classDiagram
    class User {
        +String userId
        +String name
        +String email
        +String phoneNumber
        +String role
        +login()
        +logout()
        +manageDevices()
        +setRule()
    }

    class Home {
        +String homeId
        +String address
        +String timezone
        +addRoom()
        +removeRoom()
        +getEnergyReport()
    }

    class Room {
        +String roomId
        +String name
        +String type
        +int floor
        +addDevice()
        +removeDevice()
    }

    class Device {
        +String deviceId
        +String name
        +String type
        +String protocol
        +String status
        +String firmwareVersion
        +turnOn()
        +turnOff()
        +configure()
        +updateFirmware()
    }

    class Sensor {
        +String sensorId
        +String type
        +String unit
        +float currentValue
        +DateTime lastReading
        +readValue()
        +calibrate()
    }

    class Actuator {
        +String actuatorId
        +String type
        +String currentState
        +execute(command)
        +getState()
    }

    class Gateway {
        +String gatewayId
        +String ipAddress
        +String[] supportedProtocols
        +bool cloudConnected
        +routeMessage()
        +manageLocalCache()
        +performOTA()
    }

    class CloudPlatform {
        +String platformId
        +storeData()
        +processRules()
        +sendNotification()
        +authenticateUser()
    }

    class AutomationRule {
        +String ruleId
        +String name
        +String triggerCondition
        +String action
        +bool isActive
        +evaluate()
        +execute()
    }

    class Alert {
        +String alertId
        +String type
        +String severity
        +String message
        +DateTime timestamp
        +bool acknowledged
        +send()
        +acknowledge()
    }

    User "1" --> "1..*" Home : owns
    Home "1" *-- "1..*" Room : contains
    Room "1" *-- "0..*" Device : hosts
    Device <|-- Sensor : extends
    Device <|-- Actuator : extends
    Home "1" --> "1" Gateway : connected to
    Gateway "1" --> "1" CloudPlatform : communicates with
    User "1" --> "0..*" AutomationRule : defines
    AutomationRule "1" --> "0..*" Alert : generates
    Sensor "1" --> "0..*" AutomationRule : triggers
```

### 3.3 Entity Relationship Overview

```mermaid
erDiagram
    USER ||--o{ HOME : owns
    HOME ||--|{ ROOM : contains
    ROOM ||--o{ DEVICE : hosts
    DEVICE ||--o{ SENSOR_READING : generates
    DEVICE ||--o{ ACTUATION_LOG : records
    USER ||--o{ AUTOMATION_RULE : creates
    AUTOMATION_RULE ||--o{ ALERT : triggers
    HOME ||--|| GATEWAY : "connected to"
    GATEWAY ||--o{ DEVICE : manages
```

---

## Step 4: Information Model Specification

### 4.1 Data Types and Structures

#### Sensor Reading Object
```json
{
  "readingId": "uuid-v4",
  "deviceId": "sensor-001",
  "homeId": "home-123",
  "roomId": "room-456",
  "type": "temperature",
  "value": 24.5,
  "unit": "°C",
  "timestamp": "2026-04-26T10:30:00Z",
  "quality": "good"
}
```

#### Device State Object
```json
{
  "deviceId": "light-001",
  "name": "Living Room Main Light",
  "type": "smart_bulb",
  "status": "online",
  "state": {
    "power": "on",
    "brightness": 75,
    "color": "#FFFFFF",
    "colorTemp": 4000
  },
  "lastUpdated": "2026-04-26T10:28:00Z",
  "energyConsumption_W": 8.5
}
```

#### Automation Rule Object
```json
{
  "ruleId": "rule-007",
  "name": "Night Motion Security",
  "trigger": {
    "sensor": "motion-sensor-front-door",
    "condition": "motion_detected == true",
    "timeWindow": "22:00-06:00"
  },
  "action": {
    "devices": ["security-light-01", "camera-01"],
    "command": "turn_on",
    "notification": true,
    "notificationMessage": "Motion detected at front door!"
  },
  "isActive": true,
  "createdBy": "user-001"
}
```

### 4.2 Information Flow Diagram

```mermaid
flowchart LR
    subgraph Field["🏠 Field Layer"]
        S1[Temperature\nSensor]
        S2[Motion\nSensor]
        S3[Smoke\nSensor]
        S4[Light\nSensor]
        A1[Smart\nBulb]
        A2[Smart\nLock]
        A3[AC Unit]
    end

    subgraph Edge["🔀 Edge / Gateway Layer"]
        GW[Home Gateway\nRaspberry Pi]
        LocalDB[(Local\nCache DB)]
        RuleEng[Local Rule\nEngine]
    end

    subgraph Cloud["☁️ Cloud Layer"]
        MQTT[MQTT Broker]
        API[REST API\nServer]
        CloudDB[(Time-Series\nDatabase)]
        Analytics[Analytics\nEngine]
        Notif[Notification\nService]
    end

    subgraph App["📱 Application Layer"]
        Mobile[Mobile App]
        Web[Web Dashboard]
        Voice[Voice Assistant]
    end

    S1 & S2 & S3 & S4 -->|Raw Data| GW
    GW <-->|Commands| A1 & A2 & A3
    GW <--> LocalDB
    GW --> RuleEng
    RuleEng -->|Local Actions| A1 & A2 & A3
    GW <-->|MQTT Publish/Subscribe| MQTT
    MQTT <--> API
    API <--> CloudDB
    CloudDB --> Analytics
    Analytics --> Notif
    API <-->|REST/WebSocket| Mobile
    API <-->|REST/WebSocket| Web
    API <-->|REST/WebSocket| Voice
    Notif --> Mobile
```

### 4.3 Data Retention Policy

| Data Type | Retention Period | Storage Location |
|-----------|-----------------|-----------------|
| Raw sensor readings | 7 days | Local Gateway Cache |
| Aggregated sensor data (1-min avg) | 1 year | Cloud Time-Series DB |
| Event logs & alerts | 3 years | Cloud Relational DB |
| Video recordings | 30 days | Encrypted Cloud Storage |
| User preferences & rules | Indefinite | Cloud Relational DB |

---

## Step 5: Service Specifications

### 5.1 Service Catalog

The SHAS exposes the following services to client applications:

### 5.2 Service Architecture Diagram

```mermaid
graph TB
    subgraph Client["Client Applications"]
        MA[Mobile App]
        WD[Web Dashboard]
        VA[Voice Assistant]
    end

    subgraph API_GW["API Gateway"]
        Auth[Auth Service\nOAuth 2.0 + MFA]
        RateLimit[Rate Limiter]
        Router[Service Router]
    end

    subgraph Microservices["Microservices"]
        DS[Device\nService]
        SS[Sensor\nService]
        AS[Automation\nService]
        NS[Notification\nService]
        ES[Energy\nService]
        VS[Video\nService]
        US[User\nService]
    end

    subgraph Data["Data Layer"]
        RDB[(Relational DB\nPostgreSQL)]
        TSDB[(Time-Series DB\nInfluxDB)]
        Cache[(Cache\nRedis)]
        Queue[Message Queue\nRabbitMQ]
        ObjStore[(Object Storage\nVideo/Images)]
    end

    MA & WD & VA --> Auth
    Auth --> RateLimit --> Router
    Router --> DS & SS & AS & NS & ES & VS & US
    DS <--> RDB & Cache
    SS <--> TSDB & Queue
    AS <--> RDB & Queue
    NS <--> Queue
    ES <--> TSDB & RDB
    VS <--> ObjStore
    US <--> RDB & Cache
```

### 5.3 REST API Endpoints

| Service | Method | Endpoint | Description |
|---------|--------|----------|-------------|
| Device Service | GET | `/api/v1/devices` | List all devices |
| Device Service | POST | `/api/v1/devices/{id}/command` | Send command to device |
| Device Service | GET | `/api/v1/devices/{id}/status` | Get device status |
| Sensor Service | GET | `/api/v1/sensors/{id}/readings` | Get sensor readings |
| Automation Service | GET | `/api/v1/rules` | List automation rules |
| Automation Service | POST | `/api/v1/rules` | Create new rule |
| Automation Service | PUT | `/api/v1/rules/{id}` | Update rule |
| Automation Service | DELETE | `/api/v1/rules/{id}` | Delete rule |
| Notification Service | GET | `/api/v1/alerts` | Get recent alerts |
| Energy Service | GET | `/api/v1/energy/report` | Get energy report |
| User Service | POST | `/api/v1/auth/login` | User login |
| Video Service | GET | `/api/v1/cameras/{id}/stream` | Get live stream URL |

### 5.4 MQTT Topic Structure

```
shas/{homeId}/{roomId}/{deviceId}/telemetry     ← Sensor data upload
shas/{homeId}/{roomId}/{deviceId}/command       ← Commands to device
shas/{homeId}/{roomId}/{deviceId}/status        ← Device state changes
shas/{homeId}/alerts                            ← Home-level alerts
shas/{homeId}/energy                            ← Energy consumption data
```

---

## Step 6: IoT Level Specification

### 6.1 IoT Architecture Levels

The system follows a **6-level IoT architecture**:

```mermaid
graph TB
    subgraph L1["Level 1 — Physical / Sensing Layer"]
        direction LR
        TH[Temperature &\nHumidity Sensor\nDHT22]
        MS[Motion Sensor\nPIR HC-SR501]
        SM[Smoke Sensor\nMQ-2]
        LS[Light Sensor\nBH1750]
        DS[Door / Window\nContact Sensor]
        CAM[IP Camera]
    end

    subgraph L2["Level 2 — Network / Communication Layer"]
        direction LR
        ZB[Zigbee\nMesh Network]
        ZW[Z-Wave\nMesh Network]
        WIFI[Wi-Fi 6\n802.11ax]
        BLE[Bluetooth LE 5.0]
    end

    subgraph L3["Level 3 — Gateway / Edge Layer"]
        GW[Home Gateway\nRaspberry Pi 4\nHome Assistant OS]
        EDGE[Edge Processing\nLocal Rule Engine]
    end

    subgraph L4["Level 4 — Processing / Middleware Layer"]
        MQTT2[MQTT Broker\nMosquitto]
        API2[REST API Server\nNode.js]
        MSGQ[Message Queue\nRabbitMQ]
    end

    subgraph L5["Level 5 — Data / Storage Layer"]
        TSDB2[(InfluxDB\nTime-Series)]
        RDB2[(PostgreSQL\nRelational)]
        CACHE[(Redis Cache)]
    end

    subgraph L6["Level 6 — Application Layer"]
        APP[Mobile App\nReact Native]
        WEB[Web Dashboard\nReact.js]
        VOICE[Voice Control\nAlexa / Google]
    end

    L1 --> L2
    L2 --> L3
    L3 --> L4
    L4 --> L5
    L5 --> L6
    L6 -.->|User Commands| L4
    L4 -.->|Actuation| L3
    L3 -.->|Control| L1
```

### 6.2 Protocol Selection per Level

| Level | Protocol/Technology | Justification |
|-------|-------------------|---------------|
| Sensing | Zigbee 3.0 / Z-Wave | Low power, mesh network, high reliability |
| Local Network | Wi-Fi 6 / BLE | High throughput for cameras; BLE for wearables |
| Gateway | MQTT (QoS 1 & 2) | Lightweight publish/subscribe, ideal for IoT |
| Cloud | HTTPS REST + WebSocket | Secure, standard, real-time push capability |
| Data Storage | InfluxDB + PostgreSQL | Time-series for sensor data, relational for config |
| Application | React Native / React.js | Cross-platform, responsive UI |

---

## Step 7: Functional View Specification

### 7.1 Functional Blocks Diagram

```mermaid
graph TB
    subgraph FB1["FB1: Data Acquisition"]
        DA1[Sensor Polling\nModule]
        DA2[Protocol\nAdapter\nZigbee/Z-Wave/WiFi]
        DA3[Data Validation\n& Filtering]
    end

    subgraph FB2["FB2: Communication Management"]
        CM1[MQTT Client\nPublisher]
        CM2[Message Router]
        CM3[Offline Queue\nManager]
    end

    subgraph FB3["FB3: Rule & Automation Engine"]
        RE1[Rule Parser]
        RE2[Condition Evaluator]
        RE3[Action Dispatcher]
        RE4[Schedule Manager\nCron-based]
    end

    subgraph FB4["FB4: Device Control"]
        DC1[Command Validator]
        DC2[Actuator Controller]
        DC3[State Manager]
        DC4[OTA Update\nManager]
    end

    subgraph FB5["FB5: Data Management"]
        DM1[Time-Series\nData Writer]
        DM2[Data Aggregator]
        DM3[Query Engine]
        DM4[Data Retention\nManager]
    end

    subgraph FB6["FB6: Security & Identity"]
        SEC1[Authentication\nOAuth 2.0 + MFA]
        SEC2[Authorization\nRBAC]
        SEC3[Encryption\nTLS 1.3]
        SEC4[Audit Logger]
    end

    subgraph FB7["FB7: User Interface"]
        UI1[Mobile App\nReact Native]
        UI2[Web Dashboard\nReact.js]
        UI3[Voice Interface\nAlexa/Google]
        UI4[Notification\nPush/SMS/Email]
    end

    subgraph FB8["FB8: Analytics & Reporting"]
        AN1[Energy\nAnalytics]
        AN2[Usage Pattern\nAnalysis]
        AN3[Anomaly\nDetection]
        AN4[Report Generator]
    end

    FB1 --> FB2
    FB2 --> FB3
    FB2 --> FB5
    FB3 --> FB4
    FB4 --> FB1
    FB5 --> FB8
    FB6 --> FB1 & FB2 & FB3 & FB4 & FB5
    FB7 --> FB3 & FB4
    FB8 --> FB7
```

### 7.2 Functional Block Descriptions

| Functional Block | Role | Key Components |
|-----------------|------|---------------|
| **FB1: Data Acquisition** | Collects raw data from all physical sensors using their native protocols | Sensor polling, protocol adapters, data validation |
| **FB2: Communication Management** | Manages all message routing between layers, handles offline queuing | MQTT client, message router, offline queue |
| **FB3: Rule & Automation Engine** | Evaluates trigger conditions and dispatches automated actions | Rule parser, condition evaluator, scheduler |
| **FB4: Device Control** | Validates and sends control commands to actuators; manages device state | Command validator, actuator controller, OTA |
| **FB5: Data Management** | Persists, aggregates and queries all sensor and event data | InfluxDB writer, aggregator, query engine |
| **FB6: Security & Identity** | Ensures all access is authenticated, authorized and encrypted | OAuth 2.0, RBAC, TLS 1.3, audit logs |
| **FB7: User Interface** | Provides all user-facing interfaces for control and monitoring | Mobile app, web dashboard, voice, notifications |
| **FB8: Analytics & Reporting** | Derives insights from historical data; detects anomalies | Energy reports, usage patterns, anomaly detection |

---

## Step 8: Operational View Specification

### 8.1 Performance Considerations

| Metric | Target | Mechanism |
|--------|--------|-----------|
| Sensor data latency | < 500 ms end-to-end | Edge processing, local rule engine |
| Command response time | < 1 second | Direct MQTT command channel |
| App UI response | < 3 seconds | Redis caching, CDN for static assets |
| Video stream latency | < 2 seconds | WebRTC / RTSP with adaptive bitrate |
| System uptime | 99.5% | Redundant cloud deployment, local fallback |

### 8.2 Reliability & Fault Tolerance

```mermaid
flowchart TD
    A[Device Sends Data] --> B{Gateway\nReachable?}
    B -- Yes --> C[Forward to Cloud via MQTT]
    B -- No --> D[Store in Local Cache]
    D --> E{Gateway\nRestored?}
    E -- Yes --> F[Flush Cache to Cloud]
    E -- No --> G[Continue Local Operations\nLocal Rules Still Active]

    C --> H{Cloud\nReachable?}
    H -- Yes --> I[Process & Store in Cloud]
    H -- No --> J[Gateway Queues Messages\nQoS 1 - At Least Once Delivery]
    J --> K{Cloud\nRestored?}
    K -- Yes --> L[Deliver Queued Messages]
```

### 8.3 Security Architecture

```mermaid
graph LR
    subgraph Internet["🌐 Internet"]
        UA[User App]
    end

    subgraph DMZ["🛡️ DMZ / Edge Security"]
        WAF[Web Application\nFirewall]
        APIGW[API Gateway\nRate Limiting]
        LB[Load Balancer\nSSL Termination]
    end

    subgraph Backend["🔒 Secured Backend"]
        AUTH[Auth Service\nOAuth 2.0 + JWT]
        MS2[Microservices\nInternal mTLS]
        DB2[(Encrypted\nDatabases)]
    end

    subgraph Home["🏠 Home Network"]
        GW2[Gateway\nVPN Tunnel]
        Devices[IoT Devices\nZigbee / Z-Wave]
    end

    UA -->|HTTPS TLS 1.3| WAF
    WAF --> APIGW --> LB
    LB --> AUTH
    AUTH -->|JWT Bearer Token| MS2
    MS2 <--> DB2
    GW2 <-->|VPN + MQTT TLS| LB
    GW2 <-->|Encrypted Local| Devices
```

### 8.4 Security Measures Summary

| Threat | Mitigation |
|--------|-----------|
| Unauthorized access | OAuth 2.0, MFA, JWT short-lived tokens |
| Man-in-the-middle | TLS 1.3 on all communications, VPN for gateway |
| Device tampering | Signed firmware, secure boot, OTA validation |
| Data breach | AES-256 encryption at rest, RBAC access control |
| DDoS attacks | Rate limiting at API gateway, WAF rules |
| Replay attacks | MQTT message timestamps + nonce validation |

---

## Step 9: Device & Component Integration

### 9.1 Device Inventory

| # | Device | Model | Protocol | Function |
|---|--------|-------|----------|----------|
| 1 | Temperature & Humidity Sensor | DHT22 / Sonoff SNZB-02 | Zigbee | Monitor room climate |
| 2 | Motion Sensor (PIR) | HC-SR501 / Aqara MS-S02 | Zigbee | Detect movement |
| 3 | Smoke / Gas Sensor | MQ-2 / Heiman HS3SA | Zigbee | Fire/gas detection |
| 4 | Light Level Sensor | BH1750 / Xiaomi GZCGQ01LM | Zigbee | Ambient light monitoring |
| 5 | Door/Window Contact Sensor | Aqara MCCGQ11LM | Zigbee | Open/close detection |
| 6 | Smart Bulb | Philips Hue / IKEA TRÅDFRI | Zigbee | Lighting control |
| 7 | Smart Plug | TP-Link Tapo P115 | Wi-Fi | Appliance power control |
| 8 | Smart Door Lock | Schlage BE489WB | Z-Wave | Entry access control |
| 9 | Smart Thermostat | Google Nest / Honeywell T6R | Wi-Fi | HVAC control |
| 10 | IP Security Camera | Reolink RLC-810A | Wi-Fi (RTSP) | Video surveillance |
| 11 | Home Gateway | Raspberry Pi 4 (4GB RAM) | All protocols | Central hub |
| 12 | Zigbee Coordinator | ConBee II USB Stick | Zigbee | Zigbee mesh coordinator |
| 13 | Z-Wave Controller | Aeotec Z-Stick Gen5 | Z-Wave | Z-Wave mesh controller |

### 9.2 Network Topology Diagram

```mermaid
graph TB
    subgraph Internet["☁️ Internet / Cloud"]
        CLOUD[Cloud Platform\nAWS / Azure]
    end

    subgraph Router["🌐 Home Router"]
        RTR[Wi-Fi 6 Router\n192.168.1.1]
    end

    subgraph Gateway["🖥️ Home Gateway - 192.168.1.10"]
        RPI[Raspberry Pi 4\nHome Assistant]
        ZB_COORD[ConBee II\nZigbee Coordinator]
        ZW_CTRL[Aeotec Z-Stick\nZ-Wave Controller]
    end

    subgraph WiFi_Devices["📶 Wi-Fi Devices"]
        CAM1[IP Camera\nLiving Room\n192.168.1.21]
        CAM2[IP Camera\nFront Door\n192.168.1.22]
        NEST[Smart Thermostat\n192.168.1.23]
        PLUG1[Smart Plug\n192.168.1.24]
    end

    subgraph Zigbee_Mesh["🔷 Zigbee Mesh Network"]
        ZB1[Temp Sensor\nLiving Room]
        ZB2[Motion Sensor\nHallway]
        ZB3[Smoke Sensor\nKitchen]
        ZB4[Smart Bulb\nBedroom]
        ZB5[Contact Sensor\nFront Window]
    end

    subgraph ZWave_Mesh["🔶 Z-Wave Mesh Network"]
        ZW1[Smart Door Lock\nFront Door]
        ZW2[Smart Door Lock\nBack Door]
    end

    CLOUD <-->|HTTPS / MQTT TLS| RTR
    RTR <-->|Ethernet| RPI
    RPI --- ZB_COORD
    RPI --- ZW_CTRL
    RTR <-->|Wi-Fi| CAM1 & CAM2 & NEST & PLUG1
    ZB_COORD <-->|Zigbee 3.0 2.4GHz| ZB1 & ZB2 & ZB3 & ZB4 & ZB5
    ZW_CTRL <-->|Z-Wave 908MHz| ZW1 & ZW2
```

### 9.3 Communication Protocol Stack

```mermaid
graph TB
    subgraph App_Layer["Application Layer"]
        MQTT3[MQTT 3.1.1\nSensor Telemetry]
        HTTP2[HTTP/2 REST\nAPI Commands]
        WS[WebSocket\nReal-time UI]
    end

    subgraph Trans_Layer["Transport Layer"]
        TLS2[TLS 1.3\nEncryption]
        TCP[TCP]
        UDP[UDP]
    end

    subgraph Net_Layer["Network Layer"]
        IPv4[IPv4 / IPv6]
        THREAD[Thread\nIPv6 Mesh]
    end

    subgraph MAC_Layer["MAC / Radio Layer"]
        WIFI2[Wi-Fi 6\n802.11ax]
        ZBMAC[IEEE 802.15.4\nZigbee PHY/MAC]
        ZWMAC[Z-Wave\n908 MHz]
        BLE2[BLE 5.0\nBluetooth]
    end

    App_Layer --> Trans_Layer --> Net_Layer --> MAC_Layer
```

---

## Step 10: Application Development

### 10.1 Application Architecture

The SHAS provides three client applications:
1. **Mobile App** (React Native – iOS & Android)
2. **Web Dashboard** (React.js – Browser)
3. **Voice Interface** (Alexa Skill / Google Action)

### 10.2 Application Component Diagram

```mermaid
graph TB
    subgraph MobileApp["📱 Mobile App — React Native"]
        MA_AUTH[Auth Screen\nLogin / MFA]
        MA_DASH[Dashboard\nHome Overview]
        MA_ROOMS[Rooms View\nDevice Grid]
        MA_DEVICE[Device Control\nPanel]
        MA_RULES[Automation\nRules Manager]
        MA_ENERGY[Energy\nReports]
        MA_CAM[Camera\nViewer LiveStream]
        MA_NOTIF[Notifications\nCenter]
        MA_SETTINGS[Settings &\nUser Profile]
    end

    subgraph StateManagement["🗂️ State Management — Redux Toolkit"]
        STORE[Global Store]
        API_SLICE[API Slice\nRTK Query]
        WS_SLICE[WebSocket\nSlice]
    end

    subgraph Backend["☁️ Backend API"]
        REST_API[REST API]
        WS_SRV[WebSocket Server]
        PUSH[Push Notification\nFirebase FCM]
    end

    MA_AUTH & MA_DASH & MA_ROOMS & MA_DEVICE & MA_RULES & MA_ENERGY & MA_CAM & MA_NOTIF & MA_SETTINGS --> STORE
    STORE <--> API_SLICE
    STORE <--> WS_SLICE
    API_SLICE <-->|HTTPS| REST_API
    WS_SLICE <-->|WSS| WS_SRV
    PUSH -->|FCM| MA_NOTIF
```

### 10.3 Mobile App Screen Flow

```mermaid
stateDiagram-v2
    [*] --> SplashScreen
    SplashScreen --> LoginScreen : Not Authenticated
    SplashScreen --> Dashboard : Token Valid

    LoginScreen --> MFAScreen : Credentials OK
    MFAScreen --> Dashboard : MFA Verified
    LoginScreen --> LoginScreen : Failed

    Dashboard --> RoomsView : View Rooms
    Dashboard --> AlertsView : View Alerts
    Dashboard --> EnergyView : View Energy
    Dashboard --> SettingsView : Open Settings

    RoomsView --> DeviceControl : Select Device
    DeviceControl --> RoomsView : Back
    DeviceControl --> AutomationRules : Set Rule

    AutomationRules --> RuleEditor : Create/Edit Rule
    RuleEditor --> AutomationRules : Save Rule

    SettingsView --> UserProfile : Edit Profile
    SettingsView --> DeviceManager : Manage Devices
    SettingsView --> Integrations : Voice/3rd Party

    AlertsView --> Dashboard : Back
    EnergyView --> Dashboard : Back
```

### 10.4 UI Dashboard Layout

```
┌────────────────────────────────────────────────┐
│  🏠 My Smart Home              🔔 3  👤 Profile │
├────────────────────────────────────────────────┤
│  ☀️ Good Morning, Ahmed!    23°C  💧 45% RH    │
│  All systems normal                             │
├─────────────┬──────────────┬───────────────────┤
│  💡 Lights  │  ❄️ Climate  │  🔒 Security      │
│  12 ON      │  22°C Set    │  All Locked        │
│  4 OFF      │  Auto Mode   │  0 Alerts          │
├─────────────┴──────────────┴───────────────────┤
│  ROOMS                                          │
│  [Living Room] [Kitchen] [Bedroom] [Garage] ▶  │
├────────────────────────────────────────────────┤
│  ENERGY TODAY              ⚡ 3.2 kWh / $0.48  │
│  ████████░░░░░░░░░░░░░░░  67% of daily avg     │
├────────────────────────────────────────────────┤
│  RECENT ALERTS                                  │
│  🚪 Front door opened — 07:32 AM               │
│  🌡️ Bedroom temp reached 28°C — 02:15 AM       │
└────────────────────────────────────────────────┘
```

### 10.5 Voice Control Integration Flow

```mermaid
sequenceDiagram
    actor User
    participant VA as Alexa / Google
    participant Skill as SHAS Skill/Action
    participant API3 as SHAS API
    participant GW3 as Home Gateway
    participant Dev as Device

    User->>VA: "Alexa, turn off the living room lights"
    VA->>Skill: Intent: TurnOff, Entity: living_room_lights
    Skill->>API3: POST /api/v1/devices/light-lr-01/command\n{action: "off"}
    API3->>GW3: MQTT: shas/home-1/living-room/light-lr-01/command
    GW3->>Dev: Zigbee Off Command
    Dev->>GW3: ACK
    GW3->>API3: Status Update: power=off
    API3->>Skill: 200 OK
    Skill->>VA: "OK, turning off the living room lights"
    VA->>User: "OK, turning off the living room lights"
```

### 10.6 Technology Stack Summary

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Mobile App | React Native 0.73 | Cross-platform iOS & Android from single codebase |
| Web Frontend | React.js 18 + Tailwind CSS | Fast, component-based, responsive UI |
| State Management | Redux Toolkit + RTK Query | Predictable state, efficient API caching |
| Backend API | Node.js + Express | Event-driven, ideal for real-time IoT workloads |
| MQTT Broker | Eclipse Mosquitto | Open-source, lightweight, production proven |
| Time-Series DB | InfluxDB 2.x | Optimized for high-frequency sensor data |
| Relational DB | PostgreSQL 16 | ACID-compliant, robust for user and config data |
| Cache | Redis 7 | Sub-millisecond latency for device state |
| Message Queue | RabbitMQ | Reliable async processing of sensor events |
| Cloud Platform | AWS IoT Core + EC2 | Managed MQTT, auto-scaling, global availability |
| Container Orchestration | Docker + Kubernetes | Portable, scalable microservice deployment |
| CI/CD | GitHub Actions | Automated testing and deployment pipeline |
| Gateway OS | Home Assistant OS (Raspberry Pi 4) | Rich device ecosystem, local processing |

---

## Summary & Conclusion

The **Smart Home Automation System (SHAS)** has been comprehensively designed following the 10-step IoT design methodology. The system:

- **Addresses a real problem:** energy waste, security vulnerabilities, and lack of convenience in traditional homes.
- **Employs a layered architecture:** from physical sensors through edge gateway to cloud services and user applications.
- **Prioritizes security:** with TLS 1.3, OAuth 2.0, MFA, RBAC, and VPN-secured gateway communication.
- **Ensures reliability:** through local fallback mode, MQTT QoS guarantees, and cloud redundancy.
- **Supports scalability:** via containerized microservices and a multi-protocol device ecosystem (Zigbee, Z-Wave, Wi-Fi).
- **Delivers rich user experience:** through a React Native mobile app, web dashboard, and voice assistant integration.

The prototype will be implemented in Cisco Packet Tracer simulating the core components: the home gateway, IoT sensors, actuators, Wi-Fi/LAN network, and cloud connectivity.

---

## References

1. Bahga, A., & Madisetti, V. (2014). *Internet of Things: A Hands-on Approach.* VPT.
2. Rose, K., Eldridge, S., & Chapin, L. (2015). *The Internet of Things: An Overview.* Internet Society.
3. IEEE Xplore: Smart Home Automation Survey – doi:10.1109/ACCESS.2019.2930467
4. Zigbee Alliance. (2023). *Zigbee 3.0 Specification.* CSA.
5. OWASP IoT Security Guidance. (2024). Retrieved from https://owasp.org/www-project-iot-security-verification-standard/
6. InfluxData. (2024). *InfluxDB 2.x Documentation.* Retrieved from https://docs.influxdata.com
7. Home Assistant Documentation. (2024). Retrieved from https://www.home-assistant.io/docs/

---

*Report prepared for CSE356: Internet of Things | CHEP – Spring 2026 | Ain Shams University*
