# sport_federation_of_rwanda

## 1. Real Problem You Are Solving

Many youth competitions (U13, U17, U20) face age cheating, where players falsify their birth dates to compete in younger categories.

By integrating the database with National Identification Agency, your system ensures:

1. The player’s date of birth is verified from the national ID database

2. Any mismatch is flagged automatically

3. Suspicious cases are investigated

This could help organizations like Rwanda Football Federation or other sports federations.

## 2. Key Benefits of Your System
### 1️⃣ Prevent Age Fraud

Your module:

- nida_records

- nida_verification_log

- age_fraud_investigations

creates a complete verification workflow.

Example process:

1. Player registers.

2. System checks NIDA record.

3. If DOB mismatches → flag investigation.

4. Player cannot compete until verified.

### 2️⃣ Centralized Sports Database

Instead of each club keeping separate records, your system centralizes:

- Players

- Coaches

- Referees

- Clubs

- Federations

- Competitions

- Match statistics

This helps the entire sports ecosystem.

### 3️⃣ Fair Competition Management

Your tables support:

- Age categories (U13, U17, etc.)

- Eligibility checks

- Competition registrations

Your view v_competition_eligibility ensures players meet:

- Age requirements

- NIDA verification

- Federation rules

### 4️⃣ Player Career Tracking

Your system can track:

1. Player transfers

2. Match statistics

3. Season performance

4. Disciplinary records

This is similar to systems used by organizations like FIFA.

## 3. Why Your Design Is Strong

Your schema includes professional features:

✔ Geographic hierarchy (province → district → sector)
✔ Federation structure
✔ Club and team management
✔ Player registrations and transfers
✔ Match results and statistics
✔ Disciplinary management
✔ Anti-fraud investigation module
✔ User roles and auditing

This is enterprise-level design.

## 4. How to Make It Even More Beneficial
Add Automated Age Validation

A stored procedure can automatically check eligibility.

Example logic:

IF player_age < min_age OR player_age > max_age
THEN
   eligibility_status := 'NOT ELIGIBLE';
END IF;
Add Alerts

System can automatically alert federation officers if:

DOB mismatch detected

Suspicious age difference

Player tries to register twice

Add Analytics

You could create dashboards showing:

1. Age fraud cases

2. Active players by sport

3. Competition participation

4. Player performance

## 5. Possible Future Expansion

Your system could evolve into:

1.National Sports Management System

2. Mobile app for clubs

3. Integration with NIDA API

4. Player digital ID cards

5. Online competition management

## 6. Why This Project Is Valuable

Your project touches multiple fields:

1. Database design

2. Identity verification

3. Sports management

4. Fraud detection

5. Data analytics

That makes it useful not just academically but for real organizations.


# 🏆 Rwanda National Sports Federation — Database System

