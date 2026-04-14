# Moderation Flows

This document describes the step-by-step flows for each moderation
process in the Whispr platform. Each flow includes the actors involved,
the sequence of operations, cross-service communication, and the
resulting state changes.

## 1. Report Flow

### Actors
- **Reporter**: Authenticated user who files the report.
- **messaging-service**: Receives and stores the report, captures evidence.
- **user-service**: Receives escalation events, manages sanctions.
- **Redis**: Pub/sub channel for cross-service events.

### Steps

```
Reporter            messaging-service          Redis             user-service
   |                       |                     |                    |
   |  POST /reports        |                     |                    |
   |---------------------->|                     |                    |
   |                       |                     |                    |
   |                       | 1. Validate input   |                    |
   |                       | 2. Check duplicates  |                    |
   |                       | 3. Capture evidence  |                    |
   |                       |    snapshot          |                    |
   |                       | 4. Create report     |                    |
   |                       |    record (pending)  |                    |
   |                       |                     |                    |
   |  201 Created          |                     |                    |
   |<----------------------|                     |                    |
   |                       |                     |                    |
   |                       | 5. Count recent      |                    |
   |                       |    reports for user  |                    |
   |                       |                     |                    |
   |                       | 6. Check thresholds  |                    |
   |                       |    (mute/ban/review) |                    |
   |                       |                     |                    |
   |                       |  [IF threshold met]  |                    |
   |                       |                     |                    |
   |                       | publish escalation   |                    |
   |                       |--------------------->|                    |
   |                       |                     | deliver event       |
   |                       |                     |------------------->|
   |                       |                     |                    |
   |                       |                     |    7. Apply         |
   |                       |                     |    auto-sanction    |
   |                       |                     |    8. Update        |
   |                       |                     |    reputation       |
   |                       |                     |    9. Log audit     |
   |                       |                     |    entry            |
```

### Evidence Snapshot

When a report is created, the messaging-service captures:

1. **Message content**: The exact text of the reported message at the
   time of reporting. If the message is later edited or deleted, the
   original snapshot remains intact.

2. **Sender information**: The reported user's ID, username, and
   display name at the time of the report.

3. **Conversation context**: The conversation type (direct/group),
   participant count, and the conversation ID for admin review.

4. **Timestamp**: When the original message was sent and when the
   report was filed.

The evidence is stored as a JSONB blob in the report record:

```json
{
  "message_content": "The actual message text",
  "message_sender_id": "uuid",
  "message_sender_username": "john_doe",
  "message_created_at": "2026-04-14T10:30:00Z",
  "conversation_id": "uuid",
  "conversation_type": "group",
  "participant_count": 5,
  "reported_at": "2026-04-14T12:00:00Z"
}
```

### Validation Rules

- A user cannot report themselves (400 SELF_REPORT).
- A user cannot submit duplicate reports for the same message
  (409 DUPLICATE_REPORT).
- Report reasons must be one of the predefined categories.
- Rate limiting: maximum 10 reports per hour per user.

---

## 2. Sanction Flow

### Actors
- **Admin/Moderator**: Reviews reports and applies sanctions.
- **System**: Auto-escalation engine for threshold-based sanctions.
- **user-service**: Stores sanctions and manages user state.
- **messaging-service**: Receives sanction notifications.
- **Redis**: Pub/sub for cross-service events.

### Manual Sanction Steps

```
Admin               user-service              Redis          messaging-service
  |                      |                      |                   |
  | POST /sanctions      |                      |                   |
  |--------------------->|                      |                   |
  |                      |                      |                   |
  |                      | 1. Validate input     |                   |
  |                      | 2. Check existing     |                   |
  |                      |    active sanctions   |                   |
  |                      | 3. Create sanction    |                   |
  |                      |    record             |                   |
  |                      | 4. Update user        |                   |
  |                      |    reputation         |                   |
  |                      | 5. Log audit entry    |                   |
  |                      |                      |                   |
  |  201 Created         |                      |                   |
  |<---------------------|                      |                   |
  |                      |                      |                   |
  |                      | publish sanction      |                   |
  |                      |--------------------->|                   |
  |                      |                      | deliver event      |
  |                      |                      |------------------>|
  |                      |                      |                   |
  |                      |                      |  6. Update user    |
  |                      |                      |  messaging state   |
  |                      |                      |  (e.g., block      |
  |                      |                      |  message sending   |
  |                      |                      |  for muted users)  |
```

### Auto-Sanction Steps

Auto-sanctions follow the same flow as manual sanctions except:

1. The trigger is a Redis event from messaging-service rather than
   an admin API call.
2. The `issued_by` field is set to the system UUID
   (`00000000-0000-0000-0000-000000000000`).
3. The `is_auto` field is set to `true`.
4. The audit log records `actor_role: 'system'`.

### Sanction Types and Effects

| Type    | Effect                                              | Duration          |
|---------|-----------------------------------------------------|-------------------|
| warning | No functional restriction. Visible to user.         | N/A (permanent)   |
| mute    | User cannot send messages. Can still read.          | Configurable      |
| ban     | User cannot access the platform.                    | Permanent         |

### Expiration

Mute sanctions have an `expires_at` timestamp. The system checks
active sanctions on each message send attempt. If `expires_at` has
passed, the sanction is marked as expired (`is_active = false`) and
the user can resume messaging.

Ban sanctions do not expire automatically. They must be manually
lifted by an admin or lifted via a successful appeal.

---

## 3. Appeal Flow

