# Biometric Attendance System: Project Plan

A college-classroom attendance system. A student scans a finger on an ESP32 device, the backend records attendance once per class session, and the teacher watches it live on a React dashboard.

- **Timeline:** 2 days
- **Deployment:** local only
- **Hardware now:** ESP32 only (no fingerprint sensor yet), so the sensor is mocked
- **Privacy rule:** fingerprint templates live only on the sensor. The server stores `slot_id -> student` and never any fingerprint data.

---

## 1. Scope

### In scope
- Teacher login (JWT)
- Create classes, add students
- Enroll a student (device is told to enroll, reports back the slot ID)
- Start and close a class session
- Scan from the device (simulated) marks the student present, **once per session**
- Live attendance feed on the dashboard
- Present and absent lists per session
- CSV export
- Simple session history

### Out of scope (for now)
- Real fingerprint matching (R307/AS608)
- Multiple devices per room, check-out, shifts
- Cloud deployment, mobile app, student login

---

## 2. Architecture

```
ESP32 (MockSensor) --MQTT--> Mosquitto --> FastAPI backend --> PostgreSQL
                                                |
                                      WebSocket | REST (JWT)
                                                v
                                       React dashboard
```

All infrastructure runs locally. Docker Compose runs PostgreSQL and Mosquitto. The backend and frontend run directly on the laptop.

### Tech stack
| Layer | Choice |
|---|---|
| Microcontroller | ESP32 (Arduino framework, C++) |
| Sensor | `MockSensor` now, `R307Sensor` later (same interface) |
| Transport | MQTT (Mosquitto) |
| Backend | Python, FastAPI, SQLAlchemy, Alembic, paho-mqtt |
| Database | PostgreSQL |
| Auth | JWT, bcrypt password hashing |
| Frontend | React + Vite + Tailwind CSS |
| Live updates | WebSocket |
| Dev tooling | Docker Compose, Git/GitHub, Antigravity |

---

## 3. Mock sensor design

The firmware depends on an interface, not on specific hardware.

```cpp
class FingerprintSensor {
 public:
  virtual void begin() = 0;
  virtual int  scan() = 0;                 // returns slot_id, -1 = no finger, -2 = no match
  virtual bool enroll(int slot_id) = 0;    // returns true on success
};
```

- `MockSensor`: a scan happens when you type `scan 3` in the Serial Monitor, or when the BOOT button is pressed (cycles through slot IDs 1..N). `scan 99` simulates an unknown finger. `enroll` always succeeds after a short delay and an LED blink.
- `R307Sensor`: a stub for later. Swapping hardware must be a one-line change in `main.cpp`.
- A Python **device simulator** (`simulator/device_sim.py`) speaks the same MQTT protocol, so the backend and dashboard can be tested without the board.

---

## 4. Data model

| Table | Columns |
|---|---|
| `teachers` | id, name, email (unique), password_hash |
| `classes` | id, name, subject, teacher_id |
| `students` | id, name, roll_no (unique), class_id, fingerprint_slot (unique, nullable) |
| `sessions` | id, class_id, started_at, ended_at, status (`open` / `closed`) |
| `attendance` | id, session_id, student_id, scanned_at, **UNIQUE(session_id, student_id)** |
| `devices` | id, name, last_seen |

Rules:
- Only one `open` session per class at a time.
- The unique constraint on `attendance` is what enforces "once per session". Handle the violation gracefully and return `duplicate`.
- Slot IDs are unique per device. For this project there is one device, so `fingerprint_slot` is unique globally.

---

## 5. MQTT protocol

Device ID example: `esp32-01`.

| Topic | Direction | Payload |
|---|---|---|
| `attendance/{device}/scan` | device -> server | `{"slot_id": 3, "ts": 1730000000}` |
| `attendance/{device}/result` | server -> device | `{"status": "ok", "name": "Asha"}` |
| `attendance/{device}/cmd` | server -> device | `{"cmd": "enroll", "slot_id": 3}` |
| `attendance/{device}/enroll_done` | device -> server | `{"slot_id": 3, "ok": true}` |
| `attendance/{device}/heartbeat` | device -> server | `{"ts": 1730000000}` |

`result.status` values: `ok`, `duplicate`, `unknown`, `no_session`.

Device feedback:
- `ok`: green LED + short beep + name on serial (or OLED later)
- `duplicate`: yellow LED
- `unknown` / `no_session`: red LED + long beep

---

## 6. REST API

All routes except login require `Authorization: Bearer <jwt>`.

| Method | Route | Purpose |
|---|---|---|
| POST | `/auth/login` | Returns JWT |
| GET / POST | `/classes` | List / create classes |
| GET / POST | `/students` | List (filter by class) / create |
| PUT / DELETE | `/students/{id}` | Edit / remove |
| POST | `/students/{id}/enroll` | Allocates a slot ID, publishes the enroll command |
| POST | `/sessions` | Start a session for a class |
| POST | `/sessions/{id}/close` | Close a session |
| GET | `/sessions` | History |
| GET | `/sessions/{id}/attendance` | Present and absent lists |
| GET | `/sessions/{id}/export.csv` | CSV export |
| WS | `/ws/live` | Pushes each scan event to the dashboard |

