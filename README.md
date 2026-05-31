# Handball Referee Assignment System

## Overview

mobile application designed specifically for handball referee committees to automate referee assignment, reduce scheduling conflicts, and improve communication between head referees and referees.

Instead of manually assigning matches through spreadsheets and messaging groups, RefSync automatically selects the most suitable referees for each match based on availability, location, and referee ratings. The head referee can review and modify assignments before they are finalized.

The system aims to reduce missed matches, improve assignment fairness, and simplify the weekly scheduling process.

---

## Problem Statement

Currently, referee assignments are managed using Google Forms, Excel sheets, and manual communication channels. This process often results in:

* Referees missing their assignments.
* Long travel distances for some referees.
* Uneven distribution of matches.
* Time-consuming manual scheduling.
* Difficulty finding replacements when referees decline assignments.

RefSync addresses these challenges through automated assignment and notification workflows.

---

## User Roles

### Head Referee

The Head Referee is responsible for:

* Importing weekly match schedules.
* Reviewing automatically generated assignments.
* Editing assignments when necessary.
* Managing referee ratings.
* Monitoring referee availability.
* Approving final match assignments.

### Referee

Referees can:

* Create and manage their profiles.
* Submit availability.
* Receive assignment notifications.
* Accept or reject match assignments.
* View upcoming and past matches.

---

## Referee Profile Information

Each referee account contains:

* Referee ID
* Full Name
* Date of Birth
* Phone Number
* Email Address
* Home Address
* Work Address
* College Address

The system uses these locations to calculate travel distance and determine the most suitable assignments.

---
## Availability Management

Every referee must submit their weekly availability through the mobile application before a specified deadline.

The availability form allows referees to indicate:

- Available Days
- Available Time Ranges
- Unavailable Days
- Temporary Leave Periods

### Example

| Day | From | To | Status |
|------|------|------|---------|
| Friday | 4:00 PM | 11:00 PM | Available |
| Saturday | 8:00 AM | 5:00 PM | Available |
| Sunday | - | - | Unavailable |

### Weekly Workflow

1. The system opens availability submission every week.
2. Referees update their available times.
3. Submission closes at a predefined deadline.
4. The assignment engine only considers available referees.
5. Referees who do not submit availability are considered unavailable by default.

### Availability Rules

- A referee cannot be assigned outside their available time slots.
- Availability can be updated until the weekly deadline.
- Head Referees can view availability for all referees.
- Availability history is stored for future analysis.

## Match Import Process

The Head Referee uploads an Excel file containing:

* Team A
* Team B
* Match Date
* Match Time
* Match Venue

Example:

| Team A  | Team B  | Date        | Time    | Venue         |
| ------- | ------- | ----------- | ------- | ------------- |
| Al Ahly | Zamalek | 10 Jul 2026 | 7:00 PM | Cairo Stadium |

After importing the schedule, the system automatically begins the referee assignment process.

---

## Automatic Assignment Engine

For every imported match, the system automatically selects four referees.

The assignment process follows these steps:

### Step 1: Availability Filter

The system checks all referees and removes anyone who:

- Is unavailable on the match date.
- Is unavailable during the match time.
- Has already been assigned to another match at the same time.

### Step 2: Distance Calculation

The system calculates travel distance between:

- Home Address
- Work Address
- College Address

and the match venue.

The nearest suitable referees receive higher priority.

### Step 3: Rating Evaluation

The system considers:

- Court Referee Rating
- Table Referee Rating

Higher-rated referees receive priority for higher-level matches.

### Step 4: Workload Balancing

The system checks:

- Number of assignments this week.
- Number of assignments this month.

The goal is to distribute matches fairly among referees.

### Step 5: Final Selection

The four highest-ranked eligible referees are selected automatically.

The Head Referee can then:

- Approve assignments.
- Replace referees manually.
- Publish assignments.

## Assignment Workflow

Match Schedule Imported
↓
Automatic Referee Selection
↓
Head Referee Review
↓
Assignment Confirmation
↓
Notification Sent
↓
Accept / Reject Response
↓
Automatic Replacement (if rejected)

---

## Notification System

Once assignments are published, referees receive notifications through:

* Mobile App Notifications
* WhatsApp Notifications

Each notification contains:

* Match details
* Teams
* Date and Time
* Venue
* Response deadline

---

## Acceptance and Rejection Process

Each referee must respond before a specified deadline.

### Accept

The referee confirms attendance and is officially assigned to the match.

### Reject

If a referee declines the assignment:

* The system automatically searches for the next best available referee.
* A replacement notification is immediately sent.
* The Head Referee is informed of the change.

---

## Referee Rating System

Ratings are managed exclusively by the Head Referee.

Referees are evaluated in two categories:

### Table Referee Rating

Used for referees responsible for match administration and table operations.

### Court Referee Rating

Used for referees responsible for officiating on the playing court.

These ratings influence future assignment decisions.

---

## Technology Stack

### Mobile Application

* Flutter
* Android
* iOS

### Backend

* Spring Boot
* Java

### Database

* PostgreSQL

### Notifications

* Firebase Cloud Messaging
* WhatsApp Integration

### File Processing

* Excel Import Module

---

## Future Enhancements

* GPS-based attendance verification.
* AI-powered assignment recommendations.
* Travel optimization algorithms.
* Performance analytics dashboard.
* Federation-wide referee management.

---

## Project Goal

To build a centralized mobile platform that automates referee scheduling, minimizes assignment conflicts, improves communication, and ensures fair referee distribution across all handball matches.
