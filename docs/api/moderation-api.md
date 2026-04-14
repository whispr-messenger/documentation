# Moderation API Reference

## Overview

The Whispr moderation API is distributed across two services:

- **messaging-service** (`/v1/moderation/reports/*`) -- Report management
  and evidence collection.
- **user-service** (`/v1/moderation/*`) -- Sanctions, appeals, roles,
  audit trail, reputation, and webhooks.

All endpoints require Bearer token authentication. Admin and moderator
endpoints require the `admin` or `moderator` role.

## Base URLs

| Environment | messaging-service             | user-service                  |
|-------------|-------------------------------|-------------------------------|
| Production  | `https://api.whispr.chat`     | `https://api.whispr.chat`     |
| Preprod     | `https://preprod.whispr.chat` | `https://preprod.whispr.chat` |

Requests are routed to the correct service by the API gateway based on
the URL path prefix.

## Authentication

All requests must include a valid JWT in the Authorization header:

```
Authorization: Bearer <jwt-token>
```

Tokens are issued by the auth-service. The JWT payload includes the
user's `id` and `role` fields, which are used for authorization checks.

### Role Requirements

| Role      | Can do                                                  |
|-----------|---------------------------------------------------------|
| user      | File reports, view own reports, appeal own sanctions     |
| moderator | All user actions + review reports, apply sanctions       |
| admin     | All moderator actions + manage roles, view audit logs    |

---

## Messaging-Service Endpoints

### Reports

#### POST /v1/moderation/reports

Create a new report against a message or user.

**Authentication:** Required (any authenticated user)

**Request Body:**

```json
{
  "reported_user_id": "550e8400-e29b-41d4-a716-446655440001",
  "message_id": "550e8400-e29b-41d4-a716-446655440002",
  "conversation_id": "550e8400-e29b-41d4-a716-446655440003",
  "reason": "harassment",
  "description": "User is sending threatening messages repeatedly"
}
```

| Field              | Type   | Required | Description                                        |
|--------------------|--------|----------|----------------------------------------------------|
| reported_user_id   | UUID   | Yes      | ID of the user being reported                      |
| message_id         | UUID   | No       | ID of the specific message (null for user reports)  |
| conversation_id    | UUID   | No       | ID of the conversation context                     |
| reason             | string | Yes      | One of: `offensive`, `spam`, `nudity`, `violence`, `harassment`, `other` |
| description        | string | No       | Free-text description from the reporter            |

