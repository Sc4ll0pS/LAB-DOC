# เอกสารการออกแบบฐานข้อมูล (Database Design Specification)
## โครงการ: Mactile
**เวอร์ชัน:** 1.0.0 (MVP)  
**วันที่จัดทำ:** 16 สิงหาคม 2026  
**สถานะ:** เอกสารการออกแบบฐานข้อมูลฉบับสมบูรณ์ (Database Baseline)

---

## 1. ภาพรวมและมาตรฐานการออกแบบ (Database Overview & Conventions)

ระบบ **Mactile** ได้รับการออกแบบฐานข้อมูลเชิงสัมพันธ์ (Relational Database) เพื่อรองรับการจัดการสิทธิ์ การจองคิว การออก Credential ชั่วคราว และบันทึกประวัติการ Reset สภาพแวดล้อมเครื่องอย่างรัดกุม

### 1.1 เทคโนโลยีฐานข้อมูล (Database Engine)
- **ระยะ MVP / Development:** `SQLite 3` (เปิดใช้งาน Foreign Key Constraints และ WAL Mode)
- **ระยะ Production:** `PostgreSQL 15+` (รองรับ High Concurrency และ GiST Exclusion Constraints)

### 1.2 ข้อตกลงในการตั้งชื่อ (Naming Conventions)
- **ชื่อตาราง (Table Names):** ใช้ตัวพิมพ์เล็กแบบพหูพจน์ (Snake Case, Plural) เช่น `users`, `hosts`, `bookings`
- **ชื่อคอลัมน์ (Column Names):** ใช้ตัวพิมพ์เล็กแบ่งด้วยขีดล่าง (Snake Case) เช่น `created_at`, `remote_type`
- **คีย์หลัก (Primary Key):** ตั้งชื่อว่า `id` (Auto-incrementing Integer หรือ UUID)
- **คีย์นอก (Foreign Key):** ตั้งชื่อตามรูปแบบ `<singular_table_name>_id` เช่น `user_id`, `host_id`
- **วันเวลา (Timestamps):** จัดเก็บในรูปแบบ **UTC ISO 8601** (`YYYY-MM-DD HH:MM:SSZ`)

---

## 2. แผนผังความสัมพันธ์ของข้อมูล (Entity-Relationship Diagram - ERD)

