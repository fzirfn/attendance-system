# System Design Document (SDD)

## Project Name: Smart Attendance System
**Version:** 1.0.0  
**Status:** Draft / Technical Design  
**Date:** June 8, 2026  
**Authors:** Faiz Irfan & Antigravity  

---

## 1. System Architecture

The Smart Attendance System is designed as a modern web application using a decoupled backend and frontend architecture bound together by the Inertia.js protocol.

```mermaid
graph TD
    subgraph Frontend [Presentation Layer - React SPA]
        React[React Pages & Components]
        Tailwind[TailwindCSS v4 Layouts]
        InertiaClient[Inertia.js Router]
    end

    subgraph Backend [Application Layer - Laravel 12]
        InertiaServer[Inertia.js Adapter]
        Routes[Web/API Routes]
        Middleware[Auth & RBAC Middleware]
        Controllers[Controllers & Form Requests]
        Eloquent[Eloquent Models & Relations]
    end

    subgraph Storage [Database Layer - Relational]
        DB[(MySQL / PostgreSQL)]
    end

    React --> InertiaClient
    InertiaClient -- JSON over HTTPS --> InertiaServer
    InertiaServer --> Routes
    Routes --> Middleware
    Middleware --> Controllers
    Controllers --> Eloquent
    Eloquent --> DB
```

### 1.1 Architecture Components
1.  **Frontend (React SPA):** Renders dynamic, responsive interfaces using React 19. All styles are created using TailwindCSS v4. Communication with the backend is managed by Inertia.js, avoiding the overhead of client-side routing and state stores (like Redux) by relying on backend state hydration.
2.  **Backend (Laravel 12):** Serves as the primary application engine. It handles business logic, security middleware, validations, and database interactions.
3.  **Database (MySQL/PostgreSQL):** Stores relational data with strict foreign key constraints and indexed columns for fast lookup.

---

## 2. Database Schema Design

