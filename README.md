# 🛡️ Secure Token Allocation System (TappyLine)

[![Go Version](https://img.shields.io/badge/Go-1.22+-00ADD8?style=flat&logo=go)](https://golang.org)
[![React](https://img.shields.io/badge/React-18.3-61DAFB?style=flat&logo=react)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.5-3178C6?style=flat&logo=typescript)](https://www.typescriptlang.org)
[![Vite](https://img.shields.io/badge/Vite-5.4-646CFF?style=flat&logo=vite)](https://vitejs.dev)
[![Gin Framework](https://img.shields.io/badge/Gin-v1.12-008ECF?style=flat&logo=go)](https://gin-gonic.com)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-4169E1?style=flat&logo=postgresql)](https://www.postgresql.org)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

A high-assurance, full-stack token allocation and queue management platform built for hospitals, government document verification centers, administrative service desks, and event check-ins.

The platform provides end-to-end token life-cycle management, zero-knowledge PII data encryption at rest, live queue broadcasting via Server-Sent Events (SSE), offline ticket caching with IndexedDB, dynamic custom form schemas, QR code verification with HMAC-SHA256 digital signatures, and comprehensive role-based administrative dashboards.

---

## 📑 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [Repository Structure (Submodules)](#-repository-structure-submodules)
- [Key Features](#-key-features)
- [Security & Cryptography Model](#-security--cryptography-model)
- [Tech Stack](#-tech-stack)
- [Quick Start & Local Setup](#-quick-start--local-setup)
  - [Prerequisites](#prerequisites)
  - [1. Clone Repository (with Submodules)](#1-clone-repository-with-submodules)
  - [2. Backend Setup](#2-backend-setup)
  - [3. Frontend Setup](#3-frontend-setup)
- [Environment Configuration](#-environment-configuration)
- [API Reference](#-api-reference)
- [Working with Git Submodules](#-working-with-git-submodules)
- [License](#-license)

---

## 🏗️ Architecture Overview

The project is structured as a parent orchestration repository that links the frontend and backend services through **Git Submodules**:

```
                                  ┌────────────────────────────────────────┐
                                  │           Client / Browser             │
                                  │   (Public Kiosk / Mobile / Admin)      │
                                  └──────────────────┬─────────────────────┘
                                                     │
                                       HTTP / REST & SSE Streams
                                                     │
                                                     ▼
                                  ┌────────────────────────────────────────┐
                                  │         Gin Web Framework (Go)         │
                                  │       secure-token-allocation-backend   │
                                  └──────┬───────────────┬───────────────┬─┘
                                         │               │               │
                            AES-256-CFB  │    HMAC-SHA256│   SSE Event   │
                           Data at Rest  │   Cookie Auth │   Broadcaster │
                                         │               │               │
                                         ▼               ▼               ▼
                                  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
                                  │ PostgreSQL  │ │ Admin Audit │ │ Live Client │
                                  │  Database   │ │    Trail    │ │ Connections │
                                  └─────────────┘ └─────────────┘ └─────────────┘
```

---

## 📂 Repository Structure (Submodules)

This parent repository links the two standalone repositories that make up the system:

| Submodule | Description | Repository Link |
| :--- | :--- | :--- |
| **`secure-token-allocation-backend`** | Go (Gin) backend API with GORM, AES-256 PII encryption, SSE queue broadcasting, role-based auth, and XLSX reporting. | [Backend Repo](https://github.com/gurjotsingh1774/secure-token-allocation-backend) |
| **`secure-token-allocation-frontend`** | React 18, TypeScript, and Vite single-page application with responsive kiosk UI, counter QR scanner, and offline IndexedDB caching. | [Frontend Repo](https://github.com/gurjotsingh1774/secure-token-allocation-frontend) |

---

## ✨ Key Features

### 👥 Public User Portal
- **Dynamic Session Discovery:** View real-time active sessions for today with live capacity, opening/closing hours, and waiting counts.
- **Dynamic Form Generation:** Each allocation session supports custom administrator-defined input schemas (e.g. Full Name, Aadhaar/ID, Department, Select dropdowns).
- **Anti-Spam & Idempotency:** Client fingerprinting and UUID-based idempotency keys prevent duplicate and accidental submissions.
- **Real-Time Ticket Streaming:** Server-Sent Events (SSE) deliver instant updates on queue movements, estimated waiting times, and ticket status changes without polling.
- **Offline Ticket Persistence:** Built-in IndexedDB storage preserves issued tickets locally so users never lose their spot if network connectivity drops.
- **Cryptographic QR Pass:** Generates a secure verification QR code containing a backend-signed HMAC-SHA256 signature for physical or counter verification.

### 🛡️ Administration & Counter Desks
- **Ticket Lifecycle Management:** Transition tickets through `PENDING` ➔ `ACCEPTED` (auto-assigns sequential token numbers) ➔ `HOLD` ➔ `SERVED` or `REJECTED`.
- **Live Queue Monitoring:** Real-time administrative dashboard with auto-updating counters, active session states, and instant queue metrics via SSE.
- **Counter QR Scanner:** Built-in web camera QR scanner powered by `jsQR` to instantly verify and lookup tickets presented at counter desks.
- **Zero-Code Form Customizer:** Create sessions with dynamic form fields (text, number, email, select dropdowns) directly from the admin UI.
- **Role-Based Access Control (RBAC):**
  - **`SUPER_ADMIN`:** Full control; create and manage sub-admin accounts, permanently purge sessions, and export decrypted XLSX logs.
  - **`ADMIN`:** Daily operational counter authority; accept, reject, hold, and serve tickets.
- **Excel XLSX Export:** Export comprehensive session request logs with decrypted form responses for record-keeping and audits.
- **Immutable Audit Trail:** Logs sensitive admin actions (session creation, status changes, purges) with timestamp, actor ID, and client IP address.

---

## 🔒 Security & Cryptography Model

1. **AES-256-CFB Encryption at Rest:**
   - All user-submitted Personally Identifiable Information (PII) is encrypted with AES-256 in CFB mode using a 32-byte master key before persisting into the database.
   - PII is never stored in plaintext and is only decrypted on-demand for authorized administrative reviews and XLSX exports.

2. **Tamper-Proof HMAC-SHA256 Session Cookies:**
   - Admins authenticate via signed 12-hour session cookies.
   - Includes an embedded `SessionVersion` counter. Changing admin credentials increments the version counter, immediately invalidating all active sessions across all devices.

3. **HMAC-SHA256 QR Verification:**
   - Public token QR codes embed an HMAC-SHA256 signature generated by the backend secret. Counter staff scanning the QR verify ticket authenticity and prevent counterfeit tokens.

4. **Public PII Masking:**
   - Public status endpoints automatically mask sensitive details (e.g., names and ID numbers) to protect user confidentiality from casual shoulder-surfing.

---

## 💻 Tech Stack

### Backend
- **Language:** Go 1.22+
- **Framework:** Gin Web Framework (`github.com/gin-gonic/gin`)
- **Database & ORM:** PostgreSQL / SQLite with GORM (`gorm.io/gorm`)
- **Streaming:** Server-Sent Events (SSE)
- **Encryption:** `crypto/aes`, `crypto/cipher`, `crypto/hmac`, `crypto/sha256`
- **Excel Generation:** Excelize v2 (`github.com/xuri/excelize/v2`)

### Frontend
- **Framework:** React 18
- **Language:** TypeScript 5.5
- **Build Tool:** Vite 5
- **Icons:** Lucide React (`lucide-react`)
- **Offline Storage:** IndexedDB (`idb`)
- **QR Processing:** `jsQR`
- **Spreadsheet Processing:** SheetJS (`xlsx`)
- **Styling:** Vanilla CSS with custom glassmorphic design and responsive layout

---

## 🚀 Quick Start & Local Setup

### Prerequisites

Ensure you have the following installed on your machine:
- [Git](https://git-scm.com/) (version 2.13+)
- [Go](https://go.dev/dl/) (version 1.22 or newer)
- [Node.js](https://nodejs.org/) (version 18 or newer) & npm
- [PostgreSQL](https://www.postgresql.org/) (or use SQLite fallback)

---

### 1. Clone Repository (with Submodules)

Clone the repository and automatically initialize and clone all submodules:

```bash
# Using SSH (recommended)
git clone --recurse-submodules git@github.com:gurjotsingh1774/secure-token-allocation.git

# Or using HTTPS
git clone --recurse-submodules https://github.com/gurjotsingh1774/secure-token-allocation.git

cd secure-token-allocation
```

> **Note:** If you already cloned without `--recurse-submodules`, initialize them with:
> ```bash
> git submodule update --init --recursive
> ```

---

### 2. Backend Setup

1. Navigate to the backend directory:
   ```bash
   cd secure-token-allocation-backend
   ```

2. Create an `.env` configuration file:
   ```bash
   cp .env.example .env
   ```

3. Update the `.env` file with your configuration:
   ```env
   PORT=8080
   DATABASE_URL=postgres://token_user:token_password_123@localhost:5432/token_db?sslmode=disable
   AES_KEY=Your32CharacterSecretKeyForAES25
   SESSION_SECRET=Your32CharacterSessionSecretKey!
   SUPER_ADMIN_USERNAME=admin
   SUPER_ADMIN_PASSWORD=admin123
   ```

4. Download dependencies and run the server:
   ```bash
   go mod download
   go run main.go
   ```

The backend server will start on `http://localhost:8080`.

---

### 3. Frontend Setup

1. Open a new terminal window and navigate to the frontend directory:
   ```bash
   cd secure-token-allocation-frontend
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Configure environment variables (optional for local development, defaults to proxy):
   ```env
   VITE_BACKEND_URL=http://localhost:8080
   VITE_API_BASE_URL=/api
   ```

4. Launch the Vite development server:
   ```bash
   npm run dev
   ```

Open `http://localhost:5173` in your browser.

---

## ⚙️ Environment Configuration

### Backend (`secure-token-allocation-backend/.env`)

| Variable | Required | Default | Description |
| :--- | :---: | :--- | :--- |
| `PORT` | No | `8080` | Port on which the HTTP server listens |
| `DATABASE_URL` | Yes | Local PostgreSQL | PostgreSQL connection string (supports SQLite fallback) |
| `AES_KEY` | Yes | - | 32-byte encryption key for AES-256 encryption at rest |
| `SESSION_SECRET` | Yes | - | 32-byte secret key for HMAC cookie authentication |
| `SUPER_ADMIN_USERNAME`| Yes | `admin` | Initial Super Admin username seeded on first boot |
| `SUPER_ADMIN_PASSWORD`| Yes | `admin123` | Initial Super Admin password seeded on first boot |

### Frontend (`secure-token-allocation-frontend/.env`)

| Variable | Required | Default | Description |
| :--- | :---: | :--- | :--- |
| `VITE_BACKEND_URL` | No | `http://localhost:8080` | Backend API base URL for Vite dev proxy |
| `VITE_API_BASE_URL` | No | `/api` | Base path prefix for API requests |

---

## 📡 API Reference

### Public Endpoints (`/api/public`)

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/allocations/today` | List active token allocation sessions for today |
| `GET` | `/allocations/stream` | SSE stream for real-time public queue updates |
| `GET` | `/allocations/:id` | Fetch session details and dynamic form fields |
| `POST` | `/allocations/:id/requests` | Submit an encrypted token request |
| `GET` | `/requests/:id/status` | Get ticket progress and queue position (PII masked) |
| `GET` | `/requests/:id/stream` | SSE stream for live updates on an individual ticket |
| `POST` | `/requests/:id/cancel` | Cancel an unapproved ticket |
| `GET` | `/requests/:id/qr-sig` | Obtain backend HMAC-SHA256 signature for verification QR |

### Admin Endpoints (`/api/admin`)

| Method | Endpoint | Role | Description |
| :--- | :--- | :---: | :--- |
| `POST` | `/auth/login` | Public | Admin login, returns signed 12-hour session cookie |
| `GET` | `/me` | Admin | Get currently authenticated admin profile |
| `POST` | `/auth/logout` | Admin | Invalidate and clear session cookie |
| `POST` | `/change-credentials` | Admin | Change credentials & revoke concurrent active sessions |
| `GET` | `/allocations` | Admin | Retrieve all allocation sessions |
| `POST` | `/allocations` | Admin | Create a new session with dynamic form fields |
| `PATCH`| `/allocations/:id` | Admin | Update session status (`ACTIVE`, `PAUSED`, `CLOSED`) |
| `GET` | `/requests` | Admin | List requests with filters (by date, status, session) |
| `POST` | `/requests/:id/accept` | Admin | Approve request & assign sequential token number |
| `POST` | `/requests/:id/reject` | Admin | Reject ticket request |
| `POST` | `/requests/:id/hold` | Admin | Temporarily place ticket on hold |
| `POST` | `/requests/:id/serve` | Admin | Mark token as completed/served |
| `GET` | `/dashboard/stats` | Admin | Fetch aggregate queue metrics & status distributions |
| `GET` | `/dashboard/stream` | Admin | SSE stream for real-time dashboard stat updates |
| `GET` | `/audit-logs` | Admin | Retrieve immutable administrative audit log |
| `GET` | `/users` | Super Admin | List all sub-admin accounts |
| `POST` | `/users` | Super Admin | Provision a new sub-admin account |
| `DELETE`| `/users/:id` | Super Admin | Revoke/delete a sub-admin account |
| `GET` | `/allocations/:id/export`| Super Admin | Export session requests to Excel XLSX with decrypted PII |
| `DELETE`| `/allocations/:id/destroy`| Super Admin | Permanently purge an allocation session and tickets |

---

## 🔄 Working with Git Submodules

### Pulling the Latest Changes
To fetch latest changes for both the parent repo and all submodules:
```bash
git pull --recurse-submodules
git submodule update --remote --merge
```

### Making Changes in a Submodule
1. Enter the submodule directory:
   ```bash
   cd secure-token-allocation-backend
   ```
2. Make your edits, commit, and push to origin:
   ```bash
   git add .
   git commit -m "feat: enhance session validation"
   git push origin main
   ```
3. Return to the root directory to commit the updated submodule reference:
   ```bash
   cd ..
   git add secure-token-allocation-backend
   git commit -m "chore: bump backend submodule reference"
   git push origin main
   ```

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
