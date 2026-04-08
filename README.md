# AI Workforce Optimizer for Hospitals

A Spring Boot application for intelligent nurse and healthcare staff scheduling, burnout reduction, and operational optimization.

---

## Quick Start

```bash
# Build and run (H2 in-memory DB included — no setup needed)
./mvnw spring-boot:run

# Open Swagger UI
open http://localhost:8080/swagger-ui.html

# Open H2 Console (dev)
open http://localhost:8080/h2-console
# JDBC URL: jdbc:h2:mem:hospitaldb  |  User: sa  |  Password: password
```

### Demo Credentials

| Role    | Email                     | Password      |
|---------|---------------------------|---------------|
| Admin   | admin@hospital.com        | Admin123!     |
| Manager | manager@hospital.com      | Manager123!   |
| Staff   | nurse1@hospital.com       | Nurse123!     |

---

## Architecture Overview

```
src/main/java/com/hospital/optimizer/
├── config/
│   ├── DataInitializer.java       # Seeds sample data on startup
│   ├── OpenApiConfig.java         # Swagger / OpenAPI configuration
│   ├── ScheduledTasks.java        # Cron jobs (fatigue refresh, alerts, payroll scan)
│   └── SecurityConfig.java        # JWT + role-based security
│
├── controller/
│   ├── AuthController.java        # POST /api/auth/login, /refresh
│   ├── ForecastingController.java # GET  /api/forecasting
│   ├── PayrollController.java     # GET  /api/payroll
│   ├── RebalancingController.java # POST /api/rebalance
│   ├── ScheduleController.java    # POST /api/schedule/generate
│   └── StaffController.java       # CRUD /api/staff
│
├── dto/                           # Request / Response DTOs
├── entity/                        # JPA Entities (Staff, Shift, Availability, HistoricalDemand)
├── enums/                         # ShiftType, Department, StaffRole, ShiftStatus, UserRole
├── exception/                     # GlobalExceptionHandler
├── repository/                    # Spring Data JPA repositories
├── security/                      # JwtUtil, JwtAuthFilter, CustomUserDetailsService
│
└── service/
    ├── DemandForecastingService.java  # Module 3: Demand forecasting
    ├── FatigueScoringService.java     # Module 2: Fatigue engine
    ├── PayrollService.java            # Module 6: Payroll & HR
    ├── RebalancingService.java        # Module 4: Auto-rebalancing
    ├── SchedulingService.java         # Module 1: Intelligent scheduling
    └── StaffService.java              # Staff CRUD + availability
```

---

## Core Features

### 1. Intelligent Scheduling (`SchedulingService`)
- **Shift models**: Morning (06–14), Afternoon (14–22), Night (22–06), Long Day (07–19), Long Night (19–07), **12×36** (12h on / 36h off)
- **11-hour rest rule**: Enforced between all consecutive shifts
- **Unsafe sequence detection**: Night shift → Morning shift blocked automatically
- **Weekly hours cap**: Configurable (default 60h max)
- **12×36 enforcement**: 36h mandatory rest between 12h shifts
- Fatigue-sorted assignment (lowest fatigue score first)

**POST** `/api/schedule/generate`
```json
{
  "department": "ICU",
  "startDate": "2026-04-14",
  "endDate": "2026-04-20",
  "shiftTypes": ["MORNING", "AFTERNOON", "NIGHT"],
  "requiredStaffPerShift": { "MORNING": 4, "NIGHT": 3 }
}
```

### 2. Fatigue Scoring Engine (`FatigueScoringService`)
Score 0–100 composed of:

| Component              | Max Points | Logic |
|------------------------|-----------|-------|
| Total hours (7 days)   | 35 pts    | `(hoursWorked / maxHoursPerWeek) × 35` |
| Night shifts           | 25 pts    | `count × 1.5 × 5` |
| Long shifts (≥12h)     | 20 pts    | `count × 1.3 × 4` |
| Consecutive days       | 15 pts    | `(days / maxConsecutiveDays) × 15` |
| Recovery bonus         | −5 pts    | Reduces if ≥48h since last shift |