```mermaid
erDiagram
    USERS ||--o{ BOOKINGS : "places"
    USERS ||--o{ AUDIT_LOGS : "performs"
    USERS ||--o{ RESET_LOGS : "triggers (if admin)"
    
    HOSTS ||--o{ BOOKINGS : "allocated_to"
    HOSTS ||--o{ SESSIONS : "runs"
    HOSTS ||--o{ RESET_LOGS : "resets"

    BOOKINGS ||--o| SESSIONS : "initiates"
    BOOKINGS ||--o{ RESET_LOGS : "causes_cleanup"

    USERS {
        INTEGER id PK
        VARCHAR(64) username UK
        VARCHAR(128) email UK
        VARCHAR(255) password_hash
        VARCHAR(20) role "USER | ADMIN"
        VARCHAR(20) status "ACTIVE | SUSPENDED"
        DATETIME created_at
        DATETIME updated_at
    }

    HOSTS {
        INTEGER id PK
        VARCHAR(64) name UK "e.g. Mac-Mini-M2-01"
        VARCHAR(100) hostname
        VARCHAR(45) ip_address
        VARCHAR(17) mac_address
        VARCHAR(50) os_version
        VARCHAR(50) specs_cpu
        VARCHAR(20) specs_ram
        VARCHAR(20) remote_type "rustdesk | vnc | ssh"
        VARCHAR(100) remote_id "RustDesk ID or Port"
        VARCHAR(20) status "available | in_use | resetting | maintenance | offline"
        BOOLEAN is_active
        DATETIME last_heartbeat_at
        DATETIME created_at
        DATETIME updated_at
    }

    BOOKINGS {
        INTEGER id PK
        INTEGER user_id FK
        INTEGER host_id FK
        DATETIME start_time
        DATETIME end_time
        VARCHAR(20) status "scheduled | active | completed | cancelled | terminated"
        VARCHAR(64) temp_remote_password "Hashed / Encrypted"
        VARCHAR(255) cancellation_reason
        DATETIME created_at
        DATETIME updated_at
    }

    SESSIONS {
        INTEGER id PK
        INTEGER booking_id FK UK
        INTEGER user_id FK
        INTEGER host_id FK
        DATETIME connected_at
        DATETIME disconnected_at
        INTEGER duration_seconds
        VARCHAR(30) termination_reason "normal | timeout | admin_force | host_error"
        VARCHAR(45) client_ip
        DATETIME created_at
    }

    RESET_LOGS {
        INTEGER id PK
        INTEGER host_id FK
        INTEGER booking_id FK "Nullable"
        INTEGER triggered_by_user_id FK "Nullable (Admin ID if manual)"
        VARCHAR(30) trigger_type "auto_post_session | manual_admin | emergency_reboot"
        DATETIME started_at
        DATETIME completed_at
        INTEGER duration_seconds
        VARCHAR(20) status "in_progress | success | failed"
        INTEGER exit_code
        TEXT log_output
        DATETIME created_at
    }

    AUDIT_LOGS {
        INTEGER id PK
        INTEGER user_id FK "Nullable"
        VARCHAR(50) action "LOGIN | CREATE_BOOKING | FORCE_DISCONNECT | TRIGGER_RESET"
        VARCHAR(50) entity_type "booking | host | user | system"
        INTEGER entity_id "Nullable"
        TEXT details "JSON payload"
        VARCHAR(45) ip_address
        DATETIME created_at
    }

    SYSTEM_SETTINGS {
        INTEGER id PK
        VARCHAR(64) setting_key UK
        TEXT setting_value
        VARCHAR(255) description
        DATETIME updated_at
    }
```

---

## 3. พจนานุกรมข้อมูลและโครงสร้างตาราง (Data Dictionary)

### 3.1 ตาราง `users` (ตารางผู้ใช้งานและผู้ดูแลระบบ)
เก็บข้อมูลบัญชีผู้ใช้ สิทธิ์การเข้าถึง และสถานะบัญชี

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | INTEGER | NO | Auto-Inc | รหัสประจำตัวผู้ใช้ (Primary Key) |
| `username` | VARCHAR(64) | NO | - | ชื่อบัญชีผู้ใช้ (Unique) |
| `email` | VARCHAR(128) | NO | - | อีเมลผู้ใช้ (Unique) |
| `password_hash` | VARCHAR(255) | NO | - | รหัสผ่านที่ผ่านการเข้ารหัส (Bcrypt / Argon2id) |
| `role` | VARCHAR(20) | NO | `'USER'` | สิทธิ์การใช้งาน (`'USER'`, `'ADMIN'`) |
| `status` | VARCHAR(20) | NO | `'ACTIVE'` | สถานะบัญชี (`'ACTIVE'`, `'SUSPENDED'`) |
| `created_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่สร้างบัญชี |
| `updated_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่แก้ไขล่าสุด |

---

