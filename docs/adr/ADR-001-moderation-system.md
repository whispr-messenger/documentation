# ADR-001: Moderation System Architecture

## Status

Accepted

## Date

2026-04-14

## Context

### Problem Statement

Whispr is a real-time messaging platform that enables direct and group
communication between users. As the platform scales, the risk of abuse
increases: spam, harassment, hate speech, violent content, and other
violations of community standards become inevitable. Without a robust
moderation system, the platform cannot guarantee user safety, comply with
legal obligations (DSA in the EU, platform liability laws), or maintain
the trust required for user retention.

### Current Situation

Before this decision, Whispr had no automated or structured moderation
capability. Abuse handling was entirely manual and ad-hoc, relying on
individual administrator intervention with no audit trail, no consistency
in sanctions, and no way for users to contest decisions. This approach
does not scale and exposes the platform to legal and reputational risk.

### Requirements

The moderation system must satisfy the following requirements:

1. **User reporting**: Any authenticated user can report a message or
   user for a specific reason (offensive content, spam, nudity/sexual
   content, violence, harassment, or other).
2. **Evidence preservation**: When a report is filed, the system must
   capture a snapshot of the reported content at that moment. Messages
   may be edited or deleted after reporting, so the original evidence
   must be preserved.
3. **Auto-escalation**: When a user accumulates reports above configured
   thresholds within a sliding time window, the system must automatically
   escalate the situation by applying sanctions without requiring manual
   admin intervention.
4. **Manual sanctions**: Administrators and moderators can manually apply
   sanctions (warnings, mutes, bans) to any user, with required reasons
   and evidence.
5. **Appeals**: Sanctioned users must be able to appeal their sanctions.
   Appeals are reviewed by administrators, and accepted appeals
   automatically lift the sanction.
6. **Audit trail**: Every moderation action (report, sanction, appeal,
   escalation) must be recorded with timestamps, actor identification,
   and full context for legal compliance and internal review.
7. **Analytics**: Administrators need visibility into moderation metrics:
   report volumes, sanction distributions, appeal outcomes, and
   auto-escalation frequency.
8. **Role-based access**: Only users with `admin` or `moderator` roles
   can perform moderation actions. Regular users can only file reports
   and submit appeals for their own sanctions.

### Constraints

- The system must integrate with existing services (messaging-service,
  user-service) without requiring a rewrite.
- Cross-service communication must use the existing Redis pub/sub
  infrastructure.
- The system must be deployable via the existing ArgoCD GitOps pipeline.
- Database changes must use the existing PostgreSQL instances managed
  by each service.
- The system must not introduce latency into the critical message
  delivery path.

## Decision

### Architecture: Distributed Ownership Model

We adopt a distributed ownership model where moderation responsibilities
are split across existing services based on domain alignment:

- **messaging-service** owns reports because reports are fundamentally
  about message content. The messaging-service already has access to
  message data, conversation context, and participant information.
  Placing report logic here avoids cross-service queries for evidence
  collection.

- **user-service** owns sanctions, appeals, roles, reputation, and
  audit logs because these are fundamentally about user state. Sanctions
  modify a user's ability to interact with the platform. Appeals contest
  user-level decisions. Roles define user permissions. All of these
  belong in the service that manages user identity and state.

- **moderation-service** acts as the coordination layer, providing a
  unified API gateway for admin dashboards and handling cross-service
  orchestration that does not naturally belong in either domain service.

### Pipeline: Report, Sanction, Appeal

The moderation pipeline follows a three-stage flow:

```
Report --> [Auto-Escalation Check] --> Sanction --> [Notification] --> Appeal
```

1. **Report stage**: A user files a report against a message or user.
   The messaging-service creates the report record, captures an evidence
   snapshot (message content, sender, timestamp, conversation context),
   and checks auto-escalation thresholds.

2. **Sanction stage**: If auto-escalation thresholds are met, or if an
   administrator manually decides, a sanction is applied via the
   user-service. The sanction record includes the type (warning, mute,
   ban), reason, evidence, duration (for mutes), and the acting
   administrator or "system" for auto-escalation.

3. **Appeal stage**: A sanctioned user can submit an appeal through the
   user-service. An administrator reviews the appeal and either accepts
   (lifting the sanction) or rejects it. The decision and reasoning are
   recorded.

### Cross-Service Communication

Services communicate moderation events through Redis pub/sub channels:

| Channel                  | Publisher         | Subscriber       | Payload                          |
|--------------------------|-------------------|------------------|----------------------------------|
| `moderation:report`      | messaging-service | user-service     | Report details + evidence        |
| `moderation:escalation`  | messaging-service | user-service     | Auto-escalation trigger          |
| `moderation:sanction`    | user-service      | messaging-service| Sanction applied notification    |
| `moderation:appeal`      | user-service      | messaging-service| Appeal outcome notification      |