**Response: 201 Created**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440010",
  "reporter_id": "550e8400-e29b-41d4-a716-446655440000",
  "reported_user_id": "550e8400-e29b-41d4-a716-446655440001",
  "message_id": "550e8400-e29b-41d4-a716-446655440002",
  "conversation_id": "550e8400-e29b-41d4-a716-446655440003",
  "reason": "harassment",
  "description": "User is sending threatening messages repeatedly",
  "evidence": {
    "message_content": "Original message text captured at report time",
    "message_sender_id": "550e8400-e29b-41d4-a716-446655440001",
    "message_created_at": "2026-04-14T10:30:00Z",
    "conversation_type": "direct",
    "participant_count": 2
  },
  "status": "pending",
  "created_at": "2026-04-14T12:00:00Z",
  "updated_at": "2026-04-14T12:00:00Z"
}
```

**Error Responses:**

| Status | Code                  | Description                              |
|--------|-----------------------|------------------------------------------|
| 400    | INVALID_REASON        | Reason is not one of the allowed values  |
| 400    | MISSING_REPORTED_USER | reported_user_id is required             |
| 400    | SELF_REPORT           | Cannot report yourself                   |
| 404    | MESSAGE_NOT_FOUND     | message_id does not exist                |
| 404    | USER_NOT_FOUND        | reported_user_id does not exist          |
| 409    | DUPLICATE_REPORT      | You already reported this message        |
| 429    | RATE_LIMITED          | Too many reports in a short period       |

---

#### GET /v1/moderation/reports

List reports with filtering and pagination.

**Authentication:** Required (admin or moderator)

**Query Parameters:**

| Parameter        | Type   | Default  | Description                          |
|------------------|--------|----------|--------------------------------------|
| status           | string | all      | Filter by: `pending`, `reviewed`, `dismissed`, `escalated` |
| reason           | string | all      | Filter by reason category            |
| reported_user_id | UUID   | none     | Filter by reported user              |
| reporter_id      | UUID   | none     | Filter by reporter                   |
| page             | int    | 1        | Page number                          |
| limit            | int    | 20       | Results per page (max 100)           |
| sort             | string | created_at | Sort field                         |
| order            | string | desc     | Sort order: `asc` or `desc`          |

**Response: 200 OK**

```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "reporter_id": "550e8400-e29b-41d4-a716-446655440000",
      "reported_user_id": "550e8400-e29b-41d4-a716-446655440001",
      "message_id": "550e8400-e29b-41d4-a716-446655440002",
      "conversation_id": "550e8400-e29b-41d4-a716-446655440003",
      "reason": "harassment",
      "description": "User is sending threatening messages repeatedly",
      "evidence": { "..." : "..." },
      "status": "pending",
      "reviewed_by": null,
      "reviewed_at": null,
      "admin_notes": null,
      "created_at": "2026-04-14T12:00:00Z",
      "updated_at": "2026-04-14T12:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "total_pages": 3
  }
}
```

---

#### GET /v1/moderation/reports/:id

Get a single report by ID.

**Authentication:** Required (admin/moderator, or the reporter for their own reports)

**Response: 200 OK**

Returns the full report object as shown in the list response.

**Error Responses:**

| Status | Code            | Description                    |
|--------|-----------------|--------------------------------|
| 403    | FORBIDDEN       | Not authorized to view report  |
| 404    | REPORT_NOT_FOUND| Report does not exist          |

---

#### PATCH /v1/moderation/reports/:id

Update a report's status (review, dismiss, escalate).

**Authentication:** Required (admin or moderator)

**Request Body:**

```json
{
  "status": "reviewed",
  "admin_notes": "Confirmed harassment. Sanction applied."
}
```

| Field       | Type   | Required | Description                                           |
|-------------|--------|----------|-------------------------------------------------------|
| status      | string | Yes      | New status: `reviewed`, `dismissed`, `escalated`       |
| admin_notes | string | No       | Internal notes from the reviewing admin               |

**Response: 200 OK**

Returns the updated report object.

**Error Responses:**

| Status | Code               | Description                         |
|--------|--------------------|-------------------------------------|
| 400    | INVALID_STATUS     | Status transition is not allowed    |
| 403    | FORBIDDEN          | Not authorized                      |
| 404    | REPORT_NOT_FOUND   | Report does not exist               |

---

#### GET /v1/moderation/reports/user/:userId

Get all reports filed against a specific user.

**Authentication:** Required (admin or moderator)

**Query Parameters:** Same pagination and sorting as GET /v1/moderation/reports.

**Response: 200 OK**

Same format as the reports list response.

---

#### GET /v1/moderation/reports/stats

Get report statistics and analytics.

**Authentication:** Required (admin)

**Query Parameters:**

| Parameter  | Type   | Default    | Description                      |
|------------|--------|------------|----------------------------------|
| start_date | string | 30d ago    | Start of period (ISO 8601)       |
| end_date   | string | now        | End of period (ISO 8601)         |
| group_by   | string | day        | Grouping: `day`, `week`, `month` |

**Response: 200 OK**

```json
{
  "summary": {
    "total_reports": 156,
    "pending": 23,
    "reviewed": 98,
    "dismissed": 30,
    "escalated": 5,
    "auto_escalations": 3
  },
  "by_reason": {
    "harassment": 45,
    "spam": 38,
    "offensive": 32,
    "violence": 18,
    "nudity": 12,
    "other": 11
  },
  "timeline": [
    {
      "date": "2026-04-01",
      "count": 12,
      "by_status": {
        "pending": 2,
        "reviewed": 8,
        "dismissed": 2
      }
    }
  ]
}
```

---

## User-Service Endpoints

### Sanctions

#### POST /v1/moderation/sanctions

Apply a sanction to a user.

**Authentication:** Required (admin or moderator)

**Request Body:**

```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440001",
  "type": "mute",
  "reason": "Repeated harassment after warnings",
  "evidence": {
    "report_ids": [
      "550e8400-e29b-41d4-a716-446655440010",
      "550e8400-e29b-41d4-a716-446655440011"
    ],
    "summary": "3 harassment reports in 5 days"
  },
  "duration_hours": 24,
  "report_id": "550e8400-e29b-41d4-a716-446655440010"
}
```

| Field          | Type    | Required | Description                                    |
|----------------|---------|----------|------------------------------------------------|
| user_id        | UUID    | Yes      | ID of the user to sanction                     |
| type           | string  | Yes      | One of: `warning`, `mute`, `ban`               |
| reason         | string  | Yes      | Explanation for the sanction                   |
| evidence       | object  | No       | Supporting evidence                            |
| duration_hours | int     | Cond.    | Required for `mute` type. Duration in hours.   |
| report_id      | UUID    | No       | Associated report that triggered this sanction |

**Response: 201 Created**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440020",
  "user_id": "550e8400-e29b-41d4-a716-446655440001",
  "type": "mute",
  "reason": "Repeated harassment after warnings",
  "evidence": { "..." : "..." },
  "issued_by": "550e8400-e29b-41d4-a716-446655440099",
  "is_auto": false,
  "report_id": "550e8400-e29b-41d4-a716-446655440010",
  "expires_at": "2026-04-15T12:00:00Z",
  "lifted_at": null,
  "lifted_by": null,
  "lift_reason": null,
  "is_active": true,
  "created_at": "2026-04-14T12:00:00Z",
  "updated_at": "2026-04-14T12:00:00Z"
}
```