### 3.2 ตาราง `hosts` (ตารางเครื่อง Mac Host)
เก็บข้อมูลเครื่อง Mac ในระบบ สเปกเครื่อง และสถานะการทำงานปัจจุบัน

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | INTEGER | NO | Auto-Inc | รหัสประจำตัวเครื่อง Host (Primary Key) |
| `name` | VARCHAR(64) | NO | - | ชื่อเรียกเครื่อง (เช่น `Mac-Mini-M2-01`) |
| `hostname` | VARCHAR(100) | YES | NULL | Hostname ภายในเครือข่าย |
| `ip_address` | VARCHAR(45) | NO | - | IPv4 หรือ IPv6 สำหรับเชื่อมต่อ |
| `mac_address` | VARCHAR(17) | YES | NULL | Hardware MAC Address สำหรับ Wake-on-LAN |
| `os_version` | VARCHAR(50) | YES | NULL | เวอร์ชันระบบปฏิบัติการ (เช่น `macOS Sequoia 15.1`) |
| `specs_cpu` | VARCHAR(50) | YES | NULL | ข้อมูล CPU (เช่น `Apple M2 8-Core`) |
| `specs_ram` | VARCHAR(20) | YES | NULL | ขนาดหน่วยความจำ (เช่น `16 GB`) |
| `remote_type` | VARCHAR(20) | NO | `'rustdesk'` | ชนิดโปรแกรมรีโมท (`'rustdesk'`, `'vnc'`, `'ssh'`) |
| `remote_id` | VARCHAR(100) | YES | NULL | Connection ID ของโปรแกรมรีโมท (เช่น RustDesk ID) |
| `status` | VARCHAR(20) | NO | `'available'` | สถานะปัจจุบัน (`'available'`, `'in_use'`, `'resetting'`, `'maintenance'`, `'offline'`) |
| `is_active` | BOOLEAN | NO | `TRUE` | เปิด/ปิดการใช้งานเครื่องในระบบ |
| `last_heartbeat_at` | DATETIME | YES | NULL | เวลาที่ Agent ส่งสัญญาณตรวจสอบสถานะล่าสุด |
| `created_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่ลงทะเบียนเครื่อง |
| `updated_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่แก้ไขข้อมูลล่าสุด |

---

### 3.3 ตาราง `bookings` (ตารางการจองคิว)
เก็บรายการจองคิวล่วงหน้า ช่วงเวลา และรหัสผ่านชั่วคราว (Ephemeral Credential)

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | INTEGER | NO | Auto-Inc | รหัสการจอง (Primary Key) |
| `user_id` | INTEGER | NO | - | รหัสผู้จอง (Foreign Key -> `users.id`) |
| `host_id` | INTEGER | NO | - | รหัสเครื่องที่จอง (Foreign Key -> `hosts.id`) |
| `start_time` | DATETIME | NO | - | เวลาเริ่มต้นการจอง (UTC) |
| `end_time` | DATETIME | NO | - | เวลาสิ้นสุดการจอง (UTC) |
| `status` | VARCHAR(20) | NO | `'scheduled'` | สถานะการจอง (`'scheduled'`, `'active'`, `'completed'`, `'cancelled'`, `'terminated'`) |
| `temp_remote_password` | VARCHAR(64) | YES | NULL | รหัสผ่านรีโมทชั่วคราว (สุ่มใหม่ในรอบการจอง) |
| `cancellation_reason`| VARCHAR(255) | YES | NULL | เหตุผลที่ยกเลิกการจอง (หากมี) |
| `created_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่ทำการจอง |
| `updated_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่อัปเดตสถานะล่าสุด |

---

