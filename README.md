# RefSync — Handball Referee Assignment System

> **Version:** 2.0.0-draft  
> **Status:** In Development  
> **Last Updated:** May 2026

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Problem Statement](#2-problem-statement)
3. [System Architecture](#3-system-architecture)
4. [User Roles & Permissions](#4-user-roles--permissions)
5. [Data Models](#5-data-models)
6. [Availability Management](#6-availability-management)
7. [Match Import & Validation](#7-match-import--validation)
8. [Assignment Engine](#8-assignment-engine)
9. [Role-Based Assignment](#9-role-based-assignment)
10. [Assignment Workflow](#10-assignment-workflow)
11. [Rejection & Fallback Chain](#11-rejection--fallback-chain)
12. [Conflict of Interest Rules](#12-conflict-of-interest-rules)
13. [Notification System](#13-notification-system)
14. [Rating System](#14-rating-system)
15. [Audit & Change Log](#15-audit--change-log)
16. [API Design Overview](#16-api-design-overview)
17. [Technology Stack](#17-technology-stack)
18. [Non-Functional Requirements](#18-non-functional-requirements)
19. [Security Considerations](#19-security-considerations)
20. [Future Enhancements](#20-future-enhancements)

---

## 1. Project Overview

**RefSync** is a mobile-first application designed for handball referee committees to automate referee assignment, eliminate scheduling conflicts, and streamline communication between head referees and the referee pool.

Instead of managing assignments through spreadsheets, Google Forms, and informal messaging groups, RefSync provides a centralized platform that:

- Automatically selects the most suitable referees per match based on availability, proximity, role suitability, ratings, and workload.
- Assigns referees to specific **roles** (1st court referee, 2nd court referee, scorer, timekeeper) rather than generically selecting four people.
- Enforces conflict of interest rules to protect match integrity.
- Maintains a full audit trail of every manual edit and assignment change.
- Handles rejection gracefully with an automatic multi-step fallback chain.

---

## 2. Problem Statement

Current referee assignment is managed using Google Forms, Excel sheets, and WhatsApp groups. This process consistently results in:

| Pain Point | Impact |
|---|---|
| Referees missing their assignments | Match disruptions, last-minute scrambles |
| Long travel distances for some referees | Lateness, fatigue, increased costs |
| Uneven match distribution | Experienced referees overloaded; juniors idle |
| Manual scheduling consuming hours per week | Head referee bottleneck |
| No structured replacement process | Single rejection stalls the whole assignment |
| No conflict of interest enforcement | Integrity and fairness risks |
| No audit trail for manual changes | No accountability for who changed what |
| WhatsApp-only communication | No confirmation tracking, no history |

RefSync addresses every one of these systematically.

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mobile App (Flutter)                     │
│          iOS / Android                     Head Referee UI      │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS / REST
┌────────────────────────▼────────────────────────────────────────┐
│                    API Gateway (Spring Boot)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  Auth Service│  │Match Service │  │  Assignment Engine   │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  Referee Svc │  │Notification  │  │  Availability Service│   │
│  └──────────────┘  │  Service     │  └──────────────────────┘   │
│                    └──────────────┘                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                    PostgreSQL Database                           │
└─────────────────────────────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
  Firebase FCM     WhatsApp API    File Storage
  (Push Notif.)   (Messaging)     (Excel Uploads)
```

### Service Responsibilities

**Auth Service** — JWT-based authentication, role management (HEAD_REFEREE, REFEREE).

**Match Service** — handles Excel import, match CRUD, import validation, duplicate detection.

**Assignment Engine** — core scoring algorithm, role-slot filling, fallback chain management.

**Referee Service** — profile management, ratings, club association, location data.

**Availability Service** — weekly availability submission, deadline enforcement, carry-forward logic.

**Notification Service** — orchestrates push + WhatsApp delivery, tracks delivery and response status.

---

## 4. User Roles & Permissions

### Head Referee

| Capability | Details |
|---|---|
| Import match schedule | Upload Excel file; system validates and previews before committing |
| Review assignments | See auto-generated role-slot assignments per match |
| Edit assignments manually | Replace any referee in any slot; must provide a reason (audit log) |
| Manage referee ratings | Update court and table ratings per referee |
| View all availability | Full committee availability dashboard with weekly heatmap |
| Configure scoring weights | Adjust α (rating), β (distance), γ (workload) for the assignment formula |
| Configure conflict rules | Define which club associations trigger exclusions |
| View audit trail | Full history of all assignment changes with actor, timestamp, reason |
| Approve and publish | Finalize assignments and trigger notifications |
| Manage fallback chain | Review and override automatic replacement decisions |

### Referee

| Capability | Details |
|---|---|
| Manage profile | Name, DOB, contact info, home/work/college addresses, club affiliation |
| Submit weekly availability | Per-day time ranges, unavailability, temporary leave |
| Receive assignment notifications | Push notification + WhatsApp with match details and role |
| Accept or reject assignments | Respond before deadline; rejection triggers automatic replacement |
| View upcoming matches | Personal schedule with role, venue, time |
| View assignment history | Past matches and roles |

---

## 5. Data Models

### Referee

```
referee_id         UUID (PK)
full_name          VARCHAR(100)
date_of_birth      DATE
phone_number       VARCHAR(20)
email              VARCHAR(100) UNIQUE
home_address       TEXT
home_lat           DECIMAL(9,6)
home_lng           DECIMAL(9,6)
work_address       TEXT
work_lat           DECIMAL(9,6)
work_lng           DECIMAL(9,6)
college_address    TEXT
college_lat        DECIMAL(9,6)
college_lng        DECIMAL(9,6)
club_id            UUID (FK → clubs)         -- for conflict-of-interest enforcement
court_rating       DECIMAL(4,2)              -- 0.00 – 10.00
table_rating       DECIMAL(4,2)              -- 0.00 – 10.00
is_active          BOOLEAN DEFAULT TRUE
created_at         TIMESTAMP
updated_at         TIMESTAMP
```

### Match

```
match_id           UUID (PK)
team_a             VARCHAR(100)
team_b             VARCHAR(100)
match_date         DATE
match_time         TIME
venue_name         VARCHAR(150)
venue_address      TEXT
venue_lat          DECIMAL(9,6)
venue_lng          DECIMAL(9,6)
match_level        ENUM(LEAGUE_A, LEAGUE_B, CUP, FRIENDLY)  -- influences rating priority
import_batch_id    UUID (FK → import_batches)
status             ENUM(PENDING, ASSIGNED, PUBLISHED, COMPLETED, CANCELLED)
created_at         TIMESTAMP
```

### Assignment

```
assignment_id      UUID (PK)
match_id           UUID (FK → matches)
referee_id         UUID (FK → referees)
role               ENUM(COURT_1, COURT_2, SCORER, TIMEKEEPER)
status             ENUM(PENDING_RESPONSE, ACCEPTED, REJECTED, REPLACED, AUTO_ASSIGNED)
assigned_by        ENUM(SYSTEM, HEAD_REFEREE)
notified_at        TIMESTAMP
response_deadline  TIMESTAMP
responded_at       TIMESTAMP
created_at         TIMESTAMP
```

### Assignment Audit Log

```
log_id             UUID (PK)
match_id           UUID (FK → matches)
changed_by         UUID (FK → referees)      -- the head referee who made the change
previous_referee   UUID (FK → referees)
new_referee        UUID (FK → referees)
role               ENUM(COURT_1, COURT_2, SCORER, TIMEKEEPER)
reason             TEXT                       -- mandatory free-text reason
changed_at         TIMESTAMP
change_type        ENUM(MANUAL_REPLACEMENT, REJECTION_REPLACEMENT, CANCELLATION)
```

### Availability

```
availability_id    UUID (PK)
referee_id         UUID (FK → referees)
week_start_date    DATE                       -- Monday of the target week
day_of_week        ENUM(MON, TUE, WED, THU, FRI, SAT, SUN)
from_time          TIME
to_time            TIME
status             ENUM(AVAILABLE, UNAVAILABLE, LEAVE)
submitted_at       TIMESTAMP
is_late_submission BOOLEAN DEFAULT FALSE
```

### Club

```
club_id            UUID (PK)
club_name          VARCHAR(100)
federation_code    VARCHAR(20)
```

### Referee Feedback

```
feedback_id        UUID (PK)
match_id           UUID (FK → matches)
referee_id         UUID (FK → referees)
role               ENUM(COURT_1, COURT_2, SCORER, TIMEKEEPER)
rating_score       SMALLINT (1–5)
notes              TEXT
submitted_by       UUID (FK → referees)      -- head referee
submitted_at       TIMESTAMP
```

---

## 6. Availability Management

### Weekly Submission Workflow

1. The system opens the availability window every week (configurable day, e.g. Monday 00:00).
2. A push notification reminds referees to submit by the deadline (configurable, e.g. Wednesday 23:59).
3. Referees submit per-day time slots indicating availability, unavailability, or leave.
4. At the deadline, the system locks submissions.
5. Referees who did not submit are marked **unavailable by default** for that entire week.
6. The head referee can grant late submission permission to specific referees up to the point assignments are published.

### Carry-Forward Option

If a referee has not submitted and the deadline passes, the system can optionally carry forward the **previous week's availability** rather than marking them fully unavailable. This is a committee-level configuration setting. If carry-forward is enabled, the referee is notified that their prior availability was used.

### Availability Rules

- A referee cannot be assigned outside their submitted available time slots.
- A referee cannot be double-booked: if already assigned to a match at time T, they are unavailable for any other match overlapping that window.
- Availability history is stored indefinitely for workload analytics.
- The head referee can view the full committee availability in a weekly calendar heatmap view.

### Availability Example

| Day | From | To | Status |
|---|---|---|---|
| Thursday | 5:00 PM | 11:00 PM | Available |
| Friday | 3:00 PM | 11:00 PM | Available |
| Saturday | 8:00 AM | 6:00 PM | Available |
| Sunday | — | — | Unavailable |
| Next week Mon–Sun | — | — | Leave (Ramadan travel) |

---

## 7. Match Import & Validation

### Input Format

The head referee uploads an Excel (.xlsx) file. The expected columns are:

| Column | Type | Required | Notes |
|---|---|---|---|
| Team A | String | Yes | Must not be empty |
| Team B | String | Yes | Must not be empty |
| Match Date | Date | Yes | Format: DD/MM/YYYY |
| Match Time | Time | Yes | Format: HH:MM (24h) |
| Venue Name | String | Yes | Used for geocoding if no coordinates |
| Venue Address | String | Recommended | Improves distance accuracy |
| Match Level | String | No | Defaults to LEAGUE_B if absent |

### Validation Pipeline

Before any matches are committed to the database, the import runs through:

**Step 1 — Schema validation**
- All required columns present and non-empty.
- Date and time fields parse correctly.
- No blank rows in the middle of data.

**Step 2 — Duplicate detection**
- A match is flagged as a duplicate if the same (Team A, Team B, Date, Time) already exists in the database for the current season.
- Duplicates are shown in the preview with a warning; the head referee can choose to skip or overwrite.

**Step 3 — Venue conflict detection**
- If two matches share the same venue with overlapping time windows (match duration defaults to 90 minutes), both are flagged.
- The system does not block import but highlights conflicts clearly in the preview.

**Step 4 — Geocoding**
- Venue addresses are geocoded via Google Maps Geocoding API on import.
- If geocoding fails for a venue, the match is still imported but distance-based assignment is disabled for that match until coordinates are manually entered.

**Step 5 — Preview confirmation**
- The head referee sees a preview table showing all parsed matches, validation warnings, and conflicts.
- They must explicitly confirm before the import is committed.

### Import Error Handling

| Error Type | Behavior |
|---|---|
| Invalid date format | Row highlighted in preview; import blocked until fixed |
| Empty required field | Row highlighted; import blocked |
| Duplicate match | Warning shown; head referee decides |
| Venue geocoding failure | Match imported; distance assignment flagged as limited |
| Venue time conflict | Warning shown; head referee decides |

---

## 8. Assignment Engine

### Scoring Formula

For each eligible referee `r` being evaluated for a match role, the system computes a composite score:

```
score(r) = α × normalized_rating(r, role)
         + β × (1 / (min_distance(r, venue) + 1))
         + γ × workload_penalty(r)
```

Where:
- `α`, `β`, `γ` are configurable weights set by the head referee (default: 0.4, 0.35, 0.25)
- `α + β + γ = 1.0` (enforced by the UI)
- `normalized_rating` uses the role-appropriate rating (court or table), normalized to [0, 1]
- `min_distance` is the minimum of (home → venue, work → venue, college → venue) in km
- `workload_penalty` penalizes referees who already have many assignments this week/month

### Workload Penalty Function

```
workload_penalty(r) = 1 - (weekly_assignments / max_weekly) × 0.5
                        - (monthly_assignments / max_monthly) × 0.5
```

Where `max_weekly` and `max_monthly` are configurable per committee. The penalty never makes a referee ineligible — it only reduces their score.

### Step-by-Step Assignment Process

**Step 1 — Availability filter**

Remove any referee who:
- Has not submitted availability for the week (or carry-forward is disabled).
- Is unavailable on the match date.
- Is unavailable during the match time window (match time ± 30 min travel buffer).
- Is already assigned to another match overlapping this time window.

**Step 2 — Conflict of interest filter**

Remove any referee who:
- Is affiliated with Team A or Team B's club (see Section 12).
- Matches any other configured exclusion rule for this match.

**Step 3 — Role eligibility filter**

For court referee slots (COURT_1, COURT_2): only referees with `court_rating ≥ min_court_rating` for this match level are eligible.

For table slots (SCORER, TIMEKEEPER): only referees with `table_rating ≥ min_table_rating` are eligible.

Minimum ratings per level are configurable by the head referee.

**Step 4 — Scoring and ranking**

Compute `score(r)` for each eligible referee. Rank descending.

**Step 5 — Role-slot filling**

Fill slots in priority order: COURT_1 → COURT_2 → SCORER → TIMEKEEPER.

For each slot, select the highest-ranked eligible referee not already selected for another slot in this match.

**Step 6 — Output**

Four referee-role assignments are produced. If any slot cannot be filled (insufficient eligible referees), the match is flagged as **partially unassigned** and the head referee is alerted.

---

## 9. Role-Based Assignment

Each match requires exactly four referees in specific roles:

| Role | Type | Responsibilities | Rating Used |
|---|---|---|---|
| COURT_1 | Court | Lead on-court officiating | Court Rating |
| COURT_2 | Court | Secondary on-court officiating | Court Rating |
| SCORER | Table | Score sheet, fouls, timeouts | Table Rating |
| TIMEKEEPER | Table | Game clock, shot clock | Table Rating |

### Why Role Separation Matters

Without role separation, the engine might assign a highly-rated court referee to a table position they are not suited for, or vice versa. Separate court and table ratings already exist in the data model — this change ensures those ratings are used for the correct slots.

### Role Eligibility Matrix

| Match Level | Min COURT_1 Rating | Min COURT_2 Rating | Min SCORER Rating | Min TIMEKEEPER Rating |
|---|---|---|---|---|
| LEAGUE_A | 7.0 | 6.0 | 6.0 | 5.0 |
| LEAGUE_B | 5.0 | 4.5 | 4.5 | 4.0 |
| CUP | 8.0 | 7.0 | 7.0 | 6.0 |
| FRIENDLY | 3.0 | 3.0 | 3.0 | 3.0 |

These thresholds are configurable by the head referee.

---

## 10. Assignment Workflow

```
Match Schedule Imported & Validated
          ↓
  Automatic Role-Based Assignment
  (Engine runs per match, per role slot)
          ↓
  Partial assignment flag?
  ├── Yes → Head referee alerted immediately
  └── No  → Proceed
          ↓
  Head Referee Review Dashboard
  ├── Approve as-is
  ├── Replace a referee manually (reason required)
  └── Reassign a role
          ↓
  Assignment Published
          ↓
  Notifications Sent (Push + WhatsApp)
  (Each referee sees their specific role)
          ↓
  Response Window (configurable deadline)
          ↓
  Accept → Officially assigned
  Reject → Fallback chain triggered (Section 11)
          ↓
  All slots confirmed → Match Ready
```

---

## 11. Rejection & Fallback Chain

When a referee rejects an assignment, a single replacement search is not sufficient. The following chain ensures continuity without requiring head referee intervention for every rejection.

### Chain Steps

**Step 1 — Immediate auto-replacement**

The engine re-runs the scoring algorithm for the rejected role slot, excluding already-confirmed referees for this match. The next highest-scoring eligible referee is selected and notified immediately.

**Step 2 — Second rejection**

If the replacement also rejects, the engine runs again. This repeats up to `max_auto_replacements` (default: 3, configurable).

**Step 3 — Exhaustion alert**

If the chain is exhausted without finding a confirmed referee for a slot, the head referee receives an **urgent alert** flagging the specific role that is unfilled. The match moves to **REQUIRES_MANUAL_INTERVENTION** status.

**Step 4 — Emergency pool**

Optionally, referees can opt into an "emergency pool" — they agree to be notified for last-minute slots even outside their submitted availability. Emergency pool referees are only contacted after the standard chain is exhausted.

### Chain Timeline

```
Rejection received
    ↓
Immediate: replacement candidate selected and notified
    ↓ (response deadline = original deadline or T+2h, whichever is later)
Rejection #2 → repeat
    ↓
Rejection #3 (chain exhausted)
    ↓
Urgent alert to head referee + emergency pool notification (if enabled)
```

### Head Referee Visibility

Throughout the chain, the head referee can see in real time:
- Which slot is pending.
- Which candidates have been tried and rejected.
- The current candidate and their response deadline.
- Option to intervene manually at any point.

---

## 12. Conflict of Interest Rules

To protect match integrity, the system enforces configurable exclusion rules.

### Built-in Rules

**Rule 1 — Club affiliation**
A referee affiliated with a club (via `referee.club_id`) cannot be assigned to any match where that club's team is playing (Team A or Team B).

**Rule 2 — Same-club pair prevention**
Two referees from the same club cannot both be assigned to the same match. If the engine would select them together, the lower-ranked one is skipped and the next eligible referee is used instead.

### Configurable Rules (Head Referee)

The head referee can define additional rules at the committee level:

| Rule Type | Example |
|---|---|
| Specific referee excluded from specific team | "Referee X cannot officiate Zamalek matches" |
| Specific referee excluded from specific venue | "Referee Y cannot travel to Alexandria venues" |
| Referee pair exclusion | "Referees A and B cannot be on the same panel" |
| Temporary suspension | "Referee Z is suspended for 2 weeks" |

All configured rules are visible in a dedicated "Exclusion Rules" management screen.

### Rule Audit

Every rule application is logged: which rule fired, for which match, and which referee was excluded. This provides a traceable record of integrity enforcement.

---

## 13. Notification System

### Channels

**In-App Push Notification** (Firebase Cloud Messaging)
- Delivered regardless of WhatsApp availability.
- Primary channel for all system events.
- Supports action buttons: Accept / Reject directly from the notification.

**WhatsApp** (WhatsApp Business API)
- Used for assignment notifications and urgent alerts.
- Requires pre-approved message templates.
- Rate limits and per-message costs apply — used selectively, not for every event.
- Phased rollout: push notifications launch first; WhatsApp added in v1.1 after core flow is stable.

### Notification Events

| Event | Channels | Recipient |
|---|---|---|
| Assignment published | Push + WhatsApp | Assigned referee |
| Rejection received | Push | Head referee |
| Replacement assigned | Push + WhatsApp | Replacement referee |
| Response deadline approaching (2h warning) | Push | Referee with pending response |
| Chain exhausted / urgent slot | Push + WhatsApp | Head referee |
| Availability deadline approaching | Push | All referees who haven't submitted |
| Match cancelled | Push + WhatsApp | All assigned referees |

### Assignment Notification Content

Each notification contains:

- Match: Team A vs Team B
- Date, time, venue
- **Referee's specific role** (COURT_1, COURT_2, SCORER, or TIMEKEEPER)
- Response deadline
- Accept / Reject action

### Response Tracking

Every notification delivery and referee response is recorded:

```
notification_id     UUID (PK)
assignment_id       UUID (FK)
channel             ENUM(PUSH, WHATSAPP)
sent_at             TIMESTAMP
delivered_at        TIMESTAMP (null if unconfirmed)
opened_at           TIMESTAMP (null if unopened)
action_taken        ENUM(ACCEPTED, REJECTED, EXPIRED, PENDING)
action_taken_at     TIMESTAMP
```

---

## 14. Rating System

Ratings are managed exclusively by the head referee.

### Rating Categories

**Court Referee Rating (0.00 – 10.00)**
Evaluates on-court officiating quality: rule application, positioning, communication, decision accuracy.

**Table Referee Rating (0.00 – 10.00)**
Evaluates table operations: score accuracy, foul recording, timeout management, clock accuracy.

### Post-Match Feedback

After each match, the head referee can submit a 1–5 star rating per referee per role. These ratings feed into a rolling average that influences the court/table rating scores over time.

```
updated_rating = (current_rating × history_weight) + (new_feedback_score × 2 × (1 - history_weight))
```

Where `history_weight` defaults to 0.8 (80% historical, 20% new feedback per match). This is configurable.

Benefits of data-driven ratings:
- Removes subjectivity from manual rating updates.
- Creates a feedback loop that rewards consistent performance.
- Produces a full performance history per referee over seasons.

### Rating Visibility

- Referees can see their own ratings.
- Referees cannot see each other's ratings.
- The head referee sees all ratings and the full feedback history.

---

## 15. Audit & Change Log

Every manual change made by the head referee to the auto-generated assignments is recorded in the audit log.

### What Is Logged

- **Actor**: which head referee made the change (supports multiple head referees).
- **Timestamp**: exact time of the change.
- **Match**: which match was affected.
- **Role slot**: which role was changed.
- **Previous referee**: who was removed.
- **New referee**: who was added.
- **Change type**: MANUAL_REPLACEMENT, REJECTION_REPLACEMENT, or CANCELLATION.
- **Reason**: free-text reason (mandatory for MANUAL_REPLACEMENT).

### Why This Matters

Without an audit trail, patterns that indicate favoritism, bias, or error are invisible. Over time, the log enables:
- Detecting if a specific referee is frequently manually replaced (signal for training or conflict).
- Verifying fair distribution across head referees.
- Resolving disputes about who was assigned to what.

### Access

- Head referees can view the full audit log.
- Referees can view changes that involved them personally.
- The log is read-only — it cannot be edited or deleted.

---

## 16. API Design Overview

All endpoints require JWT authentication. Role-based access is enforced at the controller level.

### Core Endpoints

**Auth**
```
POST   /api/auth/login
POST   /api/auth/refresh
POST   /api/auth/logout
```

**Referees**
```
GET    /api/referees                    (HEAD_REFEREE)
GET    /api/referees/{id}
PUT    /api/referees/{id}/profile
PUT    /api/referees/{id}/ratings       (HEAD_REFEREE only)
GET    /api/referees/{id}/assignments
GET    /api/referees/{id}/feedback      (HEAD_REFEREE only)
```

**Availability**
```
POST   /api/availability/submit
GET    /api/availability/week/{weekStart}
GET    /api/availability/week/{weekStart}/summary  (HEAD_REFEREE)
POST   /api/availability/late-request             (HEAD_REFEREE approves)
```

**Matches**
```
POST   /api/matches/import              (HEAD_REFEREE) — multipart/form-data
POST   /api/matches/import/preview      (HEAD_REFEREE) — validation only, no commit
GET    /api/matches
GET    /api/matches/{id}
PUT    /api/matches/{id}/cancel         (HEAD_REFEREE)
```

**Assignments**
```
GET    /api/assignments/match/{matchId}
POST   /api/assignments/match/{matchId}/generate   (HEAD_REFEREE)
PUT    /api/assignments/{id}/replace               (HEAD_REFEREE) — requires reason
POST   /api/assignments/{id}/publish               (HEAD_REFEREE)
POST   /api/assignments/{id}/respond               (REFEREE) — accept/reject
GET    /api/assignments/audit-log                  (HEAD_REFEREE)
```

**Notifications**
```
GET    /api/notifications
PUT    /api/notifications/{id}/read
```

**Config**
```
GET    /api/config/scoring-weights      (HEAD_REFEREE)
PUT    /api/config/scoring-weights      (HEAD_REFEREE)
GET    /api/config/conflict-rules       (HEAD_REFEREE)
POST   /api/config/conflict-rules       (HEAD_REFEREE)
DELETE /api/config/conflict-rules/{id}  (HEAD_REFEREE)
```

---

## 17. Technology Stack

### Mobile Application

| Component | Technology |
|---|---|
| Framework | Flutter 3.x |
| Platforms | Android (API 24+), iOS (15+) |
| State management | Riverpod |
| Local storage | Hive (for offline availability drafts) |
| HTTP client | Dio |

### Backend

| Component | Technology |
|---|---|
| Framework | Spring Boot 3.x |
| Language | Java 21 |
| Authentication | Spring Security + JWT (RS256) |
| ORM | Spring Data JPA + Hibernate |
| Validation | Jakarta Bean Validation |
| Excel parsing | Apache POI |

### Database

| Component | Technology |
|---|---|
| Primary DB | PostgreSQL 15 |
| Migrations | Flyway |
| Connection pool | HikariCP |

### Infrastructure & Integrations

| Component | Technology |
|---|---|
| Push notifications | Firebase Cloud Messaging (FCM) |
| WhatsApp | WhatsApp Business API (Cloud API v18+) |
| Geocoding | Google Maps Geocoding API |
| Distance calculation | Haversine formula (server-side) |
| File storage | Local filesystem (MVP); S3-compatible object storage (production) |
| Containerization | Docker + Docker Compose |

### Development & Tooling

| Component | Technology |
|---|---|
| API documentation | SpringDoc OpenAPI 3 (Swagger UI) |
| Testing | JUnit 5, Mockito, Testcontainers |
| CI/CD | GitHub Actions |
| Code quality | SonarQube, Checkstyle |

---

## 18. Non-Functional Requirements

### Performance

- Assignment engine must complete for a single match within **500ms** under normal load.
- Bulk assignment (full weekly schedule, up to 30 matches) must complete within **10 seconds**.
- API response times under normal load: p95 < 300ms for read operations, p95 < 800ms for write operations.

### Availability

- System uptime target: **99.5%** excluding scheduled maintenance.
- Scheduled maintenance windows communicated via in-app notice 24 hours in advance.

### Scalability

- Initial design supports up to **200 referees** and **100 matches per season**.
- Architecture supports horizontal scaling of the backend service layer via stateless Spring Boot instances.

### Reliability

- All assignment state changes are atomic (database transactions).
- Notification delivery failures are retried up to 3 times with exponential backoff.
- Excel import uses a preview-then-commit pattern to prevent partial imports.

### Data Retention

- Match and assignment data retained indefinitely (historical record).
- Notification logs retained for 2 years.
- Audit logs retained indefinitely and are immutable.

---

## 19. Security Considerations

### Authentication & Authorization

- All API endpoints require a valid JWT (RS256-signed).
- Access tokens expire after 15 minutes; refresh tokens after 7 days.
- Role-based access control enforced at the method level via Spring Security annotations.
- Referees can only access their own profile and assignment data; they cannot access other referees' data.

### Data Privacy

- Phone numbers and email addresses are only visible to the head referee and the referee themselves.
- Home, work, and college addresses are stored encrypted at rest (AES-256).
- Location data (lat/lng) is used only for distance calculation and is never exposed in API responses to other referees.

### Input Validation

- All Excel import data is validated and sanitized server-side before processing.
- File upload accepts `.xlsx` only; MIME type is verified, not just the extension.
- Maximum upload file size: 5MB.

### WhatsApp Integration

- WhatsApp Business API credentials are stored in environment variables, never in source code.
- Only pre-approved message templates are used; no dynamic message construction that could leak data.

### Rate Limiting

- Login endpoint: 5 attempts per IP per 15 minutes.
- Assignment response endpoint: 3 requests per referee per assignment (prevents rapid accept/reject cycling).

---

## 20. Future Enhancements

These are deferred to future versions but are architecturally accounted for in the current design.

| Enhancement | Description | Priority |
|---|---|---|
| GPS attendance verification | Confirm referee arrival at the venue at match time via device location | High |
| Performance analytics dashboard | Season-wide charts: assignments per referee, rating trends, rejection rates, distance averages | High |
| Travel optimization | Minimize total committee travel distance per week as a constraint in the assignment engine | Medium |
| AI assignment recommendations | ML model trained on past manual edits to learn head referee preferences beyond the formula | Medium |
| Multi-season data | Season boundary management; carry-forward ratings and profiles across seasons | Medium |
| Federation-wide management | Expand from a single committee to managing multiple committees under a federation umbrella | Low |
| Referee self-service rating view | Allow referees to see their own performance trend over time (not other referees') | Low |
| Offline mode | Allow referees to view their upcoming schedule and submit availability while offline, syncing when reconnected | Low |
| Head referee mobile app | Currently head referee actions require the web dashboard; a full mobile experience for the head referee role | Medium |

---

## Appendix A — Assignment Scoring Example

**Match:** Al Ahly vs Zamalek, Friday 7:00 PM, Cairo Stadium
**Slot:** COURT_1 (minimum court rating: 7.0, match level: LEAGUE_A)
**Scoring weights:** α=0.4, β=0.35, γ=0.25

| Referee | Court Rating | Min Distance (km) | Weekly Assignments | Score |
|---|---|---|---|---|
| Ahmed M. | 8.5 | 4.2 | 1 | **0.4×0.85 + 0.35×(1/5.2) + 0.25×0.95 = 0.340 + 0.067 + 0.238 = 0.645** |
| Khaled R. | 9.0 | 18.7 | 0 | 0.4×0.90 + 0.35×(1/19.7) + 0.25×1.00 = 0.360 + 0.018 + 0.250 = 0.628 |
| Youssef A. | 7.5 | 2.1 | 2 | 0.4×0.75 + 0.35×(1/3.1) + 0.25×0.85 = 0.300 + 0.113 + 0.213 = 0.626 |

**Selected: Ahmed M.** (score 0.645)

In this example, Ahmed's combination of high rating and short distance edges out Khaled (higher rating but much further) and Youssef (closer but lower rating and more workload).

---

## Appendix B — Glossary

| Term | Definition |
|---|---|
| Head Referee | Committee administrator responsible for managing assignments |
| Referee | A registered match official in the committee pool |
| Assignment | A referee-role pairing for a specific match |
| Role Slot | One of the four positions per match: COURT_1, COURT_2, SCORER, TIMEKEEPER |
| Scoring Formula | Weighted composite score used to rank referees for each assignment slot |
| Workload Penalty | Score reduction applied to referees with above-average current assignments |
| Conflict of Interest | Any condition that disqualifies a referee from a specific match for integrity reasons |
| Fallback Chain | The automatic sequence of replacement attempts triggered when a referee rejects an assignment |
| Emergency Pool | Opt-in group of referees available for last-minute assignments outside normal availability |
| Carry-Forward | Automatic reuse of a referee's previous week's availability when they fail to submit |
| Audit Log | Immutable record of all manual assignment changes |

---

*RefSync — Built to make every match officiated fairly, on time, and without friction.*
