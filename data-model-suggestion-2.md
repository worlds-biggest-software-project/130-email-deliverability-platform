# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Email Deliverability Platform · Created: 2026-05-19

## Philosophy

This model treats every state change in the platform as an immutable event appended to a central event store. The event store is the single source of truth; all queryable state (reputation scores, placement summaries, authentication status) is derived by replaying or projecting events into materialised read models. This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the append-only event store, reads come from purpose-built projections.

Email deliverability is fundamentally an event-driven domain. Every email sent, delivered, bounced, opened, or marked as spam is an event. Every DNS check, DMARC report row, warm-up interaction, and reputation score change is an event. The event-sourced approach captures the full causal chain: "this domain's reputation dropped from 85 to 72 because of 340 bounces on March 14th, which triggered a blacklisting on Spamhaus SBL on March 15th, which caused inbox placement to drop to 60% on March 16th." Reconstructing this timeline from a normalised relational model requires complex joins across multiple tables; in an event-sourced model, the timeline is the data.

This pattern is used by financial systems (ledgers are event stores), audit-critical healthcare platforms, and analytics pipelines. Amazon SES's Virtual Deliverability Manager internally tracks events this way. The trade-off is complexity: you need projections for every read pattern, and the event store grows continuously.

**Best for:** Teams building a platform where complete audit trails, temporal queries ("what was the reputation on date X?"), AI-driven pattern analysis over historical event sequences, and cross-domain causal analysis are primary requirements.

**Trade-offs:**
- Pro: Complete audit trail by default — every state change is preserved forever
- Pro: Temporal queries are trivial — replay events to any point in time
- Pro: Ideal for AI/ML pipelines — event streams are natural training data for reputation forecasting models
- Pro: New read patterns can be added by creating new projections without changing the write model
- Pro: Natural fit for webhook event ingestion — events from ESPs flow directly into the store
- Con: Higher storage requirements — events are never deleted, only compacted
- Con: Requires projection infrastructure (workers that build read models from events)
- Con: Eventually consistent — projections lag behind the event store
- Con: More complex to query ad-hoc — cannot simply JOIN tables for novel queries without a projection
- Con: Team must understand event sourcing patterns; steeper learning curve

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 7208 (SPF) | SPF validation results stored as `authentication.spf_checked` events with full mechanism details |
| RFC 6376 (DKIM) | DKIM validation results stored as `authentication.dkim_checked` events with selector and key details |
| RFC 7489 (DMARC) | DMARC policy changes stored as `authentication.dmarc_policy_changed` events; RUA report rows ingested as `dmarc.report_row_received` events |
| DMARCbis (draft) | New tag support captured in event payload — no schema migration needed when tags change |
| RFC 8058 (One-Click Unsubscribe) | Compliance status changes stored as `compliance.unsubscribe_checked` events |
| RFC 8617 (ARC) | ARC chain validation results stored as `authentication.arc_checked` events |
| BIMI (IETF Draft) | BIMI record changes stored as `authentication.bimi_checked` events |
| SendGrid Event Taxonomy | ESP webhook events mapped to platform event types and appended directly to the event store |
| CAN-SPAM / GDPR / CCPA | Compliance state changes stored as events enabling point-in-time compliance auditing |

---

## Event Store (Source of Truth)

```sql
-- The central immutable event store
-- All state changes in the platform are recorded here
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_id       UUID NOT NULL,           -- aggregate root ID (domain, mailbox, test, etc.)
    stream_type     VARCHAR(100) NOT NULL,   -- domain, mailbox, warmup, placement_test, etc.
    event_type      VARCHAR(200) NOT NULL,   -- e.g., domain.registered, email.bounced, warmup.day_completed
    event_version   INT NOT NULL DEFAULT 1,  -- schema version of this event type
    sequence_num    BIGINT NOT NULL,          -- monotonically increasing per stream
    payload         JSONB NOT NULL,           -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation IDs, causation IDs, actor
    occurred_at     TIMESTAMPTZ NOT NULL,     -- when the event happened in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
);

-- Primary query patterns for the event store
CREATE INDEX idx_events_stream ON events (stream_id, sequence_num);
CREATE INDEX idx_events_tenant ON events (tenant_id, occurred_at);
CREATE INDEX idx_events_type ON events (event_type, occurred_at);
CREATE INDEX idx_events_occurred ON events (occurred_at);

-- Partition by month for manageable table sizes
-- In production: CREATE TABLE events (...) PARTITION BY RANGE (occurred_at);
-- CREATE TABLE events_2026_01 PARTITION OF events FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- etc.
```