**Levels**: LOW (0–30) · MODERATE (31–60) · HIGH (61–80) · CRITICAL (81–100)

**GET** `/api/staff/{id}/fatigue` → detailed analysis + recommendation

### 3. Demand Forecasting (`DemandForecastingService`)
Blended model:
- **Historical average** (rolling 90-day)
- **Seasonal multipliers** (Dec +25%, Jan +20%, Jun −10%, etc.)
- **Day-of-week patterns** (Mon +10%, Sat −15%)
- **Emergency surge buffer** based on historical emergency day frequency
- **Minimum safety floors** per department (ICU morning ≥ 3, Emergency night ≥ 3, etc.)

**GET** `/api/forecasting/week/summary?department=ICU&weekStart=2026-04-14`

### 4. Auto-Rebalancing (`RebalancingService`)
1. Report absence → shift marked `ABSENT`
2. Candidate scoring (100-point scale):
   - Deducts for fatigue (× 0.4)
   - +20 for primary department match
   - +10 for role match
   - Deducts for weekly hours utilization
3. Returns top-5 ranked replacements
4. Assign replacement via `POST /api/rebalance/assign-replacement`

### 5. Skill-Based Matching
- Each staff member has a `primaryDepartment` + `qualifiedDepartments` set
- Scheduling and rebalancing filter exclusively within qualified departments
- JPQL query: `SELECT s FROM Staff s WHERE :dept MEMBER OF s.qualifiedDepartments`

### 6. Payroll & HR Integration (`PayrollService`)
- Tracks **regular hours**, **overtime** (>40h/week × 1.5), **night differential** (×1.25)
- Detects **illegal overtime** (>60h/week) and flags violations
- Generates per-staff and full-team payroll summaries
- Weekly violation scan runs automatically every Monday at 07:00

---

## REST API Reference

### Authentication
| Method | Endpoint              | Description              |
|--------|-----------------------|--------------------------|
| POST   | `/api/auth/login`     | Login, receive JWT token |
| POST   | `/api/auth/refresh`   | Refresh expired token    |

### Staff
| Method | Endpoint                           | Role            |
|--------|------------------------------------|-----------------|
| GET    | `/api/staff`                       | All             |
| GET    | `/api/staff/{id}`                  | All             |
| POST   | `/api/staff`                       | ADMIN, MANAGER  |
| PUT    | `/api/staff/{id}`                  | ADMIN, MANAGER  |
| DELETE | `/api/staff/{id}`                  | ADMIN           |
| GET    | `/api/staff/{id}/fatigue`          | All             |
| POST   | `/api/staff/{id}/fatigue/refresh`  | ADMIN, MANAGER  |
| POST   | `/api/staff/fatigue/refresh-all`   | ADMIN, MANAGER  |

### Scheduling
| Method | Endpoint                        | Description               |
|--------|---------------------------------|---------------------------|
| POST   | `/api/schedule/generate`        | Auto-generate schedule    |
| GET    | `/api/schedule?start=&end=`     | List shifts               |
| GET    | `/api/schedule/today`           | Today's shifts            |
| POST   | `/api/schedule/validate`        | Validate before assigning |
| PATCH  | `/api/schedule/{id}/status`     | Update shift status       |
| DELETE | `/api/schedule/{id}`            | Cancel shift              |

### Forecasting
| Method | Endpoint                                   |
|--------|--------------------------------------------|
| GET    | `/api/forecasting?department=&date=&shiftType=` |
| GET    | `/api/forecasting/week?department=&weekStart=`  |
| GET    | `/api/forecasting/week/summary?department=&weekStart=` |

### Rebalancing
| Method | Endpoint                                   |
|--------|--------------------------------------------|
| POST   | `/api/rebalance/absence/{shiftId}`         |
| POST   | `/api/rebalance/assign-replacement`        |
| GET    | `/api/rebalance/alerts?date=`              |