Redis pub/sub was chosen over HTTP callbacks because:
- It decouples services temporally (publisher does not wait for consumer).
- It uses existing infrastructure (Redis is already deployed for caching
  and session management).
- It supports fan-out if additional services need to react to moderation
  events in the future.

### Data Model

#### Reports (messaging-service database)

```sql
CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reporter_id     UUID NOT NULL,
    reported_user_id UUID NOT NULL,
    message_id      UUID,
    conversation_id UUID,
    reason          VARCHAR(50) NOT NULL,
    description     TEXT,
    evidence        JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    reviewed_by     UUID,
    reviewed_at     TIMESTAMPTZ,
    admin_notes     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Sanctions (user-service database)

```sql
CREATE TABLE sanctions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    type            VARCHAR(20) NOT NULL,
    reason          TEXT NOT NULL,
    evidence        JSONB NOT NULL DEFAULT '{}',
    issued_by       UUID NOT NULL,
    is_auto         BOOLEAN NOT NULL DEFAULT false,
    report_id       UUID,
    expires_at      TIMESTAMPTZ,
    lifted_at       TIMESTAMPTZ,
    lifted_by       UUID,
    lift_reason     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Appeals (user-service database)

```sql
CREATE TABLE appeals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sanction_id     UUID NOT NULL REFERENCES sanctions(id),
    user_id         UUID NOT NULL,
    reason          TEXT NOT NULL,
    evidence        JSONB DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    reviewed_by     UUID,
    reviewed_at     TIMESTAMPTZ,
    admin_response  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Audit Log (user-service database)

```sql
CREATE TABLE moderation_audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    actor_id        UUID NOT NULL,
    actor_role      VARCHAR(20) NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    ip_address      VARCHAR(45),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### User Reputation (user-service database)

