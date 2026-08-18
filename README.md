# UPI-WITHOUT-INTERNET

### Offline Payment Simulation Using Mesh Networking

No-Net-UPI is a **Spring Boot prototype that explores how a payment could be transferred through a network of nearby devices when the sender has no direct internet connection**.

Instead of relying on real Bluetooth hardware, the project models phones as virtual devices and simulates packet propagation between them. A device that eventually has internet connectivity acts as a bridge and forwards the payment packet to the backend for verification and settlement.

The project focuses on three important backend concerns:

* Protecting payment information while it passes through untrusted devices
* Preventing the same payment from being processed multiple times
* Rejecting stale or modified payment packets

> **Note:** This is an educational prototype and does not implement real UPI/NPCI infrastructure or real Bluetooth communication.

---

## ✨ Key Features

### 🔐 Secure Payment Payloads

Payment data is protected using a hybrid encryption approach:

* RSA-2048 with OAEP for securely transferring the session key
* AES-256-GCM for encrypting the payment payload
* GCM authentication tags to detect modified ciphertext
* SHA-256 hashing for identifying duplicate packets

Intermediate devices therefore carry encrypted packets rather than readable payment information.

### 🔄 Mesh Packet Propagation

The project contains a software-based mesh simulator where virtual phones exchange payment packets.

Each packet contains metadata such as:

* Packet ID
* TTL
* Creation timestamp
* Encrypted payment data

The simulator performs multiple gossip rounds to model packet movement through nearby devices.

### ♻️ Idempotent Settlement

A major part of the project is handling duplicate deliveries.

The same packet may reach the backend through multiple bridge devices. Before performing decryption and settlement, the backend calculates a hash of the ciphertext and attempts to claim it atomically.

This ensures that only one request can process a particular payment packet while duplicate requests are discarded.

The implementation currently uses `ConcurrentHashMap` for this behavior.

### ⏱️ Replay Protection

Payment packets contain a timestamp and nonce.

The backend checks the packet's age before allowing settlement. Old packets are rejected, while the nonce ensures separate legitimate payments generate different encrypted payloads.

### 💰 Transactional Settlement

Once a packet passes validation:

1. The sender's account is debited.
2. The receiver's account is credited.
3. The transaction is recorded in the ledger.

These operations are performed inside a Spring transaction.

Optimistic locking is also used on account records to provide additional protection during concurrent updates.

---

## 🏗️ Architecture

```text
                 OFFLINE DEVICE
                       │
                       │ Create payment
                       ▼
                Encrypt payload
                       │
                       ▼
                 Mesh Packet
                       │
          ┌────────────┴────────────┐
          │                         │
       Device A                  Device B
          │                         │
          └──────────┬──────────────┘
                     │
                 Gossip rounds
                     │
                     ▼
                Bridge Device
               (has internet)
                     │
                     │ HTTP POST
                     ▼
          ┌─────────────────────────┐
          │      Spring Boot API    │
          │                         │
          │  1. Hash packet         │
          │  2. Check idempotency   │
          │  3. Decrypt payload     │
          │  4. Validate freshness  │
          │  5. Settle transaction  │
          │  6. Update ledger       │
          └─────────────────────────┘
```

---

## 🔁 Example Flow

A typical demonstration follows this sequence:

### 1. Create a payment

The demo creates a payment instruction containing sender, receiver, amount, nonce and timestamp.

The payload is encrypted and wrapped inside a mesh packet.

### 2. Propagate through the mesh

The packet is inserted into a virtual device and distributed through the simulated mesh using gossip rounds.

The TTL decreases as the packet moves through the network.

### 3. Bridge reaches the backend

A simulated bridge device represents a phone that has regained internet connectivity.

It sends the packets it has collected to:

```text
POST /api/bridge/ingest
```

### 4. Backend validates the packet

The backend:

```text
Ciphertext
    ↓
SHA-256 hash
    ↓
Idempotency check
    ↓
Decryption
    ↓
Freshness validation
    ↓
Transaction settlement
```

### 5. Transaction is recorded

If validation succeeds, the account balances and transaction ledger are updated.

---

## 🧩 Project Structure

```text
src/
├── main/
│   ├── java/com/demo/upimesh/
│   │   ├── config/
│   │   ├── controller/
│   │   ├── crypto/
│   │   ├── model/
│   │   └── service/
│   │
│   └── resources/
│       ├── application.properties
│       └── templates/
│           └── dashboard.html
│
└── test/
    └── java/com/demo/upimesh/
        └── IdempotencyConcurrencyTest.java
```

### Important components

