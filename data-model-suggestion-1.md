# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Email Deliverability Platform · Created: 2026-05-19

## Philosophy

This model follows a fully normalized relational design where every concept in the email deliverability domain maps to a dedicated table with explicit foreign key relationships. Authentication records, warm-up schedules, placement tests, reputation scores, and DMARC reports each occupy their own tables with strict referential integrity. The schema is designed so that any cross-entity query (e.g., "show me all domains whose DMARC policy is p=none and whose warm-up is stalled") can be answered with standard SQL joins rather than JSONB extraction or event replay.

The approach mirrors how platforms like Validity Everest and MXToolbox structure their internal data: each ISP, each authentication check, each placement test result is a first-class row in a relational table. Reference data tables for ISP providers, bounce classifications (following SendGrid's seven-class taxonomy), and DNS record types ensure consistency across the platform.

This is the safest choice for teams with strong SQL experience who prioritise query flexibility and data integrity over write throughput. It produces the most tables but also the most predictable query patterns.

**Best for:** Teams building a compliance-heavy platform where complex cross-entity reporting, authentication auditing, and multi-dimensional filtering are primary use cases.

**Trade-offs:**
- Pro: Maximum query flexibility with standard SQL joins
- Pro: Strong referential integrity prevents orphaned records
- Pro: Well-understood by most development teams
- Pro: Straightforward indexing strategy
- Con: Higher table count (~35-40 tables) increases migration complexity
- Con: Schema changes require ALTER TABLE migrations for every new field
- Con: Write-heavy event ingestion (webhook events, DMARC reports) may bottleneck on foreign key checks
- Con: ISP-specific or provider-specific metadata requires new columns or junction tables rather than flexible storage

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 7208 (SPF) | `spf_records` table stores parsed SPF record fields: mechanisms, qualifiers, includes, IP ranges |
| RFC 6376 (DKIM) | `dkim_records` table stores selector, key length, canonicalisation, and public key hash |
| RFC 7489 (DMARC) | `dmarc_policies` table mirrors the DMARC DNS record fields (p, sp, adkim, aspf, pct, rua, ruf); `dmarc_aggregate_reports` and `dmarc_report_rows` tables map the RUA XML schema |
| DMARCbis (draft) | `dmarc_policies` includes columns for np, psd, t tags for forward compatibility |
| RFC 8058 (One-Click Unsubscribe) | `unsubscribe_compliance` table tracks List-Unsubscribe header presence and POST endpoint validity per domain |
| RFC 8617 (ARC) | `arc_chain_results` table stores ARC-Authentication-Results, ARC-Seal, and ARC-Message-Signature validation outcomes |
| BIMI (IETF Draft) | `bimi_records` table stores SVG logo URL, certificate type (VMC/CMC), certificate chain status, and DMARC readiness |
| ISO 3166 | `jurisdictions` reference table for sender/recipient geography; used in compliance filtering |
| SendGrid Event Taxonomy | `email_event_types` reference table aligns with the standard event types: processed, delivered, bounced, deferred, dropped, open, click, spam_report, unsubscribe |
| CAN-SPAM / GDPR / CCPA | `compliance_checks` table tracks per-domain compliance with unsubscribe mechanisms, physical address presence, consent basis |

---

## Core Identity & Multi-Tenancy

```sql
-- Tenants: top-level accounts (companies, agencies, individuals)
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, starter, growth, enterprise
    max_domains     INT NOT NULL DEFAULT 5,
    max_mailboxes   INT NOT NULL DEFAULT 10,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tenants_slug ON tenants (slug);

-- Users within a tenant
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,  -- RFC 5321 max email length
    name            VARCHAR(255),
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, member, viewer
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_email ON users (email);

-- API keys for programmatic access
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    key_hash        VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{}',  -- e.g., {read:domains, write:warmup}
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_keys_tenant ON api_keys (tenant_id);
```

## Domain & Mailbox Management

```sql
-- Domains registered for monitoring
CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    domain_name     VARCHAR(253) NOT NULL,  -- RFC 1035 max domain length
    status          VARCHAR(50) NOT NULL DEFAULT 'pending_verification',
    -- pending_verification, verified, suspended, archived
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, domain_name)
);
CREATE INDEX idx_domains_tenant ON domains (tenant_id);
CREATE INDEX idx_domains_name ON domains (domain_name);

-- Mailboxes (email accounts) connected to the platform
CREATE TABLE mailboxes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email_address   VARCHAR(320) NOT NULL UNIQUE,
    provider        VARCHAR(50) NOT NULL,  -- gmail, outlook, yahoo, custom_smtp
    display_name    VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    -- active, paused, warming_up, suspended, disconnected
    smtp_host       VARCHAR(253),
    smtp_port       INT,
    imap_host       VARCHAR(253),
    imap_port       INT,
    auth_method     VARCHAR(50) NOT NULL DEFAULT 'oauth2',  -- oauth2, app_password, smtp_credentials
    oauth_token_enc TEXT,  -- encrypted OAuth refresh token
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_mailboxes_domain ON mailboxes (domain_id);
CREATE INDEX idx_mailboxes_tenant ON mailboxes (tenant_id);
CREATE INDEX idx_mailboxes_provider ON mailboxes (provider);
```

## Authentication Records (SPF / DKIM / DMARC / BIMI)

```sql
-- SPF records for a domain
CREATE TABLE spf_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    raw_record      TEXT NOT NULL,           -- full TXT record value
    version         VARCHAR(10) NOT NULL,    -- v=spf1
    mechanisms      TEXT[] NOT NULL,          -- e.g., {include:_spf.google.com, ip4:203.0.113.0/24}
    qualifier       VARCHAR(10) NOT NULL DEFAULT '-all',  -- -all, ~all, ?all, +all
    dns_lookup_count INT NOT NULL DEFAULT 0, -- SPF has 10-lookup limit
    is_valid        BOOLEAN NOT NULL DEFAULT false,
    validation_errors TEXT[],
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_spf_domain ON spf_records (domain_id);

-- DKIM records for a domain
CREATE TABLE dkim_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    selector        VARCHAR(255) NOT NULL,   -- e.g., google, s1, s2
    key_type        VARCHAR(10) NOT NULL DEFAULT 'rsa',  -- rsa, ed25519
    key_length      INT,                     -- 1024, 2048, etc.
    public_key_hash VARCHAR(64),             -- SHA-256 of public key for change detection
    canonicalisation VARCHAR(20) NOT NULL DEFAULT 'relaxed/relaxed',
    is_valid        BOOLEAN NOT NULL DEFAULT false,
    validation_errors TEXT[],
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (domain_id, selector)
);
CREATE INDEX idx_dkim_domain ON dkim_records (domain_id);

-- DMARC policies for a domain
CREATE TABLE dmarc_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    raw_record      TEXT NOT NULL,
    version         VARCHAR(10) NOT NULL DEFAULT 'DMARC1',
    policy          VARCHAR(20) NOT NULL,    -- none, quarantine, reject
    subdomain_policy VARCHAR(20),            -- sp tag
    adkim           VARCHAR(10) NOT NULL DEFAULT 'r',  -- r=relaxed, s=strict
    aspf            VARCHAR(10) NOT NULL DEFAULT 'r',
    pct             INT DEFAULT 100,         -- percentage (being removed in DMARCbis)
    rua             TEXT[],                  -- aggregate report URIs
    ruf             TEXT[],                  -- forensic report URIs
    fo              VARCHAR(10) DEFAULT '0', -- failure reporting options
    -- DMARCbis forward-compatibility fields
    np              VARCHAR(20),             -- non-existent subdomain policy
    psd             BOOLEAN,                 -- public suffix domain flag
    t               BOOLEAN,                 -- testing flag
    is_valid        BOOLEAN NOT NULL DEFAULT false,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_dmarc_domain ON dmarc_policies (domain_id);

-- BIMI records for a domain
CREATE TABLE bimi_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    raw_record      TEXT,
    logo_url        TEXT,                    -- SVG Tiny P/S logo URL
    certificate_url TEXT,                    -- VMC or CMC authority URL
    certificate_type VARCHAR(10),            -- vmc, cmc
    certificate_valid BOOLEAN DEFAULT false,
    dmarc_ready     BOOLEAN DEFAULT false,   -- requires p=quarantine or p=reject
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bimi_domain ON bimi_records (domain_id);

-- Historical authentication check results (point-in-time snapshots)
CREATE TABLE auth_check_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    check_type      VARCHAR(20) NOT NULL,    -- spf, dkim, dmarc, bimi, ptr
    passed          BOOLEAN NOT NULL,
    details         TEXT,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_auth_history_domain ON auth_check_history (domain_id);
CREATE INDEX idx_auth_history_checked ON auth_check_history (checked_at);
```

## DMARC Aggregate Report Ingestion

```sql
-- Ingested DMARC aggregate reports (RUA)
-- Maps the <report_metadata> and <policy_published> sections of RFC 7489 XML
CREATE TABLE dmarc_aggregate_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id           UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    report_id           VARCHAR(255) NOT NULL,   -- from <report_id> in XML
    reporting_org       VARCHAR(255) NOT NULL,   -- e.g., google.com, yahoo.com
    reporting_email     VARCHAR(320),
    date_range_begin    TIMESTAMPTZ NOT NULL,
    date_range_end      TIMESTAMPTZ NOT NULL,
    -- Policy published at time of report
    policy_domain       VARCHAR(253) NOT NULL,
    policy_p            VARCHAR(20) NOT NULL,
    policy_sp           VARCHAR(20),
    policy_adkim        VARCHAR(10),
    policy_aspf         VARCHAR(10),
    policy_pct          INT,
    raw_xml             TEXT,                    -- original XML for reference
    ingested_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (domain_id, report_id, reporting_org)
);
CREATE INDEX idx_dmarc_reports_domain ON dmarc_aggregate_reports (domain_id);
CREATE INDEX idx_dmarc_reports_date ON dmarc_aggregate_reports (date_range_begin, date_range_end);
CREATE INDEX idx_dmarc_reports_org ON dmarc_aggregate_reports (reporting_org);

-- Individual rows within a DMARC aggregate report
-- Maps the <record> section of RFC 7489 XML
CREATE TABLE dmarc_report_rows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id           UUID NOT NULL REFERENCES dmarc_aggregate_reports(id) ON DELETE CASCADE,
    source_ip           INET NOT NULL,
    message_count       INT NOT NULL,
    disposition         VARCHAR(20) NOT NULL,    -- none, quarantine, reject
    dkim_result         VARCHAR(20) NOT NULL,    -- pass, fail, none
    spf_result          VARCHAR(20) NOT NULL,    -- pass, fail, none
    dkim_domain         VARCHAR(253),
    dkim_selector       VARCHAR(255),
    spf_domain          VARCHAR(253),
    spf_scope           VARCHAR(20),             -- mfrom, helo
    envelope_from       VARCHAR(253),
    header_from         VARCHAR(253),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_dmarc_rows_report ON dmarc_report_rows (report_id);
CREATE INDEX idx_dmarc_rows_source_ip ON dmarc_report_rows (source_ip);
CREATE INDEX idx_dmarc_rows_disposition ON dmarc_report_rows (disposition);
```

## Warm-Up Engine

```sql
-- Warm-up schedules for mailboxes
CREATE TABLE warmup_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mailbox_id      UUID NOT NULL REFERENCES mailboxes(id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    -- active, paused, completed, failed
    strategy        VARCHAR(50) NOT NULL DEFAULT 'adaptive',  -- adaptive, linear, aggressive
    daily_target    INT NOT NULL DEFAULT 30,          -- current daily email target
    daily_max       INT NOT NULL DEFAULT 200,         -- maximum daily volume
    ramp_increment  INT NOT NULL DEFAULT 5,           -- emails to add per day
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    paused_at       TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_warmup_mailbox ON warmup_schedules (mailbox_id);
CREATE INDEX idx_warmup_status ON warmup_schedules (status);

-- Daily warm-up activity log
CREATE TABLE warmup_daily_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id     UUID NOT NULL REFERENCES warmup_schedules(id) ON DELETE CASCADE,
    log_date        DATE NOT NULL,
    emails_sent     INT NOT NULL DEFAULT 0,
    emails_received INT NOT NULL DEFAULT 0,
    replies_sent    INT NOT NULL DEFAULT 0,
    marked_important INT NOT NULL DEFAULT 0,
    marked_not_spam INT NOT NULL DEFAULT 0,
    landed_inbox    INT NOT NULL DEFAULT 0,
    landed_spam     INT NOT NULL DEFAULT 0,
    landed_promotions INT NOT NULL DEFAULT 0,
    bounce_count    INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (schedule_id, log_date)
);
CREATE INDEX idx_warmup_log_schedule ON warmup_daily_log (schedule_id);
CREATE INDEX idx_warmup_log_date ON warmup_daily_log (log_date);

-- Warm-up network peers (seed inboxes participating in warm-up)
CREATE TABLE warmup_peers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_address   VARCHAR(320) NOT NULL UNIQUE,
    provider        VARCHAR(50) NOT NULL,   -- gmail, outlook, yahoo, aol
    region          VARCHAR(10),            -- ISO 3166-1 alpha-2
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_active_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_warmup_peers_provider ON warmup_peers (provider);
CREATE INDEX idx_warmup_peers_active ON warmup_peers (is_active);
```

## Inbox Placement Testing

```sql
-- Inbox placement test runs
CREATE TABLE placement_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    mailbox_id      UUID REFERENCES mailboxes(id),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- pending, sending, collecting, completed, failed
    subject_line    TEXT,
    content_hash    VARCHAR(64),            -- SHA-256 of test email body
    seed_count      INT NOT NULL DEFAULT 0, -- number of seed inboxes targeted
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_placement_tests_domain ON placement_tests (domain_id);
CREATE INDEX idx_placement_tests_tenant ON placement_tests (tenant_id);
CREATE INDEX idx_placement_tests_status ON placement_tests (status);

-- Individual seed inbox results within a placement test
CREATE TABLE placement_test_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    test_id         UUID NOT NULL REFERENCES placement_tests(id) ON DELETE CASCADE,
    seed_email      VARCHAR(320) NOT NULL,
    provider        VARCHAR(50) NOT NULL,    -- gmail, outlook, yahoo, aol, etc.
    placement       VARCHAR(50) NOT NULL,    -- inbox, spam, promotions, social, missing
    received_at     TIMESTAMPTZ,
    delivery_time_ms INT,                    -- milliseconds from send to receipt
    spam_score      DECIMAL(5,2),            -- SpamAssassin-style score
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_placement_results_test ON placement_test_results (test_id);
CREATE INDEX idx_placement_results_provider ON placement_test_results (provider);
CREATE INDEX idx_placement_results_placement ON placement_test_results (placement);

-- Spam filter analysis results
CREATE TABLE spam_filter_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    test_id         UUID NOT NULL REFERENCES placement_tests(id) ON DELETE CASCADE,
    filter_engine   VARCHAR(50) NOT NULL,    -- spamassassin, barracuda, google, microsoft
    score           DECIMAL(5,2) NOT NULL,
    threshold       DECIMAL(5,2),
    passed          BOOLEAN NOT NULL,
    rules_triggered TEXT[],                  -- e.g., {BAYES_50, HTML_MESSAGE, URIBL_BLOCKED}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_spam_filter_test ON spam_filter_results (test_id);
```

## Email Event Tracking

```sql
-- Reference table for email event types (aligned with SendGrid taxonomy)
CREATE TABLE email_event_types (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(50) NOT NULL UNIQUE,
    category        VARCHAR(50) NOT NULL,    -- delivery, engagement, account
    description     TEXT
);
-- Seed data: processed, delivered, bounced, deferred, dropped,
-- open, click, spam_report, unsubscribe, group_unsubscribe, group_resubscribe

-- Email events ingested from ESP webhooks
CREATE TABLE email_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    mailbox_id      UUID REFERENCES mailboxes(id),
    domain_id       UUID NOT NULL REFERENCES domains(id),
    event_type_id   INT NOT NULL REFERENCES email_event_types(id),
    recipient_email VARCHAR(320) NOT NULL,
    sg_event_id     VARCHAR(255),            -- external event ID for idempotency
    sg_message_id   VARCHAR(255),            -- external message ID
    timestamp       TIMESTAMPTZ NOT NULL,
    -- Bounce-specific fields
    bounce_class    VARCHAR(50),             -- hard_bounce, soft_bounce, block
    bounce_reason   TEXT,
    smtp_status     VARCHAR(10),             -- e.g., 550, 421
    -- Engagement-specific fields
    url_clicked     TEXT,
    user_agent      TEXT,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_email_events_tenant ON email_events (tenant_id);
CREATE INDEX idx_email_events_domain ON email_events (domain_id);
CREATE INDEX idx_email_events_mailbox ON email_events (mailbox_id);
CREATE INDEX idx_email_events_type ON email_events (event_type_id);
CREATE INDEX idx_email_events_timestamp ON email_events (timestamp);
CREATE INDEX idx_email_events_recipient ON email_events (recipient_email);
CREATE INDEX idx_email_events_sg_event ON email_events (sg_event_id);

-- Bounce classification reference (aligned with SendGrid 7-class taxonomy)
CREATE TABLE bounce_classifications (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT
);
-- Seed: invalid_address, technical, content, reputation,
-- frequency_volume, mailbox_unavailable, unclassified
```

## Reputation Scoring

```sql
-- Domain reputation scores (computed daily)
CREATE TABLE domain_reputation_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    score_date      DATE NOT NULL,
    overall_score   DECIMAL(5,2) NOT NULL,   -- 0.00 to 100.00
    inbox_rate      DECIMAL(5,2),            -- percentage of emails landing in inbox
    spam_rate       DECIMAL(5,2),
    bounce_rate     DECIMAL(5,2),
    complaint_rate  DECIMAL(5,4),            -- must stay below 0.003 (0.3%)
    open_rate       DECIMAL(5,2),
    click_rate      DECIMAL(5,2),
    unsubscribe_rate DECIMAL(5,4),
    emails_sent     INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (domain_id, score_date)
);
CREATE INDEX idx_reputation_domain ON domain_reputation_scores (domain_id);
CREATE INDEX idx_reputation_date ON domain_reputation_scores (score_date);

-- IP reputation tracking
CREATE TABLE ip_reputation_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ip_address      INET NOT NULL,
    score_date      DATE NOT NULL,
    overall_score   DECIMAL(5,2) NOT NULL,
    listed_on_blacklists TEXT[],
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (ip_address, score_date)
);
CREATE INDEX idx_ip_reputation_ip ON ip_reputation_scores (ip_address);
CREATE INDEX idx_ip_reputation_date ON ip_reputation_scores (score_date);

-- Blacklist monitoring
CREATE TABLE blacklist_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID REFERENCES domains(id),
    ip_address      INET,
    blacklist_name  VARCHAR(255) NOT NULL,   -- e.g., Spamhaus SBL, Barracuda BRBL
    is_listed       BOOLEAN NOT NULL,
    listing_reason  TEXT,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_blacklist_domain ON blacklist_checks (domain_id);
CREATE INDEX idx_blacklist_ip ON blacklist_checks (ip_address);
CREATE INDEX idx_blacklist_listed ON blacklist_checks (is_listed) WHERE is_listed = true;
```

## Content Analysis & Compliance

```sql
-- Spam content analysis results
CREATE TABLE content_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    subject_line    TEXT,
    body_hash       VARCHAR(64) NOT NULL,    -- SHA-256 of body content
    overall_risk    VARCHAR(20) NOT NULL,     -- low, medium, high, critical
    spam_score      DECIMAL(5,2),
    trigger_words   TEXT[],
    link_count      INT,
    image_count     INT,
    has_tracking_pixel BOOLEAN,
    html_valid      BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_content_tenant ON content_analyses (tenant_id);
CREATE INDEX idx_content_risk ON content_analyses (overall_risk);

-- Compliance checks per domain
CREATE TABLE compliance_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    check_type      VARCHAR(50) NOT NULL,    -- can_spam, gdpr, ccpa, one_click_unsubscribe
    passed          BOOLEAN NOT NULL,
    details         TEXT,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_compliance_domain ON compliance_checks (domain_id);

-- Unsubscribe compliance tracking (RFC 8058)
CREATE TABLE unsubscribe_compliance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    has_list_unsubscribe BOOLEAN NOT NULL DEFAULT false,
    has_list_unsubscribe_post BOOLEAN NOT NULL DEFAULT false,
    post_endpoint_url TEXT,
    post_endpoint_valid BOOLEAN,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_unsub_domain ON unsubscribe_compliance (domain_id);
```

## Alerts & Notifications

```sql
-- Alert rules configured by tenants
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    condition_type  VARCHAR(100) NOT NULL,
    -- e.g., reputation_drop, blacklist_added, auth_failure, bounce_rate_threshold
    threshold_value DECIMAL(10,4),
    domain_id       UUID REFERENCES domains(id),  -- NULL = all domains
    is_active       BOOLEAN NOT NULL DEFAULT true,
    notification_channels TEXT[] NOT NULL DEFAULT '{email}',  -- email, slack, webhook
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_rules_tenant ON alert_rules (tenant_id);

-- Fired alert instances
CREATE TABLE alert_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    domain_id       UUID REFERENCES domains(id),
    severity        VARCHAR(20) NOT NULL,    -- info, warning, critical
    message         TEXT NOT NULL,
    acknowledged    BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    fired_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_instances_tenant ON alert_instances (tenant_id);
CREATE INDEX idx_alert_instances_fired ON alert_instances (fired_at);
CREATE INDEX idx_alert_instances_ack ON alert_instances (acknowledged) WHERE acknowledged = false;
```

## Reference Data

```sql
-- ISP/mailbox provider reference data
CREATE TABLE isp_providers (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(100) NOT NULL UNIQUE,  -- Gmail, Outlook, Yahoo, AOL
    domain          VARCHAR(253),
    market_share    DECIMAL(5,2),
    supports_bimi   BOOLEAN DEFAULT false,
    supports_arc    BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Jurisdictions reference (ISO 3166-1)
CREATE TABLE jurisdictions (
    code            VARCHAR(2) PRIMARY KEY,  -- ISO 3166-1 alpha-2
    name            VARCHAR(255) NOT NULL,
    region          VARCHAR(100),
    applicable_regulations TEXT[]             -- e.g., {gdpr, ccpa, can_spam}
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | tenants, users, api_keys |
| Domain & Mailbox | 2 | domains, mailboxes |
| Authentication Records | 5 | spf_records, dkim_records, dmarc_policies, bimi_records, auth_check_history |
| DMARC Report Ingestion | 2 | dmarc_aggregate_reports, dmarc_report_rows |
| Warm-Up Engine | 3 | warmup_schedules, warmup_daily_log, warmup_peers |
| Inbox Placement Testing | 3 | placement_tests, placement_test_results, spam_filter_results |
| Email Event Tracking | 3 | email_events, email_event_types, bounce_classifications |
| Reputation Scoring | 3 | domain_reputation_scores, ip_reputation_scores, blacklist_checks |
| Content & Compliance | 3 | content_analyses, compliance_checks, unsubscribe_compliance |
| Alerts & Notifications | 2 | alert_rules, alert_instances |
| Reference Data | 2 | isp_providers, jurisdictions |
| **Total** | **31** | |

---

## Key Design Decisions

1. **Separate tables for each authentication protocol (SPF, DKIM, DMARC, BIMI)** rather than a generic `auth_records` table. Each protocol has fundamentally different fields (SPF has mechanisms/qualifiers, DKIM has selectors/keys, DMARC has policy/alignment modes). Normalising these into separate tables prevents NULL-heavy rows and enables protocol-specific constraints.

2. **DMARC aggregate report ingestion mirrors the RFC 7489 XML schema** with a parent `dmarc_aggregate_reports` table (report metadata + policy published) and child `dmarc_report_rows` table (per-source-IP authentication results). This allows standard SQL aggregation over DMARC data without XML parsing at query time.

3. **DMARCbis forward-compatibility columns** (np, psd, t) are included in `dmarc_policies` as nullable columns. This avoids a schema migration when the draft becomes a formal RFC.

4. **Email events use a reference table for event types** aligned with SendGrid's established taxonomy (processed, delivered, bounced, deferred, dropped, open, click, spam_report, unsubscribe). This standardises event classification across multiple ESP webhook sources.

5. **Bounce classification follows SendGrid's seven-class model** (invalid_address, technical, content, reputation, frequency_volume, mailbox_unavailable, unclassified) as a reference table, providing consistent categorisation regardless of the originating ESP's raw SMTP response.

6. **Daily reputation scores are pre-computed and stored** rather than calculated on-the-fly. The `domain_reputation_scores` table stores one row per domain per day, enabling fast time-series queries for trend dashboards without scanning the full email_events table.

7. **Warm-up daily logs are separate from warm-up schedules** to maintain a clean separation between configuration (the schedule) and observability (what actually happened each day). The `UNIQUE (schedule_id, log_date)` constraint ensures exactly one summary row per day per schedule.

8. **Placement test results store per-seed-inbox outcomes** rather than just aggregates. This enables ISP-level reporting (e.g., "Gmail placed 95% in Primary, Outlook placed 80% in Inbox") matching the granularity offered by GlockApps and Validity Everest.

9. **`email_events` is the highest-volume table** and will likely need partitioning by `timestamp` (e.g., monthly range partitions) in production. The schema is partition-ready: the timestamp index is the primary time-range filter.

10. **Multi-tenancy is row-level** with `tenant_id` foreign keys on all major tables. This is the simplest multi-tenant pattern and works well up to thousands of tenants. Schema-per-tenant would only be needed at extreme scale or for data residency requirements.