**Error Responses:**

| Status | Code                  | Description                              |
|--------|-----------------------|------------------------------------------|
| 400    | INVALID_TYPE          | Sanction type is not valid               |
| 400    | MISSING_DURATION      | duration_hours required for mute type    |
| 400    | ALREADY_BANNED        | User already has an active ban           |
| 403    | FORBIDDEN             | Not authorized to sanction users         |
| 404    | USER_NOT_FOUND        | Target user does not exist               |

---

#### GET /v1/moderation/sanctions

List all sanctions with filtering.

**Authentication:** Required (admin or moderator)

**Query Parameters:**

| Parameter | Type    | Default | Description                              |
|-----------|---------|---------|------------------------------------------|
| user_id   | UUID    | none    | Filter by sanctioned user                |
| type      | string  | all     | Filter by: `warning`, `mute`, `ban`      |
| is_active | boolean | all     | Filter by active status                  |
| is_auto   | boolean | all     | Filter by auto-escalation                |
| page      | int     | 1       | Page number                              |
| limit     | int     | 20      | Results per page (max 100)               |

**Response: 200 OK**

```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440020",
      "user_id": "550e8400-e29b-41d4-a716-446655440001",
      "type": "mute",
      "reason": "Repeated harassment after warnings",
      "evidence": { "..." : "..." },
      "issued_by": "550e8400-e29b-41d4-a716-446655440099",
      "is_auto": false,
      "report_id": "550e8400-e29b-41d4-a716-446655440010",
      "expires_at": "2026-04-15T12:00:00Z",
      "lifted_at": null,
      "is_active": true,
      "created_at": "2026-04-14T12:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 12,
    "total_pages": 1
  }
}
```

---

#### GET /v1/moderation/sanctions/:id

Get a single sanction by ID.

**Authentication:** Required (admin/moderator, or the sanctioned user for their own)

**Response: 200 OK**

Returns the full sanction object.

---

#### GET /v1/moderation/sanctions/user/:userId

Get all sanctions for a specific user.

**Authentication:** Required (admin/moderator, or the user themselves)

**Response: 200 OK**

Same format as the sanctions list response.

---

#### PATCH /v1/moderation/sanctions/:id/lift

Lift (cancel) an active sanction.

**Authentication:** Required (admin)

**Request Body:**

```json
{
  "reason": "Sanction was applied incorrectly after investigation"
}
```

| Field  | Type   | Required | Description                  |
|--------|--------|----------|------------------------------|
| reason | string | Yes      | Reason for lifting sanction  |