> A comprehensive multi-sport federation management database with NIDA identity & date-of-birth verification to eliminate age fraud in competitive sports.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Database Architecture](#database-architecture)
- [Schema Sections](#schema-sections)
  - [1. Geographic Structure](#1-geographic-structure)
  - [2. Sports & Disciplines](#2-sports--disciplines)
  - [3. Federation Hierarchy](#3-federation-hierarchy)
  - [4. Clubs & Teams](#4-clubs--teams)
  - [5. NIDA Integration](#5-nida-integration)
  - [6. Persons Registry](#6-persons-registry)
  - [7. Players](#7-players)
  - [8. Coaches & Officials](#8-coaches--officials)
  - [9. Competitions & Matches](#9-competitions--matches)
  - [10. Player Statistics](#10-player-statistics)
  - [11. Disciplinary Management](#11-disciplinary-management)
  - [12. Age Fraud Investigations](#12-age-fraud-investigations)
  - [13. System Users & Access Control](#13-system-users--access-control)
- [NIDA Anti-Fraud Logic](#nida-anti-fraud-logic)
- [Stored Functions & Triggers](#stored-functions--triggers)
- [Useful Views](#useful-views)
- [Sports Covered](#sports-covered)
- [Entity Relationship Summary](#entity-relationship-summary)
- [Setup & Installation](#setup--installation)
- [Next Steps](#next-steps)

---

## Overview

This database is designed for the **Rwanda National Sports Federation** to manage all aspects of sports administration across the country — from player registration and club management to competition scheduling and disciplinary enforcement.

The system's most critical feature is its **integration with Rwanda's NIDA (National Identification Agency)** database, which allows the federation to automatically verify each player's date of birth against official government records. This directly combats a widespread problem in youth football and other sports: **players falsifying their ages** to compete in under-age categories they are no longer eligible for.

### The Problem It Solves

| Problem | Solution |
|---|---|
| Players lying about date of birth | Auto-compare registered DOB against NIDA records |
| Underage category fraud | Block eligibility until NIDA verification passes |
| Duplicate player registrations | NID uniqueness constraint across all clubs |
| No centralised sports data | Single database for all 17+ sports |
| Manual fraud investigations | Auto-create investigation cases on large DOB mismatch |

---

## Key Features

- **Multi-sport support** — covers all Olympic and national sports disciplines
- **NIDA DOB verification** — every player verified against Rwanda's national ID database
- **Automatic fraud flagging** — mismatches > 90 days auto-create an investigation case
- **Eligibility enforcement** — players are blocked from competition until NIDA-verified
- **Full transfer tracking** — permanent transfers, loans, and returns between clubs
- **Competition management** — leagues, cups, knockout tournaments, fixtures, results
- **Disciplinary system** — cards, suspensions, bans, and appeal tracking
- **Role-based access control** — Super Admin down to Club-level users
- **Complete audit trail** — every data change logged with user, timestamp, and old/new values
- **Rwanda administrative hierarchy** — Province → District → Sector alignment

---

## Database Architecture

```
GEOGRAPHIC LAYER
└── provinces → districts → sectors

SPORTS LAYER
└── sport_categories → sports → age_categories

FEDERATION LAYER
└── federations (National → Provincial → District)

CLUB LAYER
└── clubs → teams

PERSON LAYER (NIDA-Verified)
└── nida_records ──► persons ──► players
                              ├── coaches
                              └── referees

COMPETITION LAYER
└── competitions → matches → player_match_stats

INTEGRITY LAYER
└── nida_verification_log
└── age_fraud_investigations
└── disciplinary_cases
└── audit_log
```

---

## Schema Sections

### 1. Geographic Structure

Aligned with Rwanda's official administrative divisions.

| Table | Description |
|---|---|
| `provinces` | 5 provinces of Rwanda (including Kigali City) |
| `districts` | 30 districts |
| `sectors` | Sectors within each district |

---

### 2. Sports & Disciplines

| Table | Description |
|---|---|
| `sport_categories` | Groupings: Team Sports, Individual, Combat, Aquatic, Racket, Athletics |
| `sports` | 17+ sports with governing body (FIFA, FIBA, World Athletics, etc.) |
| `age_categories` | Per-sport age brackets: U13, U15, U17, U20, Senior, Masters — with gender |

**Age Category Example (Football):**

| Category | Min Age | Max Age | Gender |
|---|---|---|---|
| U13 | — | 13 | M/F |
| U17 | — | 17 | M/F |
| U20 | — | 20 | M/F |
| Senior | 18 | — | M/F |

---

### 3. Federation Hierarchy

| Table | Description |
|---|---|
| `federations` | Supports NATIONAL, PROVINCIAL, and DISTRICT level federations |

Federations reference a `parent_fed_id` for hierarchical nesting. Each federation is linked to a specific sport.

---

### 4. Clubs & Teams

| Table | Description |
|---|---|
| `clubs` | Registered clubs with affiliation status (ACTIVE, SUSPENDED, REVOKED, PENDING) |
| `teams` | Season-specific teams per club and age category |
| `player_registrations` | Player's registration to a club per season |
| `player_transfers` | Full transfer history: PERMANENT, LOAN, FREE, RETURN_FROM_LOAN |

---

### 5. NIDA Integration

This is the **core anti-fraud engine** of the system.

| Table | Description |
|---|---|
| `nida_records` | Mirror of NIDA data pulled via API (read-only cache) |
| `nida_verification_log` | Audit trail of every DOB verification request |

**NIDA Record fields:**

```sql
national_id     CHAR(16)   -- Rwanda 16-digit NID
first_name      VARCHAR
last_name       VARCHAR
date_of_birth   DATE       -- The ground truth for age verification
gender          CHAR(1)
is_valid        BOOLEAN
last_verified   TIMESTAMP
raw_response    JSONB      -- Full API response stored for reference
```

**Verification Log fields:**

```sql
verification_type   -- REGISTRATION | COMPETITION_ELIGIBILITY | PERIODIC_AUDIT
dob_submitted       -- What the club/federation submitted
dob_from_nida       -- What NIDA returned
dob_match           -- TRUE/FALSE
mismatch_days       -- Absolute day difference
response_status     -- SUCCESS | MISMATCH | NOT_FOUND | TIMEOUT
```

---

### 6. Persons Registry

Central table for ALL individuals in the system (players, coaches, referees, staff).

**NIDA Verification Status per person:**

| Status | Meaning |
|---|---|
| `PENDING` | Not yet verified against NIDA |
| `VERIFIED` | DOB matches NIDA exactly |
| `MISMATCH` | Small discrepancy (1–90 days) — under review |
| `FLAGGED` | Large discrepancy (>90 days) — likely fraud |
| `NOT_FOUND` | NID not found in NIDA database |
| `EXEMPT` | Manually exempted (e.g. foreign players) |

> ⚠️ A person's `eligible_to_play` flag remains `FALSE` until their status reaches `VERIFIED`.

---

### 7. Players

| Table | Description |
|---|---|
| `players` | Player profile: position, height/weight, registration status |
| `player_registrations` | Club + team + season + jersey number |
| `player_transfers` | Transfer history with fees, dates, and approval workflow |

**Player Registration Status:**

```
ACTIVE → PENDING_VERIFICATION → SUSPENDED → BANNED → RETIRED
```

---

### 8. Coaches & Officials

| Table | Description |
|---|---|
| `coach_license_types` | License levels: CAF C, CAF B, CAF A, PRO |
| `coaches` | Licensed coaches with expiry tracking |
| `club_coaches` | Coach assignment to club/team with role |
| `referees` | Referee grades: International, National, Regional, Local |

All coaches and referees are also registered as `persons` and subject to NIDA verification.

---

### 9. Competitions & Matches

| Table | Description |
|---|---|
| `competition_types` | League, Cup, Friendly, Championship |
| `competitions` | Tournament with level: NATIONAL, PROVINCIAL, DISTRICT, INTERNATIONAL |
| `competition_participants` | Club entries per competition |
| `venues` | Stadiums and grounds with GPS coordinates and surface type |
| `matches` | Fixtures and results with referee assignment |

**Match Status Flow:**
```
SCHEDULED → ONGOING → COMPLETED
         ↓
    POSTPONED / CANCELLED / ABANDONED / FORFEITED
```

---

### 10. Player Statistics

| Table | Description |
|---|---|
| `player_match_stats` | Per-match: minutes played, goals, assists, cards |
| `player_season_stats` | Aggregated season totals per player per club |

Both tables include a `extra_stats JSONB` column to store sport-specific metrics (e.g. rebounds for basketball, try assists for rugby) without schema changes.

---

### 11. Disciplinary Management

| Table | Description |
|---|---|
| `disciplinary_cases` | Cases of type: YELLOW_CARD, RED_CARD, SUSPENSION, BAN, FINE, AGE_FRAUD, DOPING, MISCONDUCT |

**Case Status Flow:**
```
OPEN → UNDER_REVIEW → DECIDED → CLOSED
                   ↓
               APPEALED
```

---

### 12. Age Fraud Investigations

Dedicated table for tracking age falsification cases.

```sql
person_id             -- Who is being investigated
dob_on_file           -- What was registered
dob_from_nida         -- What NIDA says
dob_difference_days   -- Computed absolute difference
evidence_type         -- NIDA_MISMATCH | PHYSICAL_ASSESSMENT | TIPOFF
investigation_status  -- OPEN | UNDER_INVESTIGATION | CONFIRMED_FRAUD | CLEARED
action_taken          -- BANNED | SUSPENDED | CLEARED
```

> Cases with a DOB difference greater than **90 days** are **automatically created** by the system when NIDA verification runs.

---

### 13. System Users & Access Control

| Table | Description |
|---|---|
| `roles` | 9 roles from SUPER_ADMIN to VIEWER |
| `system_users` | Login accounts linked to persons, scoped to federation or club |
| `audit_log` | Every INSERT/UPDATE/DELETE logged with old and new values |

**Roles:**

| Role | Scope |
|---|---|
| `SUPER_ADMIN` | Full system access |
| `FED_ADMIN` | National federation administrator |
| `PROV_ADMIN` | Provincial federation administrator |
| `CLUB_ADMIN` | Club-level administrator |
| `REGISTRAR` | Player registration officer |
| `NIDA_OFFICER` | NIDA verification officer |
| `REFEREE_ADMIN` | Referee and officials manager |
| `COMPETITION_MGR` | Competition and fixtures manager |
| `VIEWER` | Read-only access |

---

## NIDA Anti-Fraud Logic

The DOB fraud detection flow works as follows:

```
1. Club submits player for registration
         │
         ▼
2. System sends NID to NIDA API
         │
         ▼
3. NIDA returns official date_of_birth
         │
         ▼
4. fn_check_dob_mismatch() compares DOBs
         │
    ┌────┴────┐
    │         │
  Match    Mismatch
    │         │
    ▼         ▼
VERIFIED   1–90 days → MISMATCH (manual review)
eligible   >90 days  → FLAGGED + auto-investigation
to_play              + player blocked immediately
```

**Key rule:** A player's `eligible_to_play` field is only set to `TRUE` after a clean NIDA verification (zero-day difference).

---

## Stored Functions & Triggers

### `fn_check_dob_mismatch(person_id, nida_dob)`
Called after every NIDA API response. Compares DOBs, updates person status, blocks player eligibility, and auto-creates fraud investigations for large mismatches.

### `fn_is_player_eligible(player_id, age_cat_id, check_date)`
Returns `(is_eligible BOOLEAN, reason TEXT)`. Checks NIDA verification status, suspension/ban status, and age category bounds. Used before every competition registration.

### `fn_update_timestamp()`
Trigger function on `persons`, `players`, `clubs`, `matches` — automatically sets `updated_at` on every UPDATE.

---

## Useful Views

| View | Purpose |
|---|---|
| `v_dob_mismatch_players` | All players with a DOB discrepancy, sorted by severity |
| `v_player_club_eligibility` | Player + club + current age + eligibility status |
| `v_competition_eligibility` | Cross-check players against age category bounds |
| `v_nida_verification_summary` | Dashboard totals: verified / pending / flagged / not found |

---

## Sports Covered

| Sport | Code | Governing Body | Olympic |
|---|---|---|---|
| Football | FOOTB | FIFA | ✅ |
| Basketball | BSKTB | FIBA | ✅ |
| Volleyball | VOLLB | FIVB | ✅ |
| Handball | HNDBL | IHF | ✅ |
| Rugby | RUGBY | World Rugby | ✅ |
| Cycling | CYCLE | UCI | ✅ |
| Triathlon | TRIAT | World Triathlon | ✅ |
| Weightlifting | WLIFT | IWF | ✅ |
| Boxing | BOXNG | AIBA | ✅ |
| Judo | JUDOO | IJF | ✅ |
| Taekwondo | TAEKW | WT | ✅ |
| Wrestling | WRSTL | UWW | ✅ |
| Swimming | SWIMM | FINA | ✅ |
| Tennis | TENNS | ITF | ✅ |
| Badminton | BADMN | BWF | ✅ |
| Table Tennis | TTNNS | ITTF | ✅ |
| Athletics | ATHLT | World Athletics | ✅ |

---

## Entity Relationship Summary

```
provinces
  └── districts
        └── sectors

sport_categories
  └── sports
        └── age_categories

federations (self-referencing hierarchy)
  └── clubs
        └── teams
              └── player_registrations ──► players ──► persons ──► nida_records
                                                   └── player_match_stats
                                                   └── player_season_stats
                                                   └── player_transfers

competitions
  └── competition_participants
  └── matches
        └── player_match_stats
        └── referees

persons
  ├── players
  ├── coaches ──► club_coaches
  └── referees

disciplinary_cases ──► age_fraud_investigations
audit_log ──► system_users ──► roles
```

---

## Setup & Installation

### Requirements

- PostgreSQL 14 or higher
- Access to Rwanda NIDA API (for live verification)

### Run the Schema

```bash
# Create the database
createdb rwanda_sports_federation

# Apply the full schema
psql -U postgres -d rwanda_sports_federation -f sports_federation_schema.sql

# Verify tables were created
psql -U postgres -d rwanda_sports_federation -c "\dt"
```

### Configure NIDA API Connection

Store your NIDA API credentials securely (e.g. environment variables or a secrets manager):

```env
NIDA_API_URL=https://nida.gov.rw/api/v1/verify
NIDA_API_KEY=your_api_key_here
NIDA_TIMEOUT_SECONDS=10
```

After each NIDA API response, call:

```sql
SELECT fn_check_dob_mismatch(:person_id, :nida_dob_returned);
```

### Check Fraud Dashboard

```sql
-- See all flagged players
SELECT * FROM v_dob_mismatch_players;

-- Overall verification health
SELECT * FROM v_nida_verification_summary;

-- Open fraud investigations
SELECT * FROM age_fraud_investigations
WHERE investigation_status = 'OPEN'
ORDER BY dob_difference_days DESC;
```

---

## Next Steps

| Phase | Deliverable | Description |
|---|---|---|
| **Phase 2** | Web Dashboard (UI) | Admin portal to manage players, clubs, and see NIDA flags in real time |
| **Phase 3** | NIDA API Integration | Backend service to call NIDA, handle timeouts, and store results |
| **Phase 4** | ERD Diagram | Visual entity-relationship diagram of all 30+ tables |
| **Phase 5** | Mobile App | Player ID card app for coaches and referees to scan NID on match day |
| **Phase 6** | Reporting Module | PDF/Excel exports for federation reports and audit submissions |

---

> **Database file:** `sports_federation_schema.sql` — 830 lines, PostgreSQL 14+
> **Maintained by:** Rwanda National Sports Federation IT Department