```sql
CREATE TABLE user_reputation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL UNIQUE,
    score           INTEGER NOT NULL DEFAULT 100,
    total_reports_received  INTEGER NOT NULL DEFAULT 0,
    total_reports_filed     INTEGER NOT NULL DEFAULT 0,
    total_sanctions         INTEGER NOT NULL DEFAULT 0,
    total_appeals           INTEGER NOT NULL DEFAULT 0,
    total_successful_appeals INTEGER NOT NULL DEFAULT 0,
    false_report_count      INTEGER NOT NULL DEFAULT 0,
    last_sanction_at        TIMESTAMPTZ,
    last_report_at          TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Why JSONB for Evidence

The `evidence` column uses PostgreSQL JSONB rather than normalized
relational tables for several reasons:

1. **Schema flexibility**: Evidence varies by report type. A message
   report captures message content, sender info, and conversation
   context. A user report might capture profile information, a pattern
   of messages, or external references. JSONB accommodates all these
   without schema migrations.

2. **Immutability semantics**: Evidence is a point-in-time snapshot.
   It is written once and never modified. JSONB naturally supports this
   pattern without the overhead of maintaining referential integrity
   for data that is fundamentally denormalized by design.

3. **Query capability**: PostgreSQL JSONB supports indexing and querying
   (GIN indexes, `@>` containment, `->>` field extraction), so we can
   still filter and search evidence when needed for analytics.

4. **Simplicity**: A single column with a documented JSON schema is
   simpler to manage than a constellation of evidence sub-tables with
   polymorphic relationships.

### Why Separate Tables Instead of a Single Moderation Table

We chose separate tables (reports, sanctions, appeals, audit_log,
reputation) over a single monolithic moderation_events table because:

1. **Different lifecycles**: Reports, sanctions, and appeals have
   distinct state machines. A report moves through pending/reviewed/
   dismissed. A sanction can be active/expired/lifted. An appeal goes
   through pending/accepted/rejected. Combining these into one table
   would require complex status logic and nullable columns.

2. **Different ownership**: Reports live in messaging-service. Sanctions
   and appeals live in user-service. A single table would require either
   cross-database joins or data duplication.

3. **Query patterns**: Admin dashboards query reports, sanctions, and
   appeals independently. Separate tables support efficient indexed
   queries for each entity type.

4. **Referential integrity**: Appeals reference sanctions. Sanctions
   optionally reference reports. These relationships are cleanly
   expressed with foreign keys between separate tables.

## Alternatives Considered

### Alternative 1: Single Monolithic Moderation Service

A dedicated moderation-service that owns all moderation data and logic:
reports, sanctions, appeals, audit trail, and analytics.

**Advantages:**
- Single source of truth for all moderation data.
- Simpler mental model: one service, one database.
- No cross-service communication needed for moderation workflows.

**Disadvantages:**
- Requires cross-service queries for evidence collection (fetching
  message content from messaging-service, user data from user-service).
- Creates a critical dependency: if the moderation service is down,
  no reports can be filed, no sanctions enforced, no appeals processed.
- Duplicates domain knowledge: the moderation service would need to
  understand message structure, user roles, and authentication, all
  of which are already handled by existing services.
- Increases deployment complexity with another stateful service.

**Why rejected:** The coupling required for evidence collection and
user state management outweighs the simplicity benefits. The
distributed model keeps domain logic where it belongs.

### Alternative 2: External Moderation Tool

Integrate a third-party moderation platform (e.g., Hive Moderation,
Amazon Rekognition for content, or a SaaS moderation dashboard).

**Advantages:**
- Faster time to market for content classification (ML-based detection).
- Mature admin dashboards with built-in analytics.
- Offloads moderation infrastructure maintenance.

**Disadvantages:**
- Vendor lock-in and ongoing SaaS costs that scale with message volume.
- Data sovereignty concerns: user messages and reports would transit
  through a third-party system.
- Limited customization of escalation rules and sanction policies.
- Integration complexity: webhooks, API adapters, and data
  synchronization between the external tool and our user/messaging
  databases.
- Latency: external API calls in the report/sanction path.

**Why rejected:** Data sovereignty requirements and the need for deep
integration with our user and messaging models make an in-house
solution more appropriate for v1. External ML-based content
classification can be added as a complementary layer in a future
iteration.

### Alternative 3: Event Sourcing for Moderation

Model all moderation actions as immutable events in an event store,
with materialized views for current state.

**Advantages:**
- Complete history by design, perfect for audit requirements.
- Temporal queries (state at any point in time) are natural.
- Supports complex event processing for pattern detection.

**Disadvantages:**
- Significant infrastructure overhead (event store, projection
  rebuilding, eventual consistency handling).
- The team lacks event sourcing experience, increasing implementation
  risk.
- Overkill for v1 where the moderation volume is expected to be low.
- The audit_log table provides sufficient history without the
  complexity of full event sourcing.

**Why rejected:** The complexity-to-benefit ratio is unfavorable for
v1. The audit_log table provides the necessary history. Event sourcing
can be reconsidered if moderation volume grows significantly.

## Consequences

### Positive

1. **Domain alignment**: Reports live where messages live. Sanctions
   live where user state lives. This minimizes cross-service data
   access and keeps each service's database schema coherent.

2. **Incremental deployment**: Each service can implement its moderation
   features independently. The messaging-service can ship report
   functionality before the user-service ships sanctions.

3. **Existing infrastructure reuse**: Redis pub/sub, PostgreSQL JSONB,
   ArgoCD deployment, and the existing RBAC system are all leveraged
   without new infrastructure components.

4. **Audit compliance**: The moderation_audit_log table provides a
   tamper-evident record of all moderation actions, satisfying legal
   requirements for platform moderation transparency.

5. **User trust**: The appeals process gives users recourse against
   incorrect sanctions, which is both ethically important and a legal
   requirement under the EU Digital Services Act.

6. **Scalability path**: The distributed model scales naturally because
   each service scales independently. High report volume does not
   affect sanction processing, and vice versa.

### Negative

1. **Operational complexity**: Moderation state is split across two
   databases. Debugging a moderation case requires querying both
   messaging-service and user-service databases. Admin dashboards
   must aggregate data from multiple API endpoints.

2. **Consistency**: Cross-service communication via Redis pub/sub is
   eventually consistent. There is a window where a report has been
   filed but the corresponding auto-escalation has not yet been
   processed. This is acceptable for moderation (seconds of delay
   are tolerable) but must be understood by the operations team.

3. **Testing complexity**: End-to-end moderation tests require both
   services to be running with their databases and a Redis instance.
   Unit tests for each service are straightforward, but integration
   testing is more involved.

4. **Schema coupling**: The evidence JSONB schema is implicitly shared
   between services. If messaging-service changes the evidence format,
   user-service dashboards that display evidence must be updated.
   This coupling is managed through documentation and versioning of
   the evidence schema.

5. **Future migration cost**: If we later decide to consolidate into
   a dedicated moderation service, we will need to migrate data from
   two databases and redirect API endpoints. The Redis pub/sub
   abstraction layer makes this feasible but not trivial.

## References

- EU Digital Services Act (DSA) - Articles 16-20 on content moderation
- PostgreSQL JSONB documentation
- Redis Pub/Sub documentation
- Whispr messaging-service API specification
- Whispr user-service API specification
