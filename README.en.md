# PlanSasa — gamified personal planner, offline-first, with your own server

![Kotlin](https://img.shields.io/badge/Kotlin-7F52FF?logo=kotlin&logoColor=white)
![Jetpack Compose](https://img.shields.io/badge/Jetpack_Compose-4285F4?logo=jetpackcompose&logoColor=white)
![Android](https://img.shields.io/badge/Android-34A853?logo=android&logoColor=white)
![Room](https://img.shields.io/badge/Room-3DDC84?logo=android&logoColor=white)
![Ktor](https://img.shields.io/badge/Ktor-087CFA?logo=ktor&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_17-4169E1?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Private code · showcase](https://img.shields.io/badge/Private%20code-showcase-6e7781)

🇧🇷 [Versão em português](README.md)

## Context

PlanSasa is a native Android app I built for my own use: tasks, calendar and Pomodoro in one place, with
rewards that help keep a streak going and a look you can tune however you like. Two annoyances drove the
project: productivity apps that lock your data into someone else's cloud, and apps that stop working without
internet. Here the real database lives on the phone, the app works fully offline, and syncing between devices
is optional, through a server you run yourself with Docker.

## Highlights

- **Tasks with an Eisenhower matrix**: priority quadrants, subtasks, categories, deadlines, reminders and
  recurrence (with Brazilian national holidays computed on the device); drag-and-drop reordering.
- **Calendar** in month, year, timeline and list views, merging tasks with the Android calendar's events.
- **Focus**: Pomodoro or stopwatch, several screen modes, music, a floating window, statistics and a strict
  mode that **pauses when the phone is picked up off the table** (accelerometer).
- **Gamification**: daily missions, check-ins, streaks, a coin wallet and rewards, challenges, goals with
  milestones and a virtual pet that thrives on consistency.
- **Journal and mood tracking**, countdowns and a yearly report with task and focus analytics.
- **Fully customizable theme**: any seed color, nine palette styles, contrast level, AMOLED mode and an exact
  color per role, with text contrast computed automatically.
- **Home-screen widgets** (Jetpack Glance): lists, quadrants, week and month, focus, countdowns, mood, pet and
  photos from the gallery.
- **App lock** with a PIN (stored with salted PBKDF2, never in plain text) or biometrics.
- **Optional, self-hosted sync**: a Ktor API + PostgreSQL in Docker Compose, cursor-based push/pull and
  version-based conflict resolution.

## Architecture

```mermaid
flowchart LR
    subgraph App["Android app (Kotlin + Compose)"]
        direction TB
        UI["feature/*<br/>Compose screens + ViewModels"]
        DOM["domain<br/>pure Kotlin: rules, models,<br/>repository interfaces"]
        DATA["data<br/>Room and DataStore repositories,<br/>Retrofit, sync"]
        ROOM[("Room<br/>source of truth")]
        DS[("DataStore<br/>theme and settings")]
        WM["WorkManager + AlarmManager<br/>sync every 15 min,<br/>exact reminders"]
        GL["Widgets<br/>Jetpack Glance"]
        UI --> DOM
        DATA --> DOM
        DATA --> ROOM
        DATA --> DS
        WM --> DATA
        GL --> DATA
    end

    subgraph Server["Your own server (optional)"]
        direction TB
        PX["HTTPS reverse proxy<br/>or VPN"]
        API["Sync API<br/>Ktor + Netty, JVM 21"]
        PG[("PostgreSQL 17<br/>no published port")]
        PX --> API
        API -- "JDBC + HikariCP" --> PG
    end

    DATA -- "Bearer token<br/>push / pull" --> PX
```

The dependency rule is `feature → domain ← data`: the domain knows nothing about Android, Compose or Retrofit,
and that is where the JVM-tested rules live (recurrence, focus, missions, streaks, goals, statistics).

### Sync: push, pull and conflicts

```mermaid
sequenceDiagram
    participant T as Trigger
    participant E as SyncEngine (app)
    participant R as Room
    participant A as Ktor API
    participant P as PostgreSQL

    T->>E: app opened, local change, every 15 min or button
    E->>A: GET /api/v1/sync/info
    A-->>E: server id and protocol
    opt server id changed (database recreated)
        E->>R: mark everything pending and reset the cursor
    end
    loop each synced table, in pages
        E->>R: pending rows
        E->>A: POST /sync/push with X-Device-Id
        A->>P: lock the profile (advisory lock) and UPSERT unless updated_at is older
        P-->>A: accepted with a new seq, refused
        A-->>E: accepted + conflicts carrying the server version
        E->>R: adopt the server version on conflicts and mark the rest as synced
    end
    loop while there is more
        E->>A: GET /sync/pull?since=cursor
        A->>P: rows with seq above the cursor
        A-->>E: rows from other devices + new cursor
        E->>R: apply and store the cursor
    end
```

### Server data model

```mermaid
erDiagram
    PROFILES ||--o{ SYNC_RECORD : "owns"

    SERVER_INFO {
        uuid id PK "installation identity"
        timestamptz created_at
        boolean singleton "enforces a single row"
    }
    PROFILES {
        uuid id PK
        varchar display_name
        timestamptz created_at
        timestamptz updated_at
    }
    SYNC_RECORD {
        uuid profile_id PK, FK
        varchar entity PK "Room table"
        varchar record_id PK "row key"
        jsonb data "whole row as JSON"
        bigint updated_at "device clock, in ms"
        boolean deleted "soft delete"
        varchar device_id "last device that wrote it"
        bigint seq "pull cursor, unique per profile"
        timestamptz received_at
    }
```

## Stack

| Layer | Technologies |
|---|---|
| App | Kotlin 2.3, Jetpack Compose (Material 3), Navigation Compose, MVVM + Clean Architecture, Koin, Coroutines/Flow |
| Local data | Room (versioned schema with migration tests), DataStore Preferences |
| Android | Jetpack Glance (widgets), WorkManager, AlarmManager, Biometric, Photo Picker, accelerometer, foreground service |
| Visuals | MaterialKolor (HCT palette from one color), Lottie, Coil |
| Network | Retrofit, OkHttp, kotlinx.serialization |
| Backend | Ktor 3 (Netty), JDBC + HikariCP, PostgreSQL 17, JVM 21 |
| Infra | Docker Compose (healthchecks, JRE-only final image, non-root user), `.env` kept out of git |
| Tests | JUnit 4, kotlinx-coroutines-test, Turbine, Koin Test, Room Testing (instrumented) |

## Technical decisions and challenges

- **Truly offline-first.** The phone's Room database is the source of truth and the server is just a meeting
  point and backup. That is why the default mode is "this device only", and the app loses no feature without
  a server.
- **A server that needs no migration for every new column.** Each syncable row travels as JSON into a single
  table (`sync_record`), keyed by table and id. A new column in the app does not change the server schema;
  device-only columns are left out of the payload.
- **A cursor that never skips a row.** Every write gets a new `seq`, and pushes from the same profile are
  serialized with `pg_advisory_xact_lock`, so `seq` order matches commit order. Pull only returns what came
  from other devices, but the cursor still moves past the device's own rows.
- **Simple, predictable conflicts.** The newest `updated_at` wins (a tie accepts the incoming row). The server
  refuses the older version and returns its own, which the app adopts. Deletes are soft deletes so other
  devices learn what disappeared. If the server database is recreated, the app notices through the
  installation id and re-uploads everything.
- **Ktor rather than a backend in another language.** App and API both in Kotlin share enums, names and the
  same `kotlinx.serialization`, with no contracts to keep in sync across two languages. No ORM: explicit SQL
  and a small image.
- **A theme built at runtime.** There is no fixed palette: the Material 3 scheme is derived from a seed color,
  then goes through an AMOLED layer and per-role overrides, and is saved in DataStore so the app opens with
  the right colors even offline. Colors that carry meaning (quadrants, success, streaks) are kept apart and
  keep their meaning under any palette.

## My role

A personal project I built from scratch: **52 commits, all mine**, between October and November 2026.
I built the Android app (architecture, screens, widgets, focus, gamification, theming), the Ktor sync API, the
PostgreSQL schema and the Docker Compose setup. The app has 208 JVM unit tests (domain and sync) and 66
instrumented tests (Room repositories, migrations and the local side of sync).

---

The source code is private. This page is only a showcase of the project.
Contact: [github.com/roger-mont](https://github.com/roger-mont)