### 3.4 ตาราง `sessions` (ตารางบันทึกการเชื่อมต่อจริง)
เก็บข้อมูลการเชื่อมต่อจริงของผู้ใช้ ระยะเวลาที่ใช้งานจริง และสาเหตุการยุติ Session

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | INTEGER | NO | Auto-Inc | รหัส Session (Primary Key) |
| `booking_id` | INTEGER | NO | - | รหัสการจองที่เกี่ยวข้อง (FK -> `bookings.id`, Unique) |
| `user_id` | INTEGER | NO | - | รหัสผู้ใช้งาน (FK -> `users.id`) |
| `host_id` | INTEGER | NO | - | รหัสเครื่อง Mac (FK -> `hosts.id`) |
| `connected_at` | DATETIME | YES | NULL | เวลาที่ผู้ใช้เริ่มเชื่อมต่อจริง (UTC) |
| `disconnected_at`| DATETIME | YES | NULL | เวลาที่ตัดการเชื่อมต่อ (UTC) |
| `duration_seconds` | INTEGER | YES | 0 | ระยะเวลาเชื่อมต่อรวม (วินาที) |
| `termination_reason` | VARCHAR(30) | NO | `'normal'` | สาเหตุสิ้นสุด (`'normal'`, `'timeout'`, `'admin_force'`, `'host_error'`) |
| `client_ip` | VARCHAR(45) | YES | NULL | IP Address ของเครื่องฝั่ง Client ที่เชื่อมต่อ |
| `created_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่สร้างเรคคอร์ด |

---

### 3.5 ตาราง `reset_logs` (ตารางบันทึกประวัติการรีเซ็ตเครื่อง)
เก็บบันทึกประวัติการล้างเครื่อง ระยะเวลา และผลลัพธ์ของ Script

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | INTEGER | NO | Auto-Inc | รหัส Log (Primary Key) |
| `host_id` | INTEGER | NO | - | รหัสเครื่อง Mac (FK -> `hosts.id`) |
| `booking_id` | INTEGER | YES | NULL | รหัสการจองที่ส่งผลให้เกิดการ Reset (ถ้ามี) |
| `triggered_by_user_id` | INTEGER | YES | NULL | รหัส Admin ที่สั่ง Reset แบบ Manual (ถ้ามี) |
| `trigger_type` | VARCHAR(30) | NO | `'auto_post_session'` | ประเภทคำสั่ง (`'auto_post_session'`, `'manual_admin'`, `'emergency_reboot'`) |
| `started_at` | DATETIME | NO | CURRENT_TIMESTAMP | เวลาเริ่มรันสคริปต์ |
| `completed_at` | DATETIME | YES | NULL | เวลาที่รันสคริปต์เสร็จสิ้น |
| `duration_seconds` | INTEGER | YES | NULL | ระยะเวลาที่ใช้ในการรีเซ็ต (วินาที) |
| `status` | VARCHAR(20) | NO | `'in_progress'` | ผลการรัน (`'in_progress'`, `'success'`, `'failed'`) |
| `exit_code` | INTEGER | YES | NULL | Exit Code จากกระบวนการรัน Script (0 = ปกติ) |
| `log_output` | TEXT | YES | NULL | รายละเอียด Console Output / Error Log |
| `created_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่สร้างเรคคอร์ด |

---

### 3.6 ตาราง `audit_logs` (ตารางบันทึกพฤติกรรมระบบ)
เก็บบันทึก Audit Trail สำหรับการตรวจสอบย้อนหลัง

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | INTEGER | NO | Auto-Inc | รหัส Audit Log (Primary Key) |
| `user_id` | INTEGER | YES | NULL | ผู้กระทำ (FK -> `users.id`, Null หากเป็น System) |
| `action` | VARCHAR(50) | NO | - | ชื่อการกระทำ (เช่น `LOGIN`, `FORCE_DISCONNECT`) |
| `entity_type` | VARCHAR(50) | NO | - | ชนิดของข้อมูลเป้าหมาย (`booking`, `host`, `user`, `system`) |
| `entity_id` | INTEGER | YES | NULL | ID ของข้อมูลเป้าหมาย |
| `details` | TEXT | YES | NULL | ข้อมูลรายละเอียดเพิ่มเติมในรูปแบบ JSON Payload |
| `ip_address` | VARCHAR(45) | YES | NULL | IP Address ของผู้ส่งคำขอ |
| `created_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่บันทึก |

---

### 3.7 ตาราง `system_settings` (ตารางตั้งค่าระบบ)
เก็บค่า Configuration ส่วนกลางของระบบ

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | INTEGER | NO | Auto-Inc | รหัสการตั้งค่า (Primary Key) |
| `setting_key` | VARCHAR(64) | NO | - | ชื่อคีย์ตั้งค่า (Unique) |
| `setting_value` | TEXT | NO | - | ค่าของการตั้งค่า |
| `description` | VARCHAR(255) | YES | NULL | คำอธิบายการตั้งค่า |
| `updated_at` | DATETIME | NO | CURRENT_TIMESTAMP | วันเวลาที่อัปเดตล่าสุด |

---

## 4. ดัชนีและการเพิ่มประสิทธิภาพ (Indexes & Constraints)

### 4.1 Index รายการสำคัญ
```sql
-- 1. Index ป้องกันการค้นหาและตรวจเช็คช่วงเวลาจองซ้อน (Conflict Check Optimization)
CREATE INDEX idx_bookings_host_time_status ON bookings(host_id, start_time, end_time, status);

