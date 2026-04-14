# ADR-002: Auto-Escalation Rules Design

## Status

Accepted

## Date

2026-04-14

## Context

### Problem Statement

Manual moderation does not scale. When a user accumulates multiple
reports, an administrator must notice the pattern, evaluate severity,
and decide on a sanction. This introduces delay, inconsistency (different
admins may sanction differently for similar patterns), and the risk that
a genuinely abusive user continues to harm others while waiting for
human review.

Auto-escalation solves this by automatically applying sanctions when
a user's report count exceeds configured thresholds within a sliding
time window.

### Design Goals

1. **Predictability**: Users and administrators should be able to
   understand exactly when auto-escalation triggers. No ML black boxes
   for v1.
2. **Configurability**: Thresholds must be adjustable without code
   changes, via Kubernetes ConfigMaps.
3. **Override capability**: Administrators must be able to override
   auto-escalation decisions (lift auto-sanctions, adjust thresholds).
4. **Transparency**: Auto-sanctions must be clearly marked as automatic
   in the audit trail and in user-facing notices.
5. **Graduated response**: Escalation should be proportional. First
   offenses get lighter sanctions; repeated violations get heavier ones.

## Decision

### Fixed Threshold Model

We implement a fixed threshold model with three escalation tiers:

| Tier   | Threshold          | Window  | Action         | Duration   |
|--------|--------------------|---------|----------------|------------|
| Mute   | 3 reports          | 7 days  | Temporary mute | 24 hours   |
| Ban    | 5 reports          | 14 days | Account ban    | Permanent  |
| Review | 10 reports         | 30 days | Flag for review| N/A        |

#### Configuration via ConfigMap

All thresholds are externalized as environment variables loaded from
a Kubernetes ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: moderation-config
  namespace: whispr
data:
  MOD_MUTE_THRESHOLD: "3"
  MOD_MUTE_DAYS: "7"
  MOD_MUTE_DURATION_HOURS: "24"
  MOD_BAN_THRESHOLD: "5"
  MOD_BAN_DAYS: "14"
  MOD_REVIEW_THRESHOLD: "10"
  MOD_REVIEW_DAYS: "30"
```

This configuration is deployed identically to prod and preprod
environments, though values can be adjusted per-environment if needed.

#### Threshold Evaluation Algorithm

When a new report is filed against a user, the messaging-service
executes the following check:

```
function checkAutoEscalation(reportedUserId):
    recentReports = countReports(
        userId = reportedUserId,
        status IN ('pending', 'reviewed'),
        createdAt >= NOW() - INTERVAL '{MOD_BAN_DAYS} days'
    )

    if recentReports >= MOD_BAN_THRESHOLD:
        publishEscalation(type='ban', userId=reportedUserId)
        return

    recentReportsMute = countReports(
        userId = reportedUserId,
        status IN ('pending', 'reviewed'),
        createdAt >= NOW() - INTERVAL '{MOD_MUTE_DAYS} days'
    )

    if recentReportsMute >= MOD_MUTE_THRESHOLD:
        publishEscalation(type='mute', userId=reportedUserId,
                          duration=MOD_MUTE_DURATION_HOURS)
        return

    recentReportsReview = countReports(
        userId = reportedUserId,
        status IN ('pending', 'reviewed'),
        createdAt >= NOW() - INTERVAL '{MOD_REVIEW_DAYS} days'
    )

    if recentReportsReview >= MOD_REVIEW_THRESHOLD:
        publishEscalation(type='review', userId=reportedUserId)
        return