### Event Type Catalogue

```sql
-- Registry of all known event types and their payload schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    category        VARCHAR(50) NOT NULL,    -- domain, mailbox, warmup, placement, email, authentication, compliance, alert
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,          -- JSON Schema for the payload
    version         INT NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Event Payloads

```sql
-- domain.registered
-- {
--   "domain_name": "acme.com",
--   "verified": false
-- }

-- authentication.spf_checked
-- {
--   "domain_name": "acme.com",
--   "raw_record": "v=spf1 include:_spf.google.com ~all",
--   "mechanisms": ["include:_spf.google.com"],
--   "qualifier": "~all",
--   "dns_lookup_count": 3,
--   "is_valid": true,
--   "errors": []
-- }

-- authentication.dmarc_policy_changed
-- {
--   "domain_name": "acme.com",
--   "previous_policy": "none",
--   "new_policy": "quarantine",
--   "raw_record": "v=DMARC1; p=quarantine; rua=mailto:dmarc@acme.com",
--   "adkim": "r",
--   "aspf": "r",
--   "pct": 100,
--   "np": null,
--   "psd": null,
--   "t": null
-- }

-- email.delivered
-- {
--   "recipient": "user@gmail.com",
--   "provider": "gmail",
--   "sg_event_id": "abc123",
--   "sg_message_id": "msg456"
-- }

-- email.bounced
-- {
--   "recipient": "bad@example.com",
--   "provider": "outlook",
--   "bounce_class": "invalid_address",
--   "smtp_status": "550",
--   "reason": "Mailbox not found",
--   "sg_event_id": "def789"
-- }

-- warmup.day_completed
-- {
--   "mailbox_email": "sales@acme.com",
--   "log_date": "2026-05-19",
--   "emails_sent": 35,
--   "replies_received": 28,
--   "marked_not_spam": 12,
--   "landed_inbox": 30,
--   "landed_spam": 3,
--   "landed_promotions": 2
-- }

-- placement.test_result_received
-- {
--   "test_id": "uuid-here",
--   "seed_email": "seed42@testnet.io",
--   "provider": "gmail",
--   "placement": "inbox",
--   "delivery_time_ms": 3200,
--   "spam_score": 1.2
-- }

-- dmarc.report_row_received
-- {
--   "report_id": "google-12345",
--   "reporting_org": "google.com",
--   "source_ip": "203.0.113.5",
--   "message_count": 142,
--   "disposition": "none",
--   "dkim_result": "pass",
--   "spf_result": "pass",
--   "dkim_domain": "acme.com",
--   "spf_domain": "acme.com",
--   "date_range_begin": "2026-05-18T00:00:00Z",
--   "date_range_end": "2026-05-19T00:00:00Z"
-- }

-- reputation.score_computed
-- {
--   "domain_name": "acme.com",
--   "score_date": "2026-05-19",
--   "overall_score": 87.5,
--   "inbox_rate": 92.1,
--   "spam_rate": 4.2,
--   "bounce_rate": 1.8,
--   "complaint_rate": 0.0012,
--   "emails_sent": 1250
-- }

