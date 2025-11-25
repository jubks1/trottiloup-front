# Database Schema

## Overview

This document describes the database schema for the Trottiloup application, a scout event management system that handles registrations for multiple race categories. Units can register teams and participants across different races, with each race having its own pricing and requirements.

## Conceptual Schema

### Entity Relationships Diagram

```
Unit 1 ──── 1 Leader
Unit 1 ──── N Registration
Race 1 ──── N Registration
Registration 1 ──── N Team
```

## Entities

### Unit

Represents a scout group that registers teams or participants for the event.

| Field       | Type                              | Constraints | Description                 |
| ----------- | --------------------------------- | ----------- | --------------------------- |
| `id`        | Integer (auto-increment identity) | PRIMARY KEY | Unique identifier           |
| `unitName`  | String                            | NOT NULL    | Scout unit name             |
| `region`    | String                            | NULLABLE    | Optional region information |
| `createdAt` | Timestamp                         | NOT NULL    | Record creation timestamp   |

**Relationships:**

- One Unit has one Leader
- One Unit has many Registrations

---

### Leader

Represents the responsible adult contact for a unit.

| Field       | Type                              | Constraints                    | Description               |
| ----------- | --------------------------------- | ------------------------------ | ------------------------- |
| `id`        | Integer (auto-increment identity) | PRIMARY KEY                    | Unique identifier         |
| `unitId`    | Integer                           | FOREIGN KEY → Unit(id), UNIQUE | Reference to the unit     |
| `firstName` | String                            | NOT NULL                       | Leader's first name       |
| `lastName`  | String                            | NOT NULL                       | Leader's last name        |
| `email`     | String                            | NOT NULL                       | Contact email             |
| `phone`     | String                            | NOT NULL                       | Contact phone number      |
| `createdAt` | Timestamp                         | NOT NULL                       | Record creation timestamp |

**Relationships:**

- One Leader belongs to one Unit (1:1 relationship)

---

### Team

Represents teams participating in the event.

| Field              | Type                              | Constraints                    | Description               |
| ------------------ | --------------------------------- | ------------------------------ | ------------------------- |
| `id`               | Integer (auto-increment identity) | PRIMARY KEY                    | Unique identifier         |
| `registrationId`   | Integer                           | FOREIGN KEY → Registration(id) | Reference to registration |
| `teamName`         | String                            | NOT NULL                       | Team name                 |
| `participantCount` | Integer                           | NOT NULL, CHECK (value > 0)    | Number of participants    |
| `createdAt`        | Timestamp                         | NOT NULL                       | Record creation timestamp |

**Relationships:**

- Many Teams belong to one Registration

**Business Rules:**

- Team participant count must comply with the associated race's min/max requirements (validated at application level)
- Sum of all team participant counts for a registration should equal `Registration.totalParticipants`

---

### Race

Represents a race category for registrations (e.g., Louveteaux, Chef) with its own pricing, participant requirements, and optional rich description.

| Field                | Type                              | Constraints                                | Description                                             |
| -------------------- | --------------------------------- | ------------------------------------------ | ------------------------------------------------------- |
| `id`                 | Integer (auto-increment identity) | PRIMARY KEY                                | Unique identifier                                       |
| `name`               | String                            | NOT NULL, UNIQUE                           | Race name (e.g., "LOUVETEAUX", "CHEF")                  |
| `participationPrice` | Numeric                           | NOT NULL                                   | Base price per registration or per team (business rule) |
| `minParticipants`    | Integer                           | NOT NULL, CHECK (value > 0)                | Minimum number of participants required                 |
| `maxParticipants`    | Integer                           | NOT NULL, CHECK (value >= minParticipants) | Maximum number of participants allowed                  |
| `raceDate`           | Date                              | NOT NULL                                   | Date when the race takes place                          |
| `description`        | Text                              | NULLABLE                                   | Optional description, UTF-8 (emojis allowed)            |
| `createdAt`          | Timestamp                         | NOT NULL                                   | Record creation timestamp                               |

**Relationships:**

- One Race has many Registrations

**Notes:**

- Stored in PostgreSQL using `UTF8` encoding; emojis supported out of the box.
- `participationPrice` SHOULD use `NUMERIC(10,2)` for currency handling.

---

### Registration

A canonical record that links units and participation together. This entity serves as a central point for analytics, admin dashboard, and future extensibility.

| Field               | Type                              | Constraints                 | Description                         |
| ------------------- | --------------------------------- | --------------------------- | ----------------------------------- |
| `id`                | Integer (auto-increment identity) | PRIMARY KEY                 | Unique identifier                   |
| `unitId`            | Integer                           | FOREIGN KEY → Unit(id)      | Reference to the unit               |
| `raceId`            | Integer                           | FOREIGN KEY → Race(id)      | Reference to the race               |
| `totalParticipants` | Integer                           | NOT NULL                    | Total number of participants        |
| `totalPrice`        | Numeric                           | NOT NULL                    | Aggregated registration price       |
| `paymentStatus`     | Enum                              | NOT NULL, DEFAULT "PENDING" | Payment status: "PENDING" or "PAID" |
| `createdAt`         | Timestamp                         | NOT NULL                    | Record creation timestamp           |

**Relationships:**

- Many Registrations belong to one Unit
- One Registration has many Teams

**Enums:**

- `paymentStatus`: `PENDING`, `PAID`

---

## Indexes

Recommended indexes for query performance:

- `Leader`: Index on `unitId` (unique)
- `Team`: Index on `registrationId`
- `Race`: Unique index on `name`
- `Registration`: Index on `unitId`, Index on `raceId`, Index on `paymentStatus`

## Notes

- **Participant Constraints**: Each race defines its own `minParticipants` and `maxParticipants` requirements. Application logic must validate that team and registration participant counts fall within the associated race's limits.
- **Pricing Logic**: `participationPrice` on `Race` drives how `totalPrice` is computed (e.g., per team or per participant - define in business layer).
- **Payment Tracking**: Payment status is tracked directly on the Registration entity with `paymentStatus` field (PENDING or PAID).
- **Leader Uniqueness**: Each unit has exactly one leader, enforced by a unique constraint on `unitId` in the Leader table.
- **PostgreSQL**: Schema targets PostgreSQL; use `GENERATED BY DEFAULT AS IDENTITY` (or `SERIAL` where acceptable) for auto-increment integer primary keys, `NUMERIC(10,2)` for prices, `TEXT` for descriptions with emoji support.

## Future Considerations

- Consider adding soft delete functionality (deletedAt timestamp) for audit trails
- Add audit log tables for tracking changes to critical entities
- Consider adding a Participant entity if individual participant tracking becomes necessary
- May need to add additional fields for competition results and rankings
