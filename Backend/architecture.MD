# 🏗️ Tech Stack & Architecture

## 1. Tech Stack

* **Framework:** NestJS — Node.js framework using TypeScript

* **Language:** TypeScript

* **Database:** MongoDB Atlas — Cloud-hosted NoSQL document database with indexing

* **Caching & Queues:** Redis *(Planned)* — Cloud Redis / in-memory store for:

  * Caching
  * Nonces
  * Background job queues using BullMQ

* **Architecture Style:** Modular Monolith — Clean separation of concerns across domain modules

---

## 2. Multi-Domain & Multi-Tenant Architecture

### 🌐 Universal Multi-Tenant Support

Designed to support multiple organization types while maintaining strict data isolation using `tenantId` scoping.

Supported organization types:

* 🎓 **Universities**
* 🏥 **Hospitals**
* 🔒 **Jails**
* 📦 **Warehouses**

### 🔐 Role-Based Access Control (RBAC)

Supports dynamic, domain-specific roles:

* `student`
* `teacher`
* `admin`
* `super-admin`
* `inmate`
* `staff`
* `patient`
* `operator`

---

## 3. Core Backend Modules

### 🔑 Auth Module

Handles authentication, authorization, and device security.

* JWT-based authentication
* Refresh token management
* Device UUID hardware locking
* Email verification for device switching

---

### 🏢 Tenants Module

Handles organization provisioning and tenant-level management.

* Multi-tenant organization provisioning
* Subscription tier management
* Supported tiers:

  * `FREE`
  * `BASIC`
  * `PREMIUM`
  * `ENTERPRISE`
* Tenant health status monitoring

---

### 👤 Users / Entities Module

Provides multi-domain entity management with conditional validation.

#### University-specific fields

* Identifiers
* Roles
* Program
* Semester
* Section

#### Other domains

Supports generic domain-specific profiles for:

* Hospitals
* Jails
* Warehouses

---

### 📡 Devices & Hardware Ingestion Module

Handles ESP32 device registration, monitoring, and offline synchronization.

* ESP32 MAC address registration
* 60-second heartbeat monitoring
* Offline-first MicroSD batch synchronization
* `session-batch-sync` support
* MongoDB bulk operations for efficient data ingestion

---

### 📝 Attendance & Classroom Sessions Module

Manages classroom attendance sessions and operational controls.

* On-demand session opening and closing
* Session extension
* Manual attendance/session overrides
* Single-use Redis nonce verification

---

### 👥 Cohorts & Schedules Module

Handles bulk entity management, promotions, archiving, and scheduling.

* Bulk CSV entity imports
* Node.js Streams
* `csv-parser`
* Batch promotions
* Entity archiving mechanisms
* Advanced time-slot creation
* Automated node/host overlap conflict validation
* `$or`-based time-window conflict checks

---

### ⚡ Gateway Module

Provides real-time communication between clients and the backend.

* Socket.io WebSocket integration
* Real-time classroom operational events
* Session state broadcasting
* Live operational updates

---

### 🤖 RAG AI Assistant Module

Provides domain-specific AI assistance using Retrieval-Augmented Generation (RAG).

* Domain-specific policy document uploads
* PDF processing and chunking
* Embedding generation
* MongoDB Atlas Vector Search
* Gemini AI integration
* Context-aware policy retrieval

---

### 📊 Reports Export Module

Handles generation of operational reports and official documents.

#### Excel Reports

* Customized operational logs
* ExcelJS-based generation
* Exportable attendance and operational data

#### PDF Reports

* Official PDF summary generation
* Defaulter sheet generation
* Domain-specific reporting formats

## 4. External API's


* MongoDB Atlas: Cloud-hosted NoSQL Database (External Cloud Service).

* Cloud Redis (Redis Enterprise / Redis Cloud): In-memory cache and BullMQ's backend engine (External Cloud Service).

* Google Gemini API (gemini-1.5-flash & text-embedding-004): External AI API that will be used for text embeddings aur policy document querying inside RAG module.

---

## 🏛️ Architecture Overview

```text
                        ┌───────────────────────┐
                        │      Client Apps      │
                        │ Web / Mobile / ESP32  │
                        └───────────┬───────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │      NestJS API       │
                        │   Modular Monolith    │
                        └───────────┬───────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
   ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
   │ Auth / RBAC  │        │  Tenants &   │        │  Attendance  │
   │              │        │   Entities   │        │   Sessions   │
   └──────────────┘        └──────────────┘        └──────────────┘
          │                         │                         │
          └─────────────────────────┼─────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
             ┌──────────────┐               ┌──────────────┐
             │ MongoDB Atlas│               │    Redis     │
             │              │               │   BullMQ     │
             └──────────────┘               └──────────────┘
                    │
                    ▼
             ┌──────────────┐
             │ Atlas Vector │
             │    Search    │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │  Gemini AI   │
             │ RAG Assistant│
             └──────────────┘
```