-- alert.fired
-- {
--   "rule_id": "uuid-here",
--   "condition": "reputation_drop",
--   "severity": "warning",
--   "message": "Domain acme.com reputation dropped 8 points in 24 hours",
--   "domain_name": "acme.com",
--   "previous_score": 87.5,
--   "current_score": 79.5
-- }
```

## Projection Infrastructure

```sql
-- Tracks the position of each projection consumer in the event stream
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_sequence    BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(50) NOT NULL DEFAULT 'running'
    -- running, paused, rebuilding, failed
);
```

## Materialised Read Models (Projections)

### Tenant & Domain Read Model

```sql
-- Current state of tenants (projected from tenant.* events)
CREATE TABLE rm_tenants (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL,
    max_domains     INT NOT NULL,
    max_mailboxes   INT NOT NULL,
    domain_count    INT NOT NULL DEFAULT 0,
    mailbox_count   INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Current state of users (projected from user.* events)
CREATE TABLE rm_users (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    email           VARCHAR(320) NOT NULL,
    name            VARCHAR(255),
    role            VARCHAR(50) NOT NULL,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_users_tenant ON rm_users (tenant_id);

-- Current state of domains (projected from domain.* events)
CREATE TABLE rm_domains (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    domain_name     VARCHAR(253) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    mailbox_count   INT NOT NULL DEFAULT 0,
    -- Latest authentication status (projected from authentication.* events)
    spf_valid       BOOLEAN,
    spf_last_checked TIMESTAMPTZ,
    dkim_valid      BOOLEAN,
    dkim_last_checked TIMESTAMPTZ,
    dmarc_policy    VARCHAR(20),
    dmarc_valid     BOOLEAN,
    dmarc_last_checked TIMESTAMPTZ,
    bimi_valid      BOOLEAN,
    bimi_last_checked TIMESTAMPTZ,
    -- Latest reputation (projected from reputation.* events)
    reputation_score DECIMAL(5,2),
    reputation_date DATE,
    -- Compliance status
    has_one_click_unsubscribe BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_domains_tenant ON rm_domains (tenant_id);
CREATE INDEX idx_rm_domains_name ON rm_domains (domain_name);

-- Current state of mailboxes (projected from mailbox.* events)
CREATE TABLE rm_mailboxes (
    id              UUID PRIMARY KEY,
    domain_id       UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    email_address   VARCHAR(320) NOT NULL,
    provider        VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    warmup_status   VARCHAR(50),
    warmup_daily_target INT,
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_mailboxes_domain ON rm_mailboxes (domain_id);
CREATE INDEX idx_rm_mailboxes_tenant ON rm_mailboxes (tenant_id);
```

### Reputation Timeline Read Model

```sql
-- Daily reputation scores (projected from reputation.score_computed events)
CREATE TABLE rm_reputation_daily (
    domain_id       UUID NOT NULL,
    score_date      DATE NOT NULL,
    overall_score   DECIMAL(5,2) NOT NULL,
    inbox_rate      DECIMAL(5,2),
    spam_rate       DECIMAL(5,2),
    bounce_rate     DECIMAL(5,2),
    complaint_rate  DECIMAL(5,4),
    open_rate       DECIMAL(5,2),
    click_rate      DECIMAL(5,2),
    emails_sent     INT NOT NULL DEFAULT 0,
    PRIMARY KEY (domain_id, score_date)
);
CREATE INDEX idx_rm_reputation_date ON rm_reputation_daily (score_date);

-- IP reputation (projected from reputation.ip_score_computed events)
CREATE TABLE rm_ip_reputation (
    ip_address      INET NOT NULL,
    score_date      DATE NOT NULL,
    overall_score   DECIMAL(5,2) NOT NULL,
    blacklists      TEXT[],
    PRIMARY KEY (ip_address, score_date)
);
```

### Email Event Aggregation Read Model

```sql
-- Hourly email event aggregates per domain (projected from email.* events)
CREATE TABLE rm_email_stats_hourly (
    domain_id       UUID NOT NULL,
    hour_bucket     TIMESTAMPTZ NOT NULL,    -- truncated to hour
    provider        VARCHAR(50) NOT NULL,
    delivered       INT NOT NULL DEFAULT 0,
    bounced_hard    INT NOT NULL DEFAULT 0,
    bounced_soft    INT NOT NULL DEFAULT 0,
    deferred        INT NOT NULL DEFAULT 0,
    dropped         INT NOT NULL DEFAULT 0,
    opened          INT NOT NULL DEFAULT 0,
    clicked         INT NOT NULL DEFAULT 0,
    spam_reported   INT NOT NULL DEFAULT 0,
    unsubscribed    INT NOT NULL DEFAULT 0,
    PRIMARY KEY (domain_id, hour_bucket, provider)
);
CREATE INDEX idx_rm_email_stats_hour ON rm_email_stats_hourly (hour_bucket);

-- Daily email event aggregates per domain (rolled up from hourly)
CREATE TABLE rm_email_stats_daily (
    domain_id       UUID NOT NULL,
    stat_date       DATE NOT NULL,
    provider        VARCHAR(50) NOT NULL DEFAULT 'all',
    delivered       INT NOT NULL DEFAULT 0,
    bounced_hard    INT NOT NULL DEFAULT 0,
    bounced_soft    INT NOT NULL DEFAULT 0,
    deferred        INT NOT NULL DEFAULT 0,
    dropped         INT NOT NULL DEFAULT 0,
    opened          INT NOT NULL DEFAULT 0,
    clicked         INT NOT NULL DEFAULT 0,
    spam_reported   INT NOT NULL DEFAULT 0,
    unsubscribed    INT NOT NULL DEFAULT 0,
    inbox_rate      DECIMAL(5,2),
    bounce_rate     DECIMAL(5,2),
    complaint_rate  DECIMAL(5,4),
    PRIMARY KEY (domain_id, stat_date, provider)
);
CREATE INDEX idx_rm_email_stats_daily_date ON rm_email_stats_daily (stat_date);
```

### Warm-Up Progress Read Model

```sql
-- Current warm-up state per mailbox (projected from warmup.* events)
CREATE TABLE rm_warmup_status (
    mailbox_id      UUID PRIMARY KEY,
    status          VARCHAR(50) NOT NULL,
    strategy        VARCHAR(50) NOT NULL,
    daily_target    INT NOT NULL,
    daily_max       INT NOT NULL,
    current_day     INT NOT NULL DEFAULT 0,  -- days since warm-up started
    total_sent      INT NOT NULL DEFAULT 0,
    total_inbox     INT NOT NULL DEFAULT 0,
    total_spam      INT NOT NULL DEFAULT 0,
    inbox_rate_7d   DECIMAL(5,2),            -- rolling 7-day inbox rate
    started_at      TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ
);

-- Warm-up daily history (projected from warmup.day_completed events)
CREATE TABLE rm_warmup_daily (
    mailbox_id      UUID NOT NULL,
    log_date        DATE NOT NULL,
    emails_sent     INT NOT NULL DEFAULT 0,
    replies_received INT NOT NULL DEFAULT 0,
    marked_not_spam INT NOT NULL DEFAULT 0,
    landed_inbox    INT NOT NULL DEFAULT 0,
    landed_spam     INT NOT NULL DEFAULT 0,
    landed_promotions INT NOT NULL DEFAULT 0,
    PRIMARY KEY (mailbox_id, log_date)
);
```

### Placement Test Read Model

```sql
-- Placement test summaries (projected from placement.* events)
CREATE TABLE rm_placement_tests (
    id              UUID PRIMARY KEY,
    domain_id       UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    status          VARCHAR(50) NOT NULL,
    seed_count      INT NOT NULL DEFAULT 0,
    inbox_count     INT NOT NULL DEFAULT 0,
    spam_count      INT NOT NULL DEFAULT 0,
    promotions_count INT NOT NULL DEFAULT 0,
    missing_count   INT NOT NULL DEFAULT 0,
    inbox_rate      DECIMAL(5,2),
    avg_delivery_ms INT,
    avg_spam_score  DECIMAL(5,2),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_rm_placement_domain ON rm_placement_tests (domain_id);
CREATE INDEX idx_rm_placement_tenant ON rm_placement_tests (tenant_id);

-- Per-provider placement breakdown
CREATE TABLE rm_placement_by_provider (
    test_id         UUID NOT NULL,
    provider        VARCHAR(50) NOT NULL,
    seed_count      INT NOT NULL DEFAULT 0,
    inbox_count     INT NOT NULL DEFAULT 0,
    spam_count      INT NOT NULL DEFAULT 0,
    promotions_count INT NOT NULL DEFAULT 0,
    inbox_rate      DECIMAL(5,2),
    PRIMARY KEY (test_id, provider)
);
```

### DMARC Report Read Model

```sql
-- Aggregated DMARC report summaries (projected from dmarc.* events)
CREATE TABLE rm_dmarc_summary (
    domain_id       UUID NOT NULL,
    reporting_org   VARCHAR(255) NOT NULL,
    report_date     DATE NOT NULL,
    total_messages  BIGINT NOT NULL DEFAULT 0,
    pass_count      BIGINT NOT NULL DEFAULT 0,
    quarantine_count BIGINT NOT NULL DEFAULT 0,
    reject_count    BIGINT NOT NULL DEFAULT 0,
    dkim_pass       BIGINT NOT NULL DEFAULT 0,
    dkim_fail       BIGINT NOT NULL DEFAULT 0,
    spf_pass        BIGINT NOT NULL DEFAULT 0,
    spf_fail        BIGINT NOT NULL DEFAULT 0,
    unique_source_ips INT NOT NULL DEFAULT 0,
    PRIMARY KEY (domain_id, reporting_org, report_date)
);
CREATE INDEX idx_rm_dmarc_summary_date ON rm_dmarc_summary (report_date);

-- Unauthorised senders detected in DMARC reports
CREATE TABLE rm_dmarc_unauthorised_senders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL,
    source_ip       INET NOT NULL,
    first_seen      DATE NOT NULL,
    last_seen       DATE NOT NULL,
    total_messages  BIGINT NOT NULL DEFAULT 0,
    dkim_result     VARCHAR(20) NOT NULL,
    spf_result      VARCHAR(20) NOT NULL,
    disposition     VARCHAR(20) NOT NULL
);
CREATE INDEX idx_rm_dmarc_unauth_domain ON rm_dmarc_unauthorised_senders (domain_id);
```

### Alert Read Model

```sql
-- Active alerts (projected from alert.* events)
CREATE TABLE rm_active_alerts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    domain_id       UUID,
    rule_id         UUID NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    message         TEXT NOT NULL,
    acknowledged    BOOLEAN NOT NULL DEFAULT false,
    fired_at        TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_alerts_tenant ON rm_active_alerts (tenant_id);
CREATE INDEX idx_rm_alerts_unack ON rm_active_alerts (acknowledged) WHERE acknowledged = false;
```

## Snapshot Infrastructure

```sql
-- Periodic snapshots of aggregate state for fast replay
-- Instead of replaying all events from the beginning, replay from the latest snapshot
CREATE TABLE aggregate_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    sequence_num    BIGINT NOT NULL,         -- event sequence at which snapshot was taken
    state           JSONB NOT NULL,          -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_num)
);
```

---

## Example Queries

### Replay domain authentication history to any point in time

```sql
-- "What was acme.com's DMARC policy on March 1st 2026?"
SELECT payload->>'new_policy' AS dmarc_policy,
       payload->>'raw_record' AS raw_record,
       occurred_at
FROM events
WHERE stream_id = (SELECT id FROM rm_domains WHERE domain_name = 'acme.com')
  AND event_type = 'authentication.dmarc_policy_changed'
  AND occurred_at <= '2026-03-01T23:59:59Z'
ORDER BY sequence_num DESC
LIMIT 1;
```

### Causal chain analysis for a reputation drop

```sql
-- "What happened to acme.com between May 14 and May 17?"
SELECT event_type, occurred_at, payload
FROM events
WHERE stream_id = (SELECT id FROM rm_domains WHERE domain_name = 'acme.com')
  AND occurred_at BETWEEN '2026-05-14' AND '2026-05-17'
ORDER BY sequence_num;
-- Returns: email.bounced (x340) -> blacklist.listed -> reputation.score_computed (72) -> alert.fired
```

### Feed AI model with event sequences

```sql
-- Export event sequences for reputation forecasting model training
SELECT stream_id,
       event_type,
       occurred_at,
       payload->'bounce_rate' AS bounce_rate,
       payload->'complaint_rate' AS complaint_rate,
       payload->'overall_score' AS score
FROM events
WHERE event_type IN ('reputation.score_computed', 'email.bounced', 'email.spam_reported')
  AND occurred_at >= now() - INTERVAL '90 days'
ORDER BY stream_id, sequence_num;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events (partitioned), event_type_registry |
| Projection Infrastructure | 2 | projection_checkpoints, aggregate_snapshots |
| Identity Read Models | 4 | rm_tenants, rm_users, rm_domains, rm_mailboxes |
| Reputation Read Models | 2 | rm_reputation_daily, rm_ip_reputation |
| Email Stats Read Models | 2 | rm_email_stats_hourly, rm_email_stats_daily |
| Warm-Up Read Models | 2 | rm_warmup_status, rm_warmup_daily |
| Placement Read Models | 2 | rm_placement_tests, rm_placement_by_provider |
| DMARC Read Models | 2 | rm_dmarc_summary, rm_dmarc_unauthorised_senders |
| Alert Read Model | 1 | rm_active_alerts |
| **Total** | **19** | Plus the event store which holds all data |

---

## Key Design Decisions

1. **Single event store table with JSONB payloads** rather than one event table per aggregate type. This simplifies the write path (one INSERT per event regardless of type), enables cross-aggregate correlation queries, and means new event types require zero schema changes — just a new entry in `event_type_registry`.

2. **`stream_id` + `sequence_num` for optimistic concurrency** — the UNIQUE constraint on `(stream_id, sequence_num)` prevents conflicting writes to the same aggregate. Writers read the current max sequence_num for a stream, then INSERT with sequence_num + 1; a duplicate key error signals a concurrency conflict that must be retried.

3. **Monthly range partitioning on `occurred_at`** keeps the event table manageable. Older partitions can be moved to cold storage or compressed. Projections reference events by ID, so partition pruning does not affect read model accuracy.

4. **Read models are prefixed `rm_`** to distinguish projections from source-of-truth tables. Projections can be dropped and rebuilt at any time by replaying events from the store. This makes schema evolution for read models risk-free.

5. **Hourly and daily email stats aggregations** are separate read models because dashboards need both granularities. The hourly model enables near-real-time monitoring; the daily model powers trend charts and reputation calculations.

6. **DMARC report rows are ingested as individual events** (`dmarc.report_row_received`) rather than bulk-loaded into a relational table. This preserves the temporal ordering of report ingestion and allows re-processing if the parsing logic changes.

7. **Aggregate snapshots** avoid the "long replay" problem. For a domain with 100,000 events, replaying from the beginning is expensive. Snapshots capture the aggregate state at a point in time; replay starts from the latest snapshot. Snapshots are taken periodically (e.g., every 1,000 events per stream).

8. **Event metadata includes correlation and causation IDs** enabling distributed tracing. When a `reputation.score_computed` event triggers an `alert.fired` event, the alert's metadata.causation_id points to the reputation event's ID.

9. **No foreign keys on the event store** — the event store does not reference read model tables. Read models reference event IDs for traceability but are not constrained. This decouples the write and read paths completely, allowing independent scaling.

10. **Event payloads are self-describing** — each event contains enough context to be useful without joining other events. For example, `email.bounced` includes the recipient, provider, bounce class, and SMTP status in its payload, not just a foreign key to a bounce_classifications table.