**Response: 200 OK**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440020",
  "user_id": "550e8400-e29b-41d4-a716-446655440001",
  "type": "mute",
  "is_active": false,
  "lifted_at": "2026-04-14T14:00:00Z",
  "lifted_by": "550e8400-e29b-41d4-a716-446655440099",
  "lift_reason": "Sanction was applied incorrectly after investigation"
}
```

**Error Responses:**

| Status | Code                  | Description                      |
|--------|-----------------------|----------------------------------|
| 400    | ALREADY_LIFTED        | Sanction is already inactive     |
| 403    | FORBIDDEN             | Only admins can lift sanctions   |
| 404    | SANCTION_NOT_FOUND    | Sanction does not exist          |

---

#### GET /v1/moderation/sanctions/stats

Get sanction statistics.

**Authentication:** Required (admin)

**Query Parameters:** Same as report stats (start_date, end_date, group_by).

**Response: 200 OK**

```json
{
  "summary": {
    "total_sanctions": 87,
    "active": 12,
    "expired": 45,
    "lifted": 30,
    "by_type": {
      "warning": 40,
      "mute": 35,
      "ban": 12
    },
    "auto_sanctions": 15,
    "manual_sanctions": 72
  },
  "timeline": [
    {
      "date": "2026-04-01",
      "count": 5,
      "by_type": {
        "warning": 2,
        "mute": 2,
        "ban": 1
      }
    }
  ]
}
```

---

### Appeals

#### POST /v1/moderation/appeals

Submit an appeal against a sanction.

**Authentication:** Required (the sanctioned user)

**Request Body:**

```json
{
  "sanction_id": "550e8400-e29b-41d4-a716-446655440020",
  "reason": "I was having a private joke with a friend, not harassing them",
  "evidence": {
    "context": "The reported user is my roommate, we joke like this"
  }
}
```

| Field       | Type   | Required | Description                         |
|-------------|--------|----------|-------------------------------------|
| sanction_id | UUID   | Yes      | ID of the sanction to appeal        |
| reason      | string | Yes      | Why the sanction should be reversed |
| evidence    | object | No       | Supporting evidence for the appeal  |

**Response: 201 Created**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440030",
  "sanction_id": "550e8400-e29b-41d4-a716-446655440020",
  "user_id": "550e8400-e29b-41d4-a716-446655440001",
  "reason": "I was having a private joke with a friend, not harassing them",
  "evidence": { "..." : "..." },
  "status": "pending",
  "reviewed_by": null,
  "reviewed_at": null,
  "admin_response": null,
  "created_at": "2026-04-14T13:00:00Z",
  "updated_at": "2026-04-14T13:00:00Z"
}
```

**Error Responses:**

| Status | Code                   | Description                                 |
|--------|------------------------|---------------------------------------------|
| 400    | SANCTION_NOT_YOURS     | Can only appeal your own sanctions          |
| 400    | SANCTION_NOT_ACTIVE    | Cannot appeal an inactive sanction          |
| 409    | APPEAL_EXISTS          | An appeal already exists for this sanction  |
| 404    | SANCTION_NOT_FOUND     | Sanction does not exist                     |

---

#### GET /v1/moderation/appeals

List appeals with filtering.

**Authentication:** Required (admin or moderator)

**Query Parameters:**

| Parameter   | Type   | Default | Description                           |
|-------------|--------|---------|---------------------------------------|
| status      | string | all     | Filter: `pending`, `accepted`, `rejected` |
| user_id     | UUID   | none    | Filter by appellant                   |
| sanction_id | UUID   | none    | Filter by sanction                    |
| page        | int    | 1       | Page number                           |
| limit       | int    | 20      | Results per page (max 100)            |

**Response: 200 OK**

Same pagination structure as other list endpoints.

---

#### GET /v1/moderation/appeals/:id

Get a single appeal by ID.

**Authentication:** Required (admin/moderator, or the appellant)

**Response: 200 OK**

Returns the full appeal object.

---

#### PATCH /v1/moderation/appeals/:id

Review an appeal (accept or reject).

**Authentication:** Required (admin)

**Request Body:**

```json
{
  "status": "accepted",
  "admin_response": "After reviewing the conversation context, the messages were clearly between friends. Sanction lifted."
}
```

| Field          | Type   | Required | Description                              |
|----------------|--------|----------|------------------------------------------|
| status         | string | Yes      | New status: `accepted` or `rejected`     |
| admin_response | string | Yes      | Explanation of the decision              |

When an appeal is accepted, the associated sanction is automatically
lifted with the reason "Appeal accepted: [admin_response]".