### 2.1 Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    users {
        bigint id PK
        string name
        string email UK
        string password
        enum role
        bigint department_id FK
        timestamp created_at
        timestamp updated_at
    }
    departments {
        bigint id PK
        string name
        timestamp created_at
        timestamp updated_at
    }
    attendances {
        bigint id PK
        bigint user_id FK
        datetime check_in
        datetime check_out
        decimal latitude_in
        decimal longitude_in
        decimal latitude_out
        decimal longitude_out
        enum status
        decimal working_hours
        timestamp created_at
        timestamp updated_at
    }
    leaves {
        bigint id PK
        bigint user_id FK
        date start_date
        date end_date
        text reason
        enum status
        timestamp created_at
        timestamp updated_at
    }

    departments ||--o{ users : "contains"
    users ||--o{ attendances : "logs"
    users ||--o{ leaves : "requests"
```

### 2.2 Table Definitions

#### Table: `departments`
| Column | Type | Nullable | Key | Default | Notes / Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | `bigint unsigned` | No | PK | *Auto Increment* | Primary identifier. |
| `name` | `varchar(255)` | No | | | Name of the department. |
| `created_at` | `timestamp` | Yes | | `NULL` | Laravel standard timestamp. |
| `updated_at` | `timestamp` | Yes | | `NULL` | Laravel standard timestamp. |

#### Table: `users`
| Column | Type | Nullable | Key | Default | Notes / Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | `bigint unsigned` | No | PK | *Auto Increment* | Primary identifier. |
| `name` | `varchar(255)` | No | | | Full name of the user. |
| `email` | `varchar(255)` | No | UK | | Must be unique. Used for login. |
| `password` | `varchar(255)` | No | | | Bcrypt hashed password. |
| `role` | `enum('employee', 'admin')` | No | | `'employee'` | User authorization role. |
| `department_id` | `bigint unsigned` | Yes | FK | `NULL` | References `departments.id` (ON DELETE SET NULL). |
| `created_at` | `timestamp` | Yes | | `NULL` | Laravel standard timestamp. |
| `updated_at` | `timestamp` | Yes | | `NULL` | Laravel standard timestamp. |

#### Table: `attendances`
| Column | Type | Nullable | Key | Default | Notes / Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | `bigint unsigned` | No | PK | *Auto Increment* | Primary identifier. |
| `user_id` | `bigint unsigned` | No | FK | | References `users.id` (ON DELETE CASCADE). |
| `check_in` | `datetime` | No | | | Timestamp of daily check-in. |
| `check_out` | `datetime` | Yes | | `NULL` | Timestamp of daily check-out. |
| `latitude_in` | `decimal(10, 8)` | Yes | | `NULL` | GPS Latitude at check-in. |
| `longitude_in` | `decimal(11, 8)` | Yes | | `NULL` | GPS Longitude at check-in. |
| `latitude_out` | `decimal(10, 8)` | Yes | | `NULL` | GPS Latitude at check-out. |
| `longitude_out` | `decimal(11, 8)` | Yes | | `NULL` | GPS Longitude at check-out. |
| `status` | `enum('present', 'absent', 'late')` | No | | `'present'` | Automatically computed day status. |
| `working_hours` | `decimal(5, 2)` | Yes | | `NULL` | Calculated hours difference (check_out - check_in). |
| `created_at` | `timestamp` | Yes | | `NULL` | Laravel standard timestamp. |
| `updated_at` | `timestamp` | Yes | | `NULL` | Laravel standard timestamp. |

#### Table: `leaves`
| Column | Type | Nullable | Key | Default | Notes / Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | `bigint unsigned` | No | PK | *Auto Increment* | Primary identifier. |
| `user_id` | `bigint unsigned` | No | FK | | References `users.id` (ON DELETE CASCADE). |
| `start_date` | `date` | No | | | Beginning of the leave period. |
| `end_date` | `date` | No | | | End of the leave period. |
| `reason` | `text` | No | | | Detailed justification. |
| `status` | `enum('pending', 'approved', 'rejected')` | No | | `'pending'` | Current status of the leave application. |
| `created_at` | `timestamp` | Yes | | `NULL` | Laravel standard timestamp. |
| `updated_at` | `timestamp` | Yes | | `NULL` | Laravel standard timestamp. |

---

## 3. Access Control & Role-Based Permissions (RBAC)

System authorization is enforced using Laravel policies/gates based on the `role` enum in the `users` table.

| Resource / Action | Employee Role | Admin Role | Backend Mechanism |
| :--- | :---: | :---: | :--- |
| **View own Profile & Dashboard** | Yes | Yes | `Auth::user()` Context |
| **Check In / Check Out** | Yes | No | Controller Check-in Policy |
| **View own Attendance History** | Yes | Yes | Query scoped to `auth()->id()` |
| **Submit Leave Request** | Yes | No | Form Request Validation |
| **Approve / Reject Leave** | No | Yes | `AdminMiddleware` & Gates |
| **Manage Employees & Departments** | No | Yes | `AdminMiddleware` & Gates |
| **Generate & Export Reports** | No | Yes | `AdminMiddleware` |

---

## 4. API & Page Routing Directory

The application routes are structured under Web routing and protected by session auth guards. Standard Inertia responses are served for UI rendering.

| HTTP Method | Route Endpoint | Controller Action | Authentication / Middleware | Response Format / View | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **POST** | `/login` | `LoginController@login` | Guest | JSON/Redirect | Authenticate credentials. |
| **POST** | `/logout` | `LoginController@logout` | Auth | Redirect | Terminate session. |
| **POST** | `/attendance/check-in` | `AttendanceController@checkIn` | Auth (`role:employee`) | JSON | Logs check-in and GPS. |
| **POST** | `/attendance/check-out` | `AttendanceController@checkOut` | Auth (`role:employee`) | JSON | Logs check-out, updates working hours. |
| **GET** | `/attendance/history` | `AttendanceController@history` | Auth | Inertia (`Attendance/History`) | Show list of attendance records. |
| **POST** | `/leave/request` | `LeaveController@store` | Auth (`role:employee`) | Redirect | Submits a new leave request. |
| **GET** | `/leave/history` | `LeaveController@index` | Auth | Inertia (`Leave/Index`) | Lists own leave applications. |
| **PUT** | `/leave/approve/{id}` | `LeaveController@approve` | Auth (`role:admin`) | Redirect | Updates leave status to approved/rejected. |
| **GET** | `/employees` | `EmployeeController@index` | Auth (`role:admin`) | Inertia (`Employee/Index`) | Lists all employees in organization. |
| **POST** | `/employees` | `EmployeeController@store` | Auth (`role:admin`) | Redirect | Creates a new employee profile. |
| **PUT** | `/employees/{id}` | `EmployeeController@update` | Auth (`role:admin`) | Redirect | Updates employee details/status. |
| **DELETE** | `/employees/{id}` | `EmployeeController@destroy` | Auth (`role:admin`) | Redirect | Disables or soft-deletes user. |

---

## 5. Sequence Workflows

### 5.1 Check-in / Check-out Sequence

```mermaid
sequenceDiagram
    autonumber
    actor Employee as Employee Client (React)
    participant Server as Laravel Backend
    participant DB as DB (attendances)

    Employee->>Employee: Capture Geo coordinates via browser API
    Employee->>Server: POST /attendance/check-in {latitude, longitude}
    activate Server
    Server->>Server: Verify authentication & role:employee
    Server->>DB: Query today's check-ins for current user
    DB-->>Server: Return record count
    alt Already Checked In
        Server-->>Employee: Response: 422 Unprocessable (Duplicate Check-In)
    else First check-in of the day
        Server->>DB: Insert record (check_in, user_id, lat_in, lng_in)
        DB-->>Server: Success
        Server-->>Employee: Response: 200 OK (Check-In Success)
    end
    deactivate Server
```

### 5.2 Leave Request and Approval Sequence

```mermaid
sequenceDiagram
    autonumber
    actor Employee as Employee Client (React)
    participant Server as Laravel Backend
    participant DB as DB (leaves)
    actor Admin as Admin / HR Client (React)

    Employee->>Server: POST /leave/request {start_date, end_date, reason}
    activate Server
    Server->>Server: Validate date format & business rules
    Server->>DB: Insert new leave record (status = 'pending')
    DB-->>Server: Success
    Server-->>Employee: Redirect / Response 200 (Success)
    deactivate Server

    Note over Server: Async Email Notification sent to Admin/HR

    Admin->>Server: PUT /leave/approve/{id} {status: 'approved'/'rejected'}
    activate Server
    Server->>Server: Validate user has 'admin' role
    Server->>DB: Update leave status & set updated_at
    DB-->>Server: Success
    Server-->>Admin: Redirect / Response 200 (Updated)
    deactivate Server

    Note over Server: Async Email Notification sent to Employee
```

---

## 6. Detailed Technical Logic

### 6.1 Working Hours Calculation
Working hours are computed dynamically on check-out to avoid client-side manipulation. The calculation is done inside the database transactional thread or via service logic:
$$\text{Working Hours} = \frac{\text{Check-out Timestamp} - \text{Check-in Timestamp}}{3600}$$
Rounded to 2 decimal places.

### 6.2 Security Implementations
*   **Authentication Middleware:** Handled by standard Laravel authentication guards (`auth` middleware config).
*   **CSRF Protection:** Managed transparently by Axios and Inertia.js sending the `X-XSRF-TOKEN` cookie header automatically.
*   **Passwords:** Standard Laravel hashing using standard Argon2id or Bcrypt algorithms.