-- 2. Index สำหรับค้นหารายการจองของผู้ใช้ (My Bookings View)
CREATE INDEX idx_bookings_user_id ON bookings(user_id, start_time DESC);

-- 3. Index สำหรับดึงรายการ Host ตามสถานะเพื่อแสดงบน Live Matrix
CREATE INDEX idx_hosts_status_active ON hosts(status, is_active);

-- 4. Index สำหรับสืบค้น Audit Log และ Reset Log ย้อนหลัง
CREATE INDEX idx_reset_logs_host_id ON reset_logs(host_id, started_at DESC);
CREATE INDEX idx_audit_logs_user_action ON audit_logs(user_id, action, created_at DESC);
```

### 4.2 ตรรกะการตรวจสอบการจองซ้อน (Double-Booking Validation Query)
เมื่อมีการยื่นคำขอจองช่วงเวลา `[:new_start, :new_end]` บนเครื่อง `[:host_id]` ระบบจะทำการตรวจสอบด้วย Query ต่อไปนี้ภายใต้ Transaction:

```sql
SELECT COUNT(*) 
FROM bookings 
WHERE host_id = :host_id 
  AND status IN ('scheduled', 'active')
  AND NOT (end_time <= :new_start OR start_time >= :new_end);
```
*หากผลลัพธ์มากกว่า 0 ระบบจะปฏิเสธการจองทันที (Conflict Detected)*

---

## 5. SQL DDL Schema Script (SQLite & PostgreSQL Compatible)

```sql
-- ==========================================================
-- Mactile Database DDL Script (MVP Baseline)
-- ==========================================================

-- 1. ตาราง users
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(64) NOT NULL UNIQUE,
    email VARCHAR(128) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'USER' CHECK (role IN ('USER', 'ADMIN')),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'SUSPENDED')),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 2. ตาราง hosts
CREATE TABLE IF NOT EXISTS hosts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(64) NOT NULL UNIQUE,
    hostname VARCHAR(100),
    ip_address VARCHAR(45) NOT NULL,
    mac_address VARCHAR(17),
    os_version VARCHAR(50),
    specs_cpu VARCHAR(50),
    specs_ram VARCHAR(20),
    remote_type VARCHAR(20) NOT NULL DEFAULT 'rustdesk' CHECK (remote_type IN ('rustdesk', 'vnc', 'ssh')),
    remote_id VARCHAR(100),
    status VARCHAR(20) NOT NULL DEFAULT 'available' CHECK (status IN ('available', 'in_use', 'resetting', 'maintenance', 'offline')),
    is_active BOOLEAN NOT NULL DEFAULT 1,
    last_heartbeat_at DATETIME,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 3. ตาราง bookings
CREATE TABLE IF NOT EXISTS bookings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    host_id INTEGER NOT NULL,
    start_time DATETIME NOT NULL,
    end_time DATETIME NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled' CHECK (status IN ('scheduled', 'active', 'completed', 'cancelled', 'terminated')),
    temp_remote_password VARCHAR(64),
    cancellation_reason VARCHAR(255),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT,
    FOREIGN KEY (host_id) REFERENCES hosts(id) ON DELETE RESTRICT
);

-- 4. ตาราง sessions
CREATE TABLE IF NOT EXISTS sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    booking_id INTEGER NOT NULL UNIQUE,
    user_id INTEGER NOT NULL,
    host_id INTEGER NOT NULL,
    connected_at DATETIME,
    disconnected_at DATETIME,
    duration_seconds INTEGER DEFAULT 0,
    termination_reason VARCHAR(30) NOT NULL DEFAULT 'normal' CHECK (termination_reason IN ('normal', 'timeout', 'admin_force', 'host_error')),
    client_ip VARCHAR(45),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (booking_id) REFERENCES bookings(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT,
    FOREIGN KEY (host_id) REFERENCES hosts(id) ON DELETE RESTRICT
);