**Response: 200 OK**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440030",
  "sanction_id": "550e8400-e29b-41d4-a716-446655440020",
  "status": "accepted",
  "reviewed_by": "550e8400-e29b-41d4-a716-446655440099",
  "reviewed_at": "2026-04-14T15:00:00Z",
  "admin_response": "After reviewing the conversation context, the messages were clearly between friends. Sanction lifted."
}
```

**Error Responses:**

| Status | Code                 | Description                        |
|--------|----------------------|------------------------------------|
| 400    | ALREADY_REVIEWED     | Appeal has already been decided    |
| 400    | INVALID_STATUS       | Status must be accepted or rejected|
| 400    | MISSING_RESPONSE     | admin_response is required         |
| 403    | FORBIDDEN            | Only admins can review appeals     |
| 404    | APPEAL_NOT_FOUND     | Appeal does not exist              |

---

### Roles

#### GET /v1/moderation/roles

List users with moderation roles.

**Authentication:** Required (admin)

**Query Parameters:**

| Parameter | Type   | Default | Description                       |
|-----------|--------|---------|-----------------------------------|
| role      | string | all     | Filter: `admin`, `moderator`      |
| page      | int    | 1       | Page number                       |
| limit     | int    | 20      | Results per page                  |

**Response: 200 OK**

```json
{
  "data": [
    {
      "user_id": "550e8400-e29b-41d4-a716-446655440099",
      "username": "admin_sarah",
      "role": "admin",
      "assigned_at": "2026-01-15T10:00:00Z",
      "assigned_by": "550e8400-e29b-41d4-a716-446655440098"
    }
  ],
  "pagination": { "..." : "..." }
}
```

---

#### PATCH /v1/moderation/roles/:userId

Assign or update a user's moderation role.

**Authentication:** Required (admin)

**Request Body:**

```json
{
  "role": "moderator"
}
```

| Field | Type   | Required | Description                               |
|-------|--------|----------|-------------------------------------------|
| role  | string | Yes      | One of: `user`, `moderator`, `admin`      |

Setting role to `user` effectively removes moderation privileges.

**Response: 200 OK**

```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440050",
  "role": "moderator",
  "assigned_at": "2026-04-14T12:00:00Z",
  "assigned_by": "550e8400-e29b-41d4-a716-446655440099"
}
```

**Error Responses:**

| Status | Code               | Description                        |
|--------|--------------------|------------------------------------|
| 400    | INVALID_ROLE       | Role is not one of allowed values  |
| 403    | FORBIDDEN          | Only admins can assign roles       |
| 404    | USER_NOT_FOUND     | User does not exist                |

---

### Audit Log

#### GET /v1/moderation/audit

Query the moderation audit log.

**Authentication:** Required (admin)

**Query Parameters:**

| Parameter   | Type   | Default    | Description                             |
|-------------|--------|------------|-----------------------------------------|
| action      | string | all        | Filter by action type                   |
| entity_type | string | all        | Filter: `report`, `sanction`, `appeal`  |
| entity_id   | UUID   | none       | Filter by specific entity               |
| actor_id    | UUID   | none       | Filter by actor                         |
| start_date  | string | 30d ago    | Start of period (ISO 8601)              |
| end_date    | string | now        | End of period (ISO 8601)                |
| page        | int    | 1          | Page number                             |
| limit       | int    | 50         | Results per page (max 200)              |

**Response: 200 OK**

```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440040",
      "action": "sanction_applied",
      "entity_type": "sanction",
      "entity_id": "550e8400-e29b-41d4-a716-446655440020",
      "actor_id": "550e8400-e29b-41d4-a716-446655440099",
      "actor_role": "admin",
      "details": {
        "sanction_type": "mute",
        "target_user_id": "550e8400-e29b-41d4-a716-446655440001",
        "duration_hours": 24,
        "reason": "Repeated harassment"
      },
      "ip_address": "192.168.1.100",
      "created_at": "2026-04-14T12:00:00Z"
    }
  ],
  "pagination": { "..." : "..." }
}
```

**Audit Action Types:**

| Action                 | Description                               |
|------------------------|-------------------------------------------|
| report_created         | New report filed                          |
| report_reviewed        | Report reviewed by admin                  |
| report_dismissed       | Report dismissed by admin                 |
| report_escalated       | Report escalated for further action       |
| sanction_applied       | Sanction applied to user                  |
| sanction_lifted        | Sanction manually lifted                  |
| sanction_expired       | Sanction expired automatically            |
| sanction_auto_applied  | Auto-escalation sanction applied          |
| appeal_submitted       | User submitted an appeal                  |
| appeal_accepted        | Appeal accepted by admin                  |
| appeal_rejected        | Appeal rejected by admin                  |
| role_changed           | User moderation role changed              |

---

### Reputation

#### GET /v1/moderation/reputation/:userId

Get a user's moderation reputation.

**Authentication:** Required (admin/moderator, or the user themselves)

**Response: 200 OK**

```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440001",
  "score": 75,
  "total_reports_received": 5,
  "total_reports_filed": 2,
  "total_sanctions": 2,
  "total_appeals": 1,
  "total_successful_appeals": 1,
  "false_report_count": 0,
  "last_sanction_at": "2026-04-10T08:00:00Z",
  "last_report_at": "2026-04-14T12:00:00Z",
  "created_at": "2026-03-01T00:00:00Z",
  "updated_at": "2026-04-14T12:00:00Z"
}
```

---

#### GET /v1/moderation/reputation/leaderboard

Get users sorted by reputation score (lowest first -- most problematic users).

**Authentication:** Required (admin)

**Query Parameters:**

| Parameter | Type   | Default | Description                      |
|-----------|--------|---------|----------------------------------|
| order     | string | asc     | Sort order: `asc` (worst first)  |
| page      | int    | 1       | Page number                      |
| limit     | int    | 20      | Results per page                 |

**Response: 200 OK**

```json
{
  "data": [
    {
      "user_id": "550e8400-e29b-41d4-a716-446655440001",
      "username": "problematic_user",
      "score": 35,
      "total_sanctions": 4,
      "total_reports_received": 12
    }
  ],
  "pagination": { "..." : "..." }
}
```

---

### Webhooks

#### POST /v1/moderation/webhooks

Register a webhook for moderation events.

**Authentication:** Required (admin)

**Request Body:**

```json
{
  "url": "https://hooks.slack.com/services/T00/B00/xxx",
  "events": ["sanction_applied", "appeal_submitted", "auto_escalation"],
  "secret": "whsec_abc123"
}
```

| Field  | Type     | Required | Description                               |
|--------|----------|----------|-------------------------------------------|
| url    | string   | Yes      | HTTPS endpoint to receive webhook payloads|
| events | string[] | Yes      | List of event types to subscribe to       |
| secret | string   | No       | Secret for HMAC signature verification    |

**Webhook Events:**

| Event              | Triggered when                            |
|--------------------|-------------------------------------------|
| report_created     | New report filed                          |
| sanction_applied   | Manual or auto sanction applied           |
| sanction_lifted    | Sanction lifted                           |
| appeal_submitted   | User submitted an appeal                  |
| appeal_decided     | Admin decided on an appeal                |
| auto_escalation    | Auto-escalation threshold crossed         |

**Webhook Payload Example:**

```json
{
  "event": "sanction_applied",
  "timestamp": "2026-04-14T12:00:00Z",
  "data": {
    "sanction_id": "550e8400-e29b-41d4-a716-446655440020",
    "user_id": "550e8400-e29b-41d4-a716-446655440001",
    "type": "mute",
    "is_auto": false,
    "issued_by": "550e8400-e29b-41d4-a716-446655440099"
  },
  "signature": "sha256=abc123..."
}
```

**Response: 201 Created**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440050",
  "url": "https://hooks.slack.com/services/T00/B00/xxx",
  "events": ["sanction_applied", "appeal_submitted", "auto_escalation"],
  "is_active": true,
  "created_at": "2026-04-14T12:00:00Z"
}
```