```

The check evaluates from most severe (ban) to least severe (review),
applying only the highest applicable tier.

#### Sliding Window

The time windows use a sliding window approach (current time minus N
days) rather than fixed calendar windows. This means:

- A user who receives 3 reports on day 1 and no reports for 8 days
  will not be auto-muted on day 9 because the early reports have
  fallen outside the 7-day window.
- A user who receives 2 reports on day 1 and 1 report on day 6 will
  be auto-muted because all 3 reports fall within the 7-day window.

This approach is more fair than fixed windows because it does not
penalize users for the arbitrary timing of window boundaries.

#### Idempotency

Auto-escalation checks are idempotent. If a user already has an
active mute sanction and receives another report that triggers the
mute threshold again, no duplicate mute is applied. The system checks
for existing active sanctions of the same type before applying a new
one.

However, if a user has an active mute and reaches the ban threshold,
the ban will be applied (escalation to a higher tier is always
permitted).

### Why Fixed Thresholds Over Trust Scores (for v1)

#### Trust Score Approach (Considered and Deferred)

A trust score system would assign each user a dynamic score based on
their behavior history:

- Start at 100 points.
- Lose points for received reports (-5 per report), sanctions (-20
  per sanction), and report abuse (-10 per false report filed).
- Gain points for successful appeals (+15), time without incidents
  (+1 per week), and positive contributions.
- Sanction triggers would be based on score thresholds rather than
  raw report counts.

**Why this is better in theory:**

- Accounts for user history and rehabilitation.
- Reduces the impact of coordinated false reporting (a high-trust
  user requires more reports to trigger escalation).
- Enables graduated responses based on overall behavior pattern.

**Why we chose fixed thresholds for v1:**

1. **Simplicity**: Fixed thresholds are easy to understand, implement,
   test, and explain to users. A trust score system requires careful
   calibration of point values, decay rates, and threshold curves.

2. **Transparency**: When a user asks "why was I muted?", the answer
   is simple: "You received 3 reports in 7 days." A trust score
   answer would be: "Your trust score dropped below 60 due to a
   combination of reports, previous sanctions, and time-weighted
   decay," which is harder to communicate.

3. **Predictability for admins**: Administrators can look at a user's
   recent report count and immediately know whether auto-escalation
   will trigger. Trust scores require dashboards and score history
   visualization.

4. **Low moderation volume**: Whispr is early-stage. The expected
   moderation volume does not justify the complexity of a trust score
   system. Fixed thresholds are sufficient for the current scale.

5. **Iteration path**: We can migrate from fixed thresholds to trust
   scores without changing the external API. The auto-escalation
   check is internal to the messaging-service and can be swapped
   without affecting sanctions, appeals, or admin workflows.

### Override Capability

Administrators can override auto-escalation in several ways:

1. **Lift auto-sanctions**: Any admin can lift an auto-applied sanction
   with a reason. This is recorded in the audit log.

2. **Adjust thresholds**: Modify the ConfigMap values and restart the
   affected pods (or wait for ConfigMap refresh if dynamic reloading
   is implemented).

3. **Whitelist users**: Future enhancement. A whitelist of user IDs
   excluded from auto-escalation for verified accounts, partner
   accounts, or users who have been falsely targeted.

4. **Dismiss reports**: Admins can dismiss reports that are clearly
   false, which reduces the report count for threshold evaluation.
   Dismissed reports do not count toward auto-escalation.

### Auto-Sanction Identification

Auto-applied sanctions are marked with:
- `is_auto: true` in the sanctions table.
- `issued_by` set to a system UUID (`00000000-0000-0000-0000-000000000000`).
- The audit log entry records `actor_role: 'system'`.

This allows:
- Dashboards to filter auto vs. manual sanctions.
- Analytics to track auto-escalation effectiveness separately.
- Users to see that the sanction was automatic (and may be more
  inclined to appeal, which is desirable for catching false positives).

## Consequences

### Positive

1. **Immediate response**: Abusive users are sanctioned within seconds
   of crossing a threshold, rather than waiting hours or days for
   admin review.

2. **Consistency**: The same behavior always produces the same
   escalation outcome. No variation based on which admin happens to
   be on duty.

3. **Configurable without deploys**: Threshold adjustments require
   only a ConfigMap change and pod restart, not a code deployment.

4. **Clear upgrade path**: The threshold check function is isolated
   and can be replaced with a trust score system when the platform
   matures.

### Negative

1. **False positive risk**: Coordinated false reporting by multiple
   users can trigger auto-escalation against an innocent user. The
   appeal process mitigates this, but the user experiences disruption.

2. **Threshold gaming**: Sophisticated abusers may stay just below
   thresholds (2 reports in 7 days, then wait). This is a known
   limitation of fixed thresholds that trust scores would address.

3. **No content awareness**: The system counts reports without
   evaluating content severity. One report for spam and one report
   for violence are weighted equally. Content-aware weighting is
   deferred to a future iteration.

## Future: Trust Score System

When moderation volume justifies the investment, the trust score
system should be implemented with the following design principles:

1. **Score range**: 0 to 100, starting at 100 for new users.
2. **Decay**: Negative events decay over time (half-life of 90 days).
3. **Weighted reports**: Reports weighted by category severity
   (harassment = 2x, spam = 0.5x).
4. **Reporter credibility**: Reports from users with high trust scores
   carry more weight than reports from users with low scores or
   histories of false reporting.
5. **Rehabilitation**: Time without incidents gradually restores score.
6. **Transparency**: Users can see their trust score and the factors
   affecting it.

The trust score system should be introduced behind a feature flag,
running in shadow mode (calculating scores but not using them for
escalation) before being activated.

## References

- ADR-001: Moderation System Architecture
- Kubernetes ConfigMap documentation
- Redis pub/sub for escalation event delivery