-- 5. ตาราง reset_logs
CREATE TABLE IF NOT EXISTS reset_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    host_id INTEGER NOT NULL,
    booking_id INTEGER,
    triggered_by_user_id INTEGER,
    trigger_type VARCHAR(30) NOT NULL DEFAULT 'auto_post_session' CHECK (trigger_type IN ('auto_post_session', 'manual_admin', 'emergency_reboot')),
    started_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at DATETIME,
    duration_seconds INTEGER,
    status VARCHAR(20) NOT NULL DEFAULT 'in_progress' CHECK (status IN ('in_progress', 'success', 'failed')),
    exit_code INTEGER,
    log_output TEXT,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (host_id) REFERENCES hosts(id) ON DELETE CASCADE,
    FOREIGN KEY (booking_id) REFERENCES bookings(id) ON DELETE SET NULL,
    FOREIGN KEY (triggered_by_user_id) REFERENCES users(id) ON DELETE SET NULL
);

-- 6. ตาราง audit_logs
CREATE TABLE IF NOT EXISTS audit_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id INTEGER,
    details TEXT,
    ip_address VARCHAR(45),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);

-- 7. ตาราง system_settings
CREATE TABLE IF NOT EXISTS system_settings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    setting_key VARCHAR(64) NOT NULL UNIQUE,
    setting_value TEXT NOT NULL,
    description VARCHAR(255),
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ==========================================================
-- สร้างดัชนี (Indexes)
-- ==========================================================
CREATE INDEX IF NOT EXISTS idx_bookings_host_time_status ON bookings(host_id, start_time, end_time, status);
CREATE INDEX IF NOT EXISTS idx_bookings_user_id ON bookings(user_id, start_time DESC);
CREATE INDEX IF NOT EXISTS idx_hosts_status_active ON hosts(status, is_active);
CREATE INDEX IF NOT EXISTS idx_reset_logs_host_id ON reset_logs(host_id, started_at DESC);
CREATE INDEX IF NOT EXISTS idx_audit_logs_user_action ON audit_logs(user_id, action, created_at DESC);
```

---

## 6. ข้อมูลเริ่มต้นระบบ (Seed / Initial Data)

```sql
-- 1. บัญชีเริ่มต้น (Default Admin & Test User)
-- Password Hashed: 'Admin@1234' และ 'User@1234' (Example Bcrypt hashes)
INSERT INTO users (username, email, password_hash, role, status) VALUES
('admin', 'admin@mactile.local', '$2b$12$e7kC62W...sampleAdminHash...', 'ADMIN', 'ACTIVE'),
('developer1', 'dev1@mactile.local', '$2b$12$k8lP91Q...sampleUserHash...', 'USER', 'ACTIVE');

-- 2. ข้อมูลเครื่อง Mac Host เริ่มต้น
INSERT INTO hosts (name, hostname, ip_address, specs_cpu, specs_ram, remote_type, remote_id, status, is_active) VALUES
('Mac-Mini-M2-01', 'mac-mini-01.local', '192.168.1.101', 'Apple M2 8-Core', '16 GB', 'rustdesk', '981244512', 'available', 1),
('Mac-Mini-M2-02', 'mac-mini-02.local', '192.168.1.102', 'Apple M2 8-Core', '16 GB', 'rustdesk', '981244513', 'available', 1),
('Mac-Studio-M2Max-01', 'mac-studio-01.local', '192.168.1.105', 'Apple M2 Max 12-Core', '32 GB', 'vnc', '5900', 'available', 1);

-- 3. ค่าตั้งค่าระบบเริ่มต้น
INSERT INTO system_settings (setting_key, setting_value, description) VALUES
('MAX_SLOT_DURATION_MINUTES', '120', 'ระยะเวลาสูงสุดที่อนุญาตให้จองต่อ 1 สล็อต (นาที)'),
('SLOT_BUFFER_MINUTES', '5', 'ระยะเวลาบัฟเฟอร์สำหรับกระบวนการ Reset ระหว่างสล็อต (นาที)'),
('EXPIRY_WARNING_MINUTES', '5', 'เวลาแจ้งเตือนล่วงหน้าก่อน Session หมดเวลา (นาที)'),
('DEFAULT_REMOTE_TYPE', 'rustdesk', 'โปรแกรมรีโมทเริ่มต้นสำหรับเครื่องใหม่');
```