| Component                | Responsibility                               |
| ------------------------ | -------------------------------------------- |
| `HybridCryptoService`    | Encryption, decryption and packet hashing    |
| `ServerKeyHolder`        | Generates the server RSA key pair            |
| `MeshSimulatorService`   | Simulates packet propagation                 |
| `VirtualDevice`          | Represents a device in the simulated network |
| `IdempotencyService`     | Prevents duplicate packet processing         |
| `BridgeIngestionService` | Handles packets received from bridge devices |
| `SettlementService`      | Performs account and ledger updates          |
| `DemoService`            | Creates demo accounts and payments           |
| `ApiController`          | REST API endpoints                           |
| `DashboardController`    | Serves the demonstration dashboard           |

---

## 🛠️ Tech Stack

**Backend**

* Java 17+
* Spring Boot 3.3
* Spring Web
* Spring Data JPA
* Hibernate

**Database**

* H2 in-memory database

**Security / Cryptography**

* RSA-OAEP
* AES-256-GCM
* SHA-256

**Build**

* Maven Wrapper

**Frontend**

* HTML dashboard served by Spring Boot

---

## 🚀 Running the Project

### Requirements

* JDK 17 or newer
* No external database required
* No separate Maven installation required

### Windows

```cmd
mvnw.cmd spring-boot:run
```

### Linux / macOS

```bash
./mvnw spring-boot:run
```

After startup, open:

```text
http://localhost:8080
```

The H2 database console is available at:

```text
http://localhost:8080/h2-console
```

Database configuration:

```text
JDBC URL: jdbc:h2:mem:upimesh
Username: sa
Password: [empty]
```

---

## 🔌 API Endpoints

| Method | Endpoint             | Purpose                                 |
| ------ | -------------------- | --------------------------------------- |
| GET    | `/`                  | Open the demo dashboard                 |
| GET    | `/api/server-key`    | Retrieve the server public key          |
| GET    | `/api/accounts`      | View demo accounts                      |
| GET    | `/api/transactions`  | View recent transactions                |
| GET    | `/api/mesh/state`    | Inspect virtual device state            |
| POST   | `/api/demo/send`     | Generate and inject a payment           |
| POST   | `/api/mesh/gossip`   | Execute a mesh propagation round        |
| POST   | `/api/mesh/flush`    | Upload packets from bridge devices      |
| POST   | `/api/mesh/reset`    | Reset the simulated environment         |
| POST   | `/api/bridge/ingest` | Process a packet received from a bridge |

---

## 🧪 Testing

Run the complete test suite with:

```cmd
mvnw.cmd test
```

The tests cover:

* Encryption/decryption round trips
* Detection of modified ciphertext
* Concurrent duplicate delivery

The concurrency scenario is particularly important: multiple simulated bridge requests attempt to process the same packet, while the system verifies that the payment is settled only once.

---

## 🔍 Design Highlights

### Why hybrid encryption?

RSA is used to protect the AES session key, while AES-GCM handles the actual payment payload.

This combines:

```text
RSA → key protection
AES-GCM → payload encryption + integrity
```

### Why use the ciphertext hash for idempotency?

The backend can identify duplicate packets before performing expensive decryption.

The hash also provides a stable identifier for identical encrypted deliveries.

### Why use a TTL?

The TTL limits how far a packet can propagate through the simulated mesh and prevents indefinite forwarding.

---

## ⚠️ Current Limitations

This project is intentionally a simulation rather than a production payment system.

Current limitations include:

* The mesh network is simulated in software.
* No real Bluetooth/BLE communication is implemented.
* H2 is used instead of a production database.
* Idempotency state is local to the JVM.
* RSA keys are generated when the application starts.
* Bridge authentication is not implemented.
* There is no integration with banks, NPCI or real UPI infrastructure.
* Offline balance verification and real-world double-spending prevention are outside the scope of this prototype.

A production implementation would require persistent key management, distributed infrastructure, authenticated bridge devices, real mobile networking and integration with the banking/payment ecosystem.

---

## 🎯 Learning Objectives

This project can be used to explore several backend and distributed-system concepts together:

* Spring Boot application architecture
* REST API development
* JPA transactions
* Concurrent request handling
* Idempotent APIs
* Optimistic locking
* Hybrid cryptography
* Replay protection
* Distributed packet propagation
* Transaction processing
* Automated testing

---

## 📌 Project Status

**Prototype / Educational Project**

The current implementation is designed to demonstrate the architecture and backend mechanisms involved in a hypothetical mesh-routed payment system rather than provide a production-ready payment service.