### Payroll
| Method | Endpoint                                   |
|--------|--------------------------------------------|
| GET    | `/api/payroll/summary/{staffId}?periodStart=&periodEnd=` |
| GET    | `/api/payroll/report?periodStart=&periodEnd=` |
| GET    | `/api/payroll/violations?weekStart=`       |
| GET    | `/api/payroll/utilization?start=&end=`     |

---

## Role-Based Access Control

| Endpoint Category | ADMIN | MANAGER | STAFF |
|-------------------|:-----:|:-------:|:-----:|
| Auth              | ✓     | ✓       | ✓     |
| View staff        | ✓     | ✓       | ✓     |
| Create/edit staff | ✓     | ✓       | ✗     |
| Delete staff      | ✓     | ✗       | ✗     |
| Generate schedule | ✓     | ✓       | ✗     |
| Forecasting       | ✓     | ✓       | ✗     |
| Payroll reports   | ✓     | ✓       | ✗     |
| Rebalancing       | ✓     | ✓       | ✗     |
| View own shifts   | ✓     | ✓       | ✓     |
| Own fatigue score | ✓     | ✓       | ✓     |

---

## Configuration

All settings in `application.yml`:

```yaml
hospital:
  scheduling:
    min-rest-hours-between-shifts: 11   # Legal minimum
    max-hours-per-week: 60              # Legal maximum
    max-consecutive-days: 6            # Before mandatory day off
    fatigue-score-threshold: 75.0       # Above = not schedulable
    night-shift-weight: 1.5             # Fatigue multiplier
    long-shift-weight: 1.3              # Fatigue multiplier
  forecasting:
    lookback-days: 90                   # Historical data window
    seasonal-weight: 0.3
    trend-weight: 0.4
    emergency-surge-multiplier: 1.25
```

---

## Scheduled Background Jobs

| Job                   | Schedule          | Description                       |
|-----------------------|-------------------|-----------------------------------|
| Fatigue score refresh | Every 6 hours     | Recalculates all staff scores     |
| Understaffing check   | Daily at 05:00    | Logs/alerts on short shifts       |
| Overtime scan         | Mondays at 07:00  | Detects illegal overtime for HR   |

---

## Running Tests

```bash
./mvnw test
```

Test coverage includes:
- `FatigueScoringServiceTest` — 6 tests (score bounds, level thresholds, night/long shifts)
- `SchedulingServiceTest` — 5 tests (eligibility, unsafe sequences, weekly hours, validation)
- `PayrollServiceTest` — 5 tests (regular/OT calculation, night diff, illegal OT detection)
- `DemandForecastingServiceTest` — 5 tests (defaults, ICU minimum, emergency surge, weekly size)

---

## Production Setup (PostgreSQL)

1. Uncomment PostgreSQL block in `application.yml`
2. Set environment variables:
   ```bash
   export DB_USERNAME=postgres
   export DB_PASSWORD=yourpassword
   export JWT_SECRET=your-256-bit-secret-key-here
   ```
3. Change `ddl-auto: create-drop` → `update` (or use Flyway migrations)
4. Set `h2.console.enabled: false`

---

## Sample Data

On startup, the system seeds:
- **17 staff members** across ICU, Emergency, Surgery, Pediatrics, General Ward, Cardiology, Oncology
- **3 roles**: Admin, Manager, Staff (Registered Nurse / Senior Nurse)
- **90 days** of historical demand records (~1,350 records) for forecasting
- **3 weeks of shifts** (14 past + 7 future) with realistic patterns
- **Availability records** for all staff across all days of the week

---

## Technology Stack

| Layer       | Technology                              |
|-------------|----------------------------------------|
| Framework   | Spring Boot 3.2 + Spring Security 6    |
| Database    | H2 (dev) / PostgreSQL (production)     |
| ORM         | Spring Data JPA + Hibernate            |
| Auth        | JWT (jjwt 0.11.5)                      |
| API Docs    | SpringDoc OpenAPI 3 / Swagger UI       |
| Testing     | JUnit 5 + Mockito + AssertJ            |
| Build       | Maven                                  |
| Java        | 17+                                    |