---

#### GET /v1/moderation/webhooks

List registered webhooks.

**Authentication:** Required (admin)

**Response: 200 OK**

```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440050",
      "url": "https://hooks.slack.com/services/T00/B00/xxx",
      "events": ["sanction_applied", "appeal_submitted"],
      "is_active": true,
      "created_at": "2026-04-14T12:00:00Z",
      "last_triggered_at": "2026-04-14T14:30:00Z",
      "failure_count": 0
    }
  ]
}
```

---

#### DELETE /v1/moderation/webhooks/:id

Delete a webhook registration.

**Authentication:** Required (admin)

**Response: 204 No Content**

---

## Common Error Format

All error responses follow this format:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable description of the error",
    "details": {}
  }
}
```

## Rate Limiting

| Endpoint Category      | Rate Limit              |
|------------------------|-------------------------|
| Report creation        | 10 per hour per user    |
| Appeal submission       | 3 per day per user      |
| Admin list/stats       | 100 per minute          |
| Webhook delivery       | 1000 per hour           |

Rate limit headers are included in all responses:

```
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 7
X-RateLimit-Reset: 1681473600
```

## Pagination

All list endpoints support cursor-based pagination with the following
response structure:

```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

Maximum `limit` is 100 for standard endpoints and 200 for audit log
queries.