---

## 7. Folder structure

```
attendance-system/
├── docker-compose.yml
├── PLAN.md
├── README.md
├── .gitignore
├── .env.example
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── db.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── auth.py
│   │   ├── mqtt_client.py
│   │   └── routers/ (auth.py, classes.py, students.py, sessions.py, ws.py)
│   ├── alembic/
│   ├── seed.py
│   └── requirements.txt
├── frontend/
│   └── src/ (pages/, components/, api.js, ws.js)
├── firmware/
│   └── esp32/ (main.cpp, sensor.h, mock_sensor.h, r307_sensor.h, config.example.h)
├── simulator/
│   └── device_sim.py
└── mosquitto/
    └── mosquitto.conf
```

---

## 8. Milestones

Each milestone ends with a working, testable state and a Git commit.

### Day 1: working system without a UI

**M1: Skeleton and infrastructure (~1 hr)**
- Create the repo structure, `.gitignore`, `.env.example`
- `docker-compose.yml` with PostgreSQL and Mosquitto
- Backend project with config loading and DB connection
- SQLAlchemy models and the first Alembic migration
- **Done when:** `docker compose up -d` starts both services and the tables exist.

**M2: Backend core (~2 hr)**
- Auth (login, JWT, password hashing)
- CRUD for classes and students
- Sessions start/close with the one-open-session rule
- Attendance read endpoints
- `seed.py` with one teacher, one class and a few students
- **Done when:** the whole flow works through Swagger at `/docs`.

**M3: MQTT ingestion and simulator (~1.5 hr)**
- Background MQTT subscriber inside the backend
- Scan handler: slot -> student -> insert attendance -> publish result
- Handle `unknown`, `duplicate`, `no_session`
- `simulator/device_sim.py` (CLI that publishes scans and listens for results)
- **Done when:** a simulated scan creates exactly one attendance row, and repeating it returns `duplicate`.

**M4: ESP32 firmware (~2 hr)**
- Wi-Fi + MQTT client with reconnect
- `FingerprintSensor` interface and `MockSensor`
- Serial and BOOT-button scans, LED/buzzer feedback
- Heartbeat and enroll command handling
- **Done when:** a scan from the real board lands in the database.

### Day 2: dashboard and polish

**M5: React foundation (~3 hr)**
- Vite + Tailwind setup, routing, login page, token storage
- Classes and students pages
- Enroll button per student, showing enrolled status
- **Done when:** a teacher can log in, add a student and trigger enrollment.

**M6: Live session page (~2 hr)**
- Start/close session controls
- WebSocket live feed, present/absent table that updates without refresh
- **Done when:** a scan appears on screen within about a second.

**M7: Reports and polish (~1.5 hr)**
- CSV export, history page
- Loading, empty and error states
- Device online/offline indicator from heartbeats
- **Done when:** exported CSV opens cleanly in Excel.

**M8: Demo readiness (~1.5 hr)**
- Seed data, README with setup steps, demo script
- One full dry run, start to finish
- **Done when:** the entire demo runs in under 5 minutes from a clean start.

### Cut order if time runs short
1. History page
2. CSV export
3. Device online indicator
4. Edit/delete for students

The live session flow (M1 to M6) is the core and must not be cut.

---

## 9. Definition of done

- [ ] Teacher can log in and manage a class and students
- [ ] Enrollment assigns a slot ID and links it to a student
- [ ] A scan marks a student present exactly once per session
- [ ] Unknown fingers and scans with no open session are rejected with correct device feedback
- [ ] Dashboard updates live
- [ ] Absent list is correct when the session closes
- [ ] No fingerprint data is stored anywhere on the server
- [ ] README lets someone else run the project from scratch

---

## 10. Local setup notes

- The ESP32 connects to the laptop's **LAN IP** (not `localhost`) on port 1883. Both must be on the same Wi-Fi.
- Allow port 1883 through the laptop firewall.
- The ESP32 supports **2.4 GHz** Wi-Fi only. If the college network isolates clients, use a phone hotspot.
- Keep Wi-Fi credentials and the MQTT host in `firmware/esp32/config.h`, which is **git-ignored**. Commit only `config.example.h`.
- Never commit `.env`. Commit `.env.example`.

---

## 11. Rules for AI-assisted development (Antigravity)

- Work on **one milestone at a time**. Do not start the next until the current one's "Done when" is met.
- Before coding a milestone, state the plan in a few lines and wait for approval.
- Do not add features, libraries or files outside this plan without asking.
- Keep secrets out of the repo.
- After each milestone: run it, show how it was verified, and suggest a commit message.
- Keep the mock sensor and the real sensor behind the same interface.

---

## 12. Future work

- Plug in the real R307/AS608 sensor by implementing `R307Sensor`
- OLED display on the device
- Multiple devices and rooms
- Late marking based on a grace period
- Student-facing view, notifications to parents
- Cloud deployment (Render/Railway for backend, Vercel for frontend)
