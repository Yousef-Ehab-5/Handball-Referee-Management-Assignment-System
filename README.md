# Handball Referee Management System

## Overview

The Handball Referee Management System is a APP-based platform designed to automate and optimize the process of assigning referees to handball matches.

Traditional scheduling methods using **Google Forms** and **Excel spreadsheets** often lead to missed assignments, unfair workload distribution, long travel distances, and communication delays. This system provides a centralized solution that streamlines referee management, match scheduling, assignment generation, and notifications.

---

## Problem Statement

Current referee assignment workflows rely heavily on manual processes:

- Referees submit availability through forms.
- Assignments are created manually using spreadsheets.
- Referees may miss their assigned matches.
- Travel distance is often not considered.
- Workloads may be distributed unfairly.
- Last-minute replacements require significant manual effort.

The goal of this project is to automate and optimize these operations.

---

## Features

### Referee Management
- Create and manage referee profiles.
- Store referee levels, and contact information.
- Track referee history and statistics.

### Availability Management
- Weekly availability submission.
- Time-slot based scheduling.
- Availability conflict detection.

### Match Management
- Create and manage matches.
- Assign venues and match details.
- Support multiple referee requirements per match.

### Automated Assignment Engine
- Assign referees based on:
  - Availability
  - Distance from venue
  - Experience level
  - Workload balance
- Reduce scheduling conflicts.

### Notification System
- Assignment notifications.
- Match reminders.
- Acceptance and rejection workflow.

### Replacement Management
- Automatically find alternative referees when an assignment is rejected.

### Analytics Dashboard
- Match statistics.
- Referee performance metrics.
- Workload analysis.
- Travel distance reports.

---

## System Architecture

```text
Frontend (React / Next.js)
            |
            v
Backend (Spring Boot REST API)
            |
            v
PostgreSQL Database
            |
            +---- Maps Service
            |
            +---- Notification Service
```

---

## Technology Stack

### Frontend
- React.js
- Next.js
- Tailwind CSS

### Backend
- Java
- Spring Boot
- Spring Security
- JWT Authentication

### Database
- PostgreSQL

### APIs & Services
- Google Maps API / OpenStreetMap
- Firebase Cloud Messaging
- Email Notification Service

### Deployment
- Docker
- AWS / Azure

---

## Core Workflow

```text
Referee submits availability
            ↓
Head referee creates matches
            ↓
System finds eligible referees
            ↓
Assignment engine generates assignments
            ↓
Notifications are sent
            ↓
Referee accepts or rejects
            ↓
Automatic replacement if needed
            ↓
Match completed
            ↓
Statistics updated
```

---

## Assignment Algorithm

The system evaluates referees using multiple criteria:

- Availability
- Travel distance
- Experience level
- Current workload
- Historical performance

The highest-ranked eligible referees are selected automatically.

### Example Scoring Formula

Score =

(0.40 × Distance Score) +
(0.30 × Availability Score) +
(0.20 × Experience Score) +
(0.10 × Workload Balance Score)

The referees with the highest scores are assigned to the match.

---

## Database Design

### Referees

| Field | Type |
|---------|---------|
| id | Long |
| name | String |
| email | String |
| phone | String |
| level | String |
| rating | Double |
| latitude | Double |
| longitude | Double |

### Matches

| Field | Type |
|---------|---------|
| id | Long |
| homeTeam | String |
| awayTeam | String |
| matchDate | Date |
| venueId | Long |

### Assignments

| Field | Type |
|---------|---------|
| id | Long |
| refereeId | Long |
| matchId | Long |
| status | Enum |

### Availability

| Field | Type |
|---------|---------|
| id | Long |
| refereeId | Long |
| availableDate | Date |
| available | Boolean |

---

## Future Enhancements

- Mobile application (Android/iOS)
- GPS attendance verification
- AI-powered assignment recommendations
- Predictive referee availability analysis
- Route optimization
- WhatsApp integration
- Federation-wide management dashboard

---

## Project Status

🚧 In Development

Current Version: v1.0.0

Planned Releases:

- v1.0 — Core Scheduling System
- v1.5 — Automatic Assignment Engine
- v2.0 — Mobile Application
- v3.0 — AI-Powered Scheduling

---

## License

This project is developed for educational and organizational purposes and can be adapted for sports federations, leagues, and referee committees.

---

## Author

Yousef Ehab

Passionate about software engineering, optimization algorithms, sports technology, and building real-world solutions that solve operational challenges.