### Actors
- **Sanctioned User**: Submits the appeal.
- **Admin**: Reviews and decides on the appeal.
- **user-service**: Manages appeal state and sanction updates.

### Steps

```
User                user-service                     Admin
  |                      |                             |
  | POST /appeals        |                             |
  |--------------------->|                             |
  |                      |                             |
  |                      | 1. Validate:                |
  |                      |    - Sanction exists         |
  |                      |    - Sanction is active      |
  |                      |    - User owns sanction      |
  |                      |    - No existing appeal      |
  |                      | 2. Create appeal record      |
  |                      | 3. Log audit entry           |
  |                      |                             |
  |  201 Created         |                             |
  |<---------------------|                             |
  |                      |                             |
  |                      |  [Appeal appears in          |
  |                      |   admin dashboard]           |
  |                      |                             |
  |                      |        PATCH /appeals/:id    |
  |                      |<----------------------------|
  |                      |                             |
  |                      | 4. Validate decision         |
  |                      |                             |
  |                      | [IF accepted]                |
  |                      | 5. Lift the sanction         |
  |                      |    (set is_active=false,     |
  |                      |     lifted_at, lifted_by,    |
  |                      |     lift_reason)             |
  |                      | 6. Update reputation         |
  |                      |    (+15 for successful       |
  |                      |     appeal)                  |
  |                      |                             |
  |                      | [IF rejected]                |
  |                      | 5. Record rejection          |
  |                      |    reason                    |
  |                      |                             |
  |                      | 7. Log audit entry           |
  |                      |                             |
  |                      |        200 OK                |
  |                      |---------------------------->|
  |                      |                             |
  |  [User notified of   |                             |
  |   appeal outcome]    |                             |
```

### Appeal Rules

1. Only the sanctioned user can appeal their own sanction.
2. Only one appeal per sanction is allowed.
3. Only active sanctions can be appealed (expired or already-lifted
   sanctions cannot be appealed).
4. Rate limit: maximum 3 appeals per day per user.
5. An accepted appeal automatically lifts the sanction.
6. A rejected appeal is final for that sanction (no re-appeal).

---

## 4. Auto-Escalation Flow

### Actors
- **messaging-service**: Evaluates thresholds on each new report.
- **Redis**: Delivers escalation events.
- **user-service**: Applies auto-sanctions.

### Steps

```
[New report filed]
        |
        v
messaging-service
        |
        | 1. Count reports against user
        |    in the past N days
        |    (N = MOD_BAN_DAYS)
        |
        v
   >= MOD_BAN_THRESHOLD?
        |
   YES -+-> Publish ban escalation to Redis
        |        |
   NO   |        v
        |   user-service applies ban
        |   (is_auto=true)
        |
        v
   Count reports in past
   MOD_MUTE_DAYS days
        |
        v
   >= MOD_MUTE_THRESHOLD?
        |
   YES -+-> Check: active mute exists?
        |        |
        |   YES: skip (idempotent)
        |   NO:  Publish mute escalation
        |        |
   NO   |        v
        |   user-service applies mute
        |   (duration=MOD_MUTE_DURATION_HOURS)
        |
        v
   Count reports in past
   MOD_REVIEW_DAYS days
        |
        v
   >= MOD_REVIEW_THRESHOLD?
        |
   YES -+-> Flag user for admin review
        |
   NO --+-> No action
```

### Threshold Configuration

All thresholds are loaded from the `moderation-config` ConfigMap:

| Variable               | Default | Description                        |
|------------------------|---------|------------------------------------|
| MOD_MUTE_THRESHOLD     | 3       | Reports to trigger auto-mute       |
| MOD_MUTE_DAYS          | 7       | Window for mute threshold (days)   |
| MOD_MUTE_DURATION_HOURS| 24      | Duration of auto-mute (hours)      |
| MOD_BAN_THRESHOLD      | 5       | Reports to trigger auto-ban        |
| MOD_BAN_DAYS           | 14      | Window for ban threshold (days)    |
| MOD_REVIEW_THRESHOLD   | 10      | Reports to trigger review flag     |
| MOD_REVIEW_DAYS        | 30      | Window for review threshold (days) |

### Redis Event Format

Escalation events published to `moderation:escalation`:

```json
{
  "type": "auto_escalation",
  "escalation_type": "mute",
  "user_id": "550e8400-e29b-41d4-a716-446655440001",
  "report_count": 3,
  "window_days": 7,
  "threshold": 3,
  "duration_hours": 24,
  "triggered_by_report_id": "550e8400-e29b-41d4-a716-446655440010",
  "timestamp": "2026-04-14T12:00:00Z"
}
```

### Edge Cases

1. **Rapid reports**: If 5 reports arrive within seconds, only one
   escalation event is published. The threshold check uses a database
   count, so concurrent checks may both trigger, but the user-service
   deduplicates by checking for existing active sanctions.

2. **Report dismissal after escalation**: If an admin dismisses reports
   after auto-escalation has triggered, the auto-sanction remains
   active. The admin must explicitly lift it if they believe it was
   incorrect.

3. **Appeal of auto-sanction**: Auto-sanctions can be appealed through
   the standard appeal flow. There is no special treatment.

4. **Threshold change**: If thresholds are lowered, existing users
   who now exceed the new threshold are not retroactively sanctioned.
   The check only runs when a new report is filed.

## References

- ADR-001: Moderation System Architecture
- ADR-002: Auto-Escalation Rules Design
- Moderation API Reference
