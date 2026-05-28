# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Email Deliverability Platform · Created: 2026-05-19

## Philosophy

This model uses a relational backbone for core entities (tenants, domains, mailboxes) and high-frequency query paths, but stores variable, provider-specific, and evolving data in JSONB columns. The insight is that email deliverability spans many ISPs, authentication protocols, and ESP integrations, each with subtly different data shapes. Gmail's feedback loop fields differ from Outlook's. A DMARC report from Google has nuances that Yahoo's does not. Warm-up strategies vary by provider. Rather than creating dozens of nullable columns or junction tables for every variation, the hybrid approach stores the stable, queryable core relationally and the variable, provider-specific detail in indexed JSONB.

This pattern is used by platforms like Stripe (relational IDs and amounts, JSONB metadata), Shopify (relational products, JSONB metafields), and modern SaaS tools that need to ship quickly while accommodating domain complexity. PostgreSQL's JSONB support — with GIN indexes, containment operators, and jsonpath queries — makes this practical without sacrificing query performance on the structured fields.

The result is fewer tables than a fully normalised model, faster time to MVP, and built-in flexibility for multi-provider and multi-jurisdiction variation. The trade-off is that JSONB fields lack the constraint enforcement of typed columns, requiring application-level validation.

**Best for:** Teams prioritising rapid development, multi-provider integration flexibility, and the ability to accommodate ISP-specific and jurisdiction-specific data variations without constant schema migrations.

**Trade-offs:**
- Pro: Fewer tables (~20) and simpler migrations than fully normalised
- Pro: New provider-specific fields require no schema changes — just add to the JSONB
- Pro: JSONB GIN indexes enable fast containment and existence queries
- Pro: Natural fit for storing raw ESP webhook payloads alongside parsed fields
- Pro: Easy to extend for new authentication protocols or compliance frameworks
- Con: JSONB fields lack database-level constraints (NOT NULL, UNIQUE, FK)
- Con: Application must validate JSONB structure — bugs can create inconsistent data
- Con: JSONB queries with deep nesting can be slower than indexed relational columns
- Con: Harder to enforce referential integrity for data inside JSONB
- Con: Reporting queries that filter on JSONB fields require GIN or expression indexes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 7208 (SPF) | Parsed SPF fields stored in `auth_records.details` JSONB with type='spf'; raw record in relational column |
| RFC 6376 (DKIM) | DKIM selector, key length, canonicalisation stored in `auth_records.details` JSONB with type='dkim' |
| RFC 7489 (DMARC) | DMARC policy fields in `auth_records.details` JSONB; full RUA XML in `dmarc_reports.raw_data`; report rows in `dmarc_reports.rows` JSONB array |
| DMARCbis (draft) | New tags (np, psd, t) added to the JSONB payload without schema migration |
| RFC 8058 (One-Click Unsubscribe) | Compliance fields stored in `domains.compliance_status` JSONB |
| RFC 8617 (ARC) | ARC validation stored as auth_records entry with type='arc' and chain details in JSONB |
| BIMI (IETF Draft) | BIMI status stored in `auth_records.details` JSONB with type='bimi' |
| SendGrid Event Taxonomy | Raw webhook payload preserved in `email_events.raw_payload` JSONB; parsed fields in relational columns |
| ISO 3166 | Jurisdiction codes in `tenants.settings` JSONB and `domains.settings` JSONB |
| CAN-SPAM / GDPR / CCPA | Per-domain compliance status in `domains.compliance_status` JSONB with regulation-specific fields |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "max_domains": 5,
    --   "max_mailboxes": 10,
    --   "timezone": "America/New_York",
    --   "jurisdiction": "US",
    --   "notification_preferences": {
    --     "email": true, "slack_webhook": "https://hooks.slack.com/...", "daily_digest": true
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tenants_slug ON tenants (slug);
CREATE INDEX idx_tenants_settings ON tenants USING GIN (settings);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    name            VARCHAR(255),
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "https://...",
    --   "notification_channels": ["email", "slack"],
    --   "last_viewed_domain": "uuid-here"
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
CREATE INDEX idx_users_tenant ON users (tenant_id);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    key_hash        VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    scopes          JSONB NOT NULL DEFAULT '[]',
    -- scopes example: ["read:domains", "write:warmup", "read:reports"]
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_keys_tenant ON api_keys (tenant_id);
```

## Domains & Mailboxes

```sql
CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    domain_name     VARCHAR(253) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending_verification',
    verified_at     TIMESTAMPTZ,
    -- Aggregated authentication health (updated by background workers)
    auth_health     JSONB NOT NULL DEFAULT '{}',
    -- auth_health example:
    -- {
    --   "spf": {"valid": true, "last_checked": "2026-05-19T10:00:00Z", "qualifier": "~all", "lookup_count": 3},
    --   "dkim": {"valid": true, "last_checked": "2026-05-19T10:00:00Z", "selectors": ["google", "s1"], "key_length": 2048},
    --   "dmarc": {"valid": true, "last_checked": "2026-05-19T10:00:00Z", "policy": "quarantine", "adkim": "r", "aspf": "r"},
    --   "bimi": {"valid": false, "last_checked": "2026-05-19T10:00:00Z", "reason": "DMARC not at reject"},
    --   "arc": {"supported": true},
    --   "ptr": {"valid": true, "last_checked": "2026-05-19T10:00:00Z"}
    -- }
    compliance_status JSONB NOT NULL DEFAULT '{}',
    -- compliance_status example:
    -- {
    --   "one_click_unsubscribe": {"compliant": true, "post_url": "https://acme.com/unsub", "last_checked": "2026-05-19"},
    --   "can_spam": {"physical_address": true, "unsubscribe_mechanism": true},
    --   "gdpr": {"consent_basis": "legitimate_interest", "suppression_list_active": true},
    --   "ccpa": {"opt_out_available": true}
    -- }
    -- Latest reputation snapshot
    reputation      JSONB NOT NULL DEFAULT '{}',
    -- reputation example:
    -- {
    --   "overall_score": 87.5,
    --   "inbox_rate": 92.1,
    --   "spam_rate": 4.2,
    --   "bounce_rate": 1.8,
    --   "complaint_rate": 0.0012,
    --   "computed_at": "2026-05-19",
    --   "trend": "stable"
    -- }
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "check_interval_hours": 6,
    --   "alert_thresholds": {"reputation_drop": 5, "bounce_rate": 0.05, "complaint_rate": 0.003}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, domain_name)
);
CREATE INDEX idx_domains_tenant ON domains (tenant_id);
CREATE INDEX idx_domains_name ON domains (domain_name);
CREATE INDEX idx_domains_auth_health ON domains USING GIN (auth_health);
CREATE INDEX idx_domains_reputation ON domains USING GIN (reputation);

CREATE TABLE mailboxes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email_address   VARCHAR(320) NOT NULL UNIQUE,
    provider        VARCHAR(50) NOT NULL,    -- gmail, outlook, yahoo, custom_smtp
    display_name    VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    connection      JSONB NOT NULL DEFAULT '{}',
    -- connection example (varies by provider):
    -- Gmail/Outlook (OAuth):
    -- {
    --   "auth_method": "oauth2",
    --   "oauth_token_enc": "encrypted-token-here",
    --   "scopes": ["https://www.googleapis.com/auth/gmail.send"],
    --   "token_expires_at": "2026-05-20T10:00:00Z"
    -- }
    -- Custom SMTP:
    -- {
    --   "auth_method": "smtp_credentials",
    --   "smtp_host": "smtp.acme.com",
    --   "smtp_port": 587,
    --   "imap_host": "imap.acme.com",
    --   "imap_port": 993,
    --   "username_enc": "encrypted-username"
    -- }
    health          JSONB NOT NULL DEFAULT '{}',
    -- health example:
    -- {
    --   "last_synced_at": "2026-05-19T10:00:00Z",
    --   "last_send_at": "2026-05-19T09:45:00Z",
    --   "error_count_24h": 0,
    --   "last_error": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_mailboxes_domain ON mailboxes (domain_id);
CREATE INDEX idx_mailboxes_tenant ON mailboxes (tenant_id);
CREATE INDEX idx_mailboxes_provider ON mailboxes (provider);
```

## Authentication Records (Unified Table with JSONB Details)

```sql
-- Single table for all authentication record types
-- The relational columns handle filtering; JSONB holds protocol-specific details
CREATE TABLE auth_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    record_type     VARCHAR(20) NOT NULL,    -- spf, dkim, dmarc, bimi, arc, ptr
    raw_record      TEXT,                    -- raw DNS TXT record value
    is_valid        BOOLEAN NOT NULL DEFAULT false,
    details         JSONB NOT NULL DEFAULT '{}',
    -- SPF details:
    -- {
    --   "version": "spf1",
    --   "mechanisms": ["include:_spf.google.com", "ip4:203.0.113.0/24"],
    --   "qualifier": "-all",
    --   "dns_lookup_count": 3,
    --   "errors": []
    -- }
    --
    -- DKIM details:
    -- {
    --   "selector": "google",
    --   "key_type": "rsa",
    --   "key_length": 2048,
    --   "canonicalisation": "relaxed/relaxed",
    --   "public_key_hash": "sha256-abc123..."
    -- }
    --
    -- DMARC details:
    -- {
    --   "version": "DMARC1",
    --   "policy": "quarantine",
    --   "subdomain_policy": null,
    --   "adkim": "r",
    --   "aspf": "r",
    --   "pct": 100,
    --   "rua": ["mailto:dmarc@acme.com"],
    --   "ruf": [],
    --   "fo": "0",
    --   "np": null,
    --   "psd": null,
    --   "t": null
    -- }
    --
    -- BIMI details:
    -- {
    --   "logo_url": "https://acme.com/.well-known/bimi/logo.svg",
    --   "certificate_url": "https://...",
    --   "certificate_type": "cmc",
    --   "certificate_valid": true,
    --   "dmarc_ready": true
    -- }
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_auth_records_domain ON auth_records (domain_id);
CREATE INDEX idx_auth_records_type ON auth_records (record_type);
CREATE INDEX idx_auth_records_valid ON auth_records (is_valid);
CREATE INDEX idx_auth_records_details ON auth_records USING GIN (details);
```

## DMARC Reports

```sql
-- DMARC aggregate reports with rows stored as JSONB array
CREATE TABLE dmarc_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    report_id       VARCHAR(255) NOT NULL,
    reporting_org   VARCHAR(255) NOT NULL,
    date_range_begin TIMESTAMPTZ NOT NULL,
    date_range_end  TIMESTAMPTZ NOT NULL,
    -- Policy snapshot at report time
    policy_published JSONB NOT NULL,
    -- policy_published example:
    -- {
    --   "domain": "acme.com",
    --   "p": "quarantine",
    --   "sp": null,
    --   "adkim": "r",
    --   "aspf": "r",
    --   "pct": 100
    -- }
    -- Report rows stored as JSONB array — avoids a separate table
    rows            JSONB NOT NULL DEFAULT '[]',
    -- rows example:
    -- [
    --   {
    --     "source_ip": "203.0.113.5",
    --     "count": 142,
    --     "disposition": "none",
    --     "dkim": {"domain": "acme.com", "result": "pass", "selector": "google"},
    --     "spf": {"domain": "acme.com", "result": "pass", "scope": "mfrom"},
    --     "envelope_from": "acme.com",
    --     "header_from": "acme.com"
    --   },
    --   {
    --     "source_ip": "198.51.100.10",
    --     "count": 3,
    --     "disposition": "quarantine",
    --     "dkim": {"domain": "unknown.com", "result": "fail"},
    --     "spf": {"domain": "unknown.com", "result": "fail", "scope": "mfrom"},
    --     "envelope_from": "unknown.com",
    --     "header_from": "acme.com"
    --   }
    -- ]
    -- Aggregate summary (pre-computed from rows for fast queries)
    summary         JSONB NOT NULL DEFAULT '{}',
    -- summary example:
    -- {
    --   "total_messages": 145,
    --   "pass_count": 142,
    --   "quarantine_count": 3,
    --   "reject_count": 0,
    --   "unique_source_ips": 2,
    --   "unauthorised_ips": ["198.51.100.10"]
    -- }
    raw_xml         TEXT,
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (domain_id, report_id, reporting_org)
);
CREATE INDEX idx_dmarc_reports_domain ON dmarc_reports (domain_id);
CREATE INDEX idx_dmarc_reports_date ON dmarc_reports (date_range_begin, date_range_end);
CREATE INDEX idx_dmarc_reports_org ON dmarc_reports (reporting_org);
CREATE INDEX idx_dmarc_reports_summary ON dmarc_reports USING GIN (summary);
CREATE INDEX idx_dmarc_reports_rows ON dmarc_reports USING GIN (rows);
```

### Example: Query unauthorised senders across all reports

```sql
-- Find all DMARC report rows with SPF failures using JSONB containment
SELECT d.domain_name, r.reporting_org, r.date_range_begin,
       row_elem->>'source_ip' AS source_ip,
       (row_elem->>'count')::int AS message_count,
       row_elem->'spf'->>'result' AS spf_result
FROM dmarc_reports r
JOIN domains d ON d.id = r.domain_id
CROSS JOIN LATERAL jsonb_array_elements(r.rows) AS row_elem
WHERE row_elem->'spf'->>'result' = 'fail'
  AND d.tenant_id = 'tenant-uuid-here'
ORDER BY r.date_range_begin DESC;
```

## Warm-Up Engine

```sql
CREATE TABLE warmup_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mailbox_id      UUID NOT NULL REFERENCES mailboxes(id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "strategy": "adaptive",
    --   "daily_target": 30,
    --   "daily_max": 200,
    --   "ramp_increment": 5,
    --   "provider_distribution": {"gmail": 0.5, "outlook": 0.3, "yahoo": 0.15, "aol": 0.05},
    --   "warm_topics": ["technology", "business"],
    --   "language": "en",
    --   "pause_on_spam_rate": 0.10
    -- }
    -- AI model parameters (adaptive strategy)
    ai_state        JSONB NOT NULL DEFAULT '{}',
    -- ai_state example:
    -- {
    --   "model_version": "warmup-v3",
    --   "engagement_momentum": 0.85,
    --   "recommended_daily": 42,
    --   "risk_score": 0.12,
    --   "last_adjustment": "2026-05-18",
    --   "adjustment_reason": "positive engagement trend"
    -- }
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    paused_at       TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_warmup_mailbox ON warmup_schedules (mailbox_id);
CREATE INDEX idx_warmup_status ON warmup_schedules (status);

-- Daily warm-up log (relational for time-series queries)
CREATE TABLE warmup_daily_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id     UUID NOT NULL REFERENCES warmup_schedules(id) ON DELETE CASCADE,
    log_date        DATE NOT NULL,
    emails_sent     INT NOT NULL DEFAULT 0,
    replies_received INT NOT NULL DEFAULT 0,
    landed_inbox    INT NOT NULL DEFAULT 0,
    landed_spam     INT NOT NULL DEFAULT 0,
    landed_promotions INT NOT NULL DEFAULT 0,
    -- Provider-level breakdown in JSONB
    provider_breakdown JSONB NOT NULL DEFAULT '{}',
    -- provider_breakdown example:
    -- {
    --   "gmail": {"sent": 15, "inbox": 14, "spam": 1, "promotions": 0},
    --   "outlook": {"sent": 10, "inbox": 9, "spam": 0, "promotions": 1},
    --   "yahoo": {"sent": 5, "inbox": 4, "spam": 1, "promotions": 0}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (schedule_id, log_date)
);
CREATE INDEX idx_warmup_log_schedule ON warmup_daily_log (schedule_id);
CREATE INDEX idx_warmup_log_date ON warmup_daily_log (log_date);
```

## Email Events

```sql
-- Email events with relational core fields and JSONB for provider-specific details
CREATE TABLE email_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    domain_id       UUID NOT NULL REFERENCES domains(id),
    mailbox_id      UUID REFERENCES mailboxes(id),
    event_type      VARCHAR(50) NOT NULL,    -- delivered, bounced, deferred, dropped, open, click, spam_report, unsubscribe
    event_category  VARCHAR(20) NOT NULL,    -- delivery, engagement
    recipient_email VARCHAR(320) NOT NULL,
    provider        VARCHAR(50),             -- gmail, outlook, yahoo, etc.
    occurred_at     TIMESTAMPTZ NOT NULL,
    -- Provider-specific and event-specific details in JSONB
    details         JSONB NOT NULL DEFAULT '{}',
    -- Bounce details example:
    -- {
    --   "bounce_type": "hard",
    --   "classification": "invalid_address",
    --   "smtp_status": "550",
    --   "reason": "5.1.1 The email account that you tried to reach does not exist",
    --   "diagnostic_code": "smtp; 550-5.1.1",
    --   "mta_response": "mx.google.com"
    -- }
    --
    -- Open/click details example:
    -- {
    --   "user_agent": "Mozilla/5.0...",
    --   "ip_address": "203.0.113.42",
    --   "url_clicked": "https://acme.com/offer",
    --   "geo": {"country": "US", "region": "CA", "city": "San Francisco"}
    -- }
    -- Raw ESP webhook payload preserved for debugging
    raw_payload     JSONB,
    -- External IDs for idempotency
    external_event_id VARCHAR(255),          -- sg_event_id or equivalent
    external_message_id VARCHAR(255),        -- sg_message_id or equivalent
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core relational indexes for high-frequency queries
CREATE INDEX idx_events_tenant ON email_events (tenant_id);
CREATE INDEX idx_events_domain ON email_events (domain_id);
CREATE INDEX idx_events_type ON email_events (event_type);
CREATE INDEX idx_events_occurred ON email_events (occurred_at);
CREATE INDEX idx_events_recipient ON email_events (recipient_email);
CREATE INDEX idx_events_external ON email_events (external_event_id);
-- GIN index for JSONB detail queries
CREATE INDEX idx_events_details ON email_events USING GIN (details);

-- Partition hint: in production, partition by occurred_at (monthly)
```

## Inbox Placement Testing

```sql
CREATE TABLE placement_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    mailbox_id      UUID REFERENCES mailboxes(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    subject_line    TEXT,
    seed_count      INT NOT NULL DEFAULT 0,
    -- Results stored as JSONB for flexibility across seed providers
    results         JSONB NOT NULL DEFAULT '{}',
    -- results example:
    -- {
    --   "summary": {
    --     "inbox_rate": 0.87,
    --     "spam_rate": 0.08,
    --     "promotions_rate": 0.03,
    --     "missing_rate": 0.02,
    --     "avg_delivery_ms": 4200
    --   },
    --   "by_provider": {
    --     "gmail": {"seeds": 25, "inbox": 23, "spam": 1, "promotions": 1, "missing": 0},
    --     "outlook": {"seeds": 20, "inbox": 16, "spam": 3, "promotions": 0, "missing": 1},
    --     "yahoo": {"seeds": 15, "inbox": 14, "spam": 0, "promotions": 0, "missing": 1}
    --   },
    --   "spam_filters": {
    --     "spamassassin": {"score": 2.1, "threshold": 5.0, "passed": true, "rules": ["HTML_MESSAGE", "MIME_HTML_ONLY"]},
    --     "barracuda": {"score": 1.5, "passed": true},
    --     "google": {"score": null, "passed": true}
    --   },
    --   "seed_results": [
    --     {"email": "seed1@testnet.io", "provider": "gmail", "placement": "inbox", "delivery_ms": 3100, "spam_score": 1.8},
    --     {"email": "seed2@testnet.io", "provider": "outlook", "placement": "spam", "delivery_ms": 5200, "spam_score": 4.2}
    --   ]
    -- }
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_placement_domain ON placement_tests (domain_id);
CREATE INDEX idx_placement_tenant ON placement_tests (tenant_id);
CREATE INDEX idx_placement_status ON placement_tests (status);
CREATE INDEX idx_placement_results ON placement_tests USING GIN (results);
```

## Reputation History

```sql
-- Daily reputation scores (relational for time-series)
CREATE TABLE reputation_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id) ON DELETE CASCADE,
    score_date      DATE NOT NULL,
    overall_score   DECIMAL(5,2) NOT NULL,
    -- Core metrics as relational columns for fast aggregation
    inbox_rate      DECIMAL(5,2),
    bounce_rate     DECIMAL(5,2),
    complaint_rate  DECIMAL(5,4),
    emails_sent     INT NOT NULL DEFAULT 0,
    -- Detailed breakdown in JSONB
    breakdown       JSONB NOT NULL DEFAULT '{}',
    -- breakdown example:
    -- {
    --   "spam_rate": 4.2,
    --   "open_rate": 22.5,
    --   "click_rate": 3.1,
    --   "unsubscribe_rate": 0.08,
    --   "by_provider": {
    --     "gmail": {"inbox_rate": 95.0, "spam_rate": 2.1, "volume": 800},
    --     "outlook": {"inbox_rate": 85.0, "spam_rate": 8.5, "volume": 350},
    --     "yahoo": {"inbox_rate": 91.0, "spam_rate": 5.0, "volume": 100}
    --   },
    --   "blacklists": [],
    --   "ip_scores": {
    --     "203.0.113.5": {"score": 90.2, "blacklists": []},
    --     "198.51.100.10": {"score": 72.1, "blacklists": ["spamhaus_sbl"]}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (domain_id, score_date)
);
CREATE INDEX idx_reputation_domain ON reputation_scores (domain_id);
CREATE INDEX idx_reputation_date ON reputation_scores (score_date);
CREATE INDEX idx_reputation_score ON reputation_scores (overall_score);
```

## Content Analysis

```sql
CREATE TABLE content_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    body_hash       VARCHAR(64) NOT NULL,
    overall_risk    VARCHAR(20) NOT NULL,    -- low, medium, high, critical
    spam_score      DECIMAL(5,2),
    -- Detailed analysis in JSONB
    analysis        JSONB NOT NULL DEFAULT '{}',
    -- analysis example:
    -- {
    --   "trigger_words": ["free", "limited time", "act now"],
    --   "trigger_word_count": 3,
    --   "link_count": 5,
    --   "link_reputation": {"safe": 4, "suspicious": 1, "blocked": 0},
    --   "image_count": 2,
    --   "image_to_text_ratio": 0.3,
    --   "has_tracking_pixel": true,
    --   "html_valid": true,
    --   "html_issues": [],
    --   "subject_line_risk": "low",
    --   "recommendations": [
    --     "Remove 'free' from subject line",
    --     "Reduce link count to under 3",
    --     "Add alt text to images"
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_content_tenant ON content_analyses (tenant_id);
CREATE INDEX idx_content_risk ON content_analyses (overall_risk);
```

## Alerts

```sql
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    domain_id       UUID REFERENCES domains(id),
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL,
    -- config example:
    -- {
    --   "condition": "reputation_drop",
    --   "threshold": 5,
    --   "window_hours": 24,
    --   "severity": "warning",
    --   "channels": ["email", "slack"],
    --   "slack_webhook": "https://hooks.slack.com/...",
    --   "cooldown_hours": 4
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_rules_tenant ON alert_rules (tenant_id);
CREATE INDEX idx_alert_rules_active ON alert_rules (is_active) WHERE is_active = true;

CREATE TABLE alert_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    domain_id       UUID REFERENCES domains(id),
    severity        VARCHAR(20) NOT NULL,
    message         TEXT NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {
    --   "previous_score": 87.5,
    --   "current_score": 79.5,
    --   "drop_amount": 8.0,
    --   "trigger_events": ["340 bounces on 2026-05-14", "Spamhaus SBL listing on 2026-05-15"],
    --   "recommended_actions": ["Review bounce recipients", "Request Spamhaus delisting"]
    -- }
    acknowledged    BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    fired_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_instances_tenant ON alert_instances (tenant_id);
CREATE INDEX idx_alert_instances_unack ON alert_instances (acknowledged) WHERE acknowledged = false;
```

## Blacklist Monitoring

```sql
CREATE TABLE blacklist_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID REFERENCES domains(id),
    ip_address      INET,
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    results         JSONB NOT NULL DEFAULT '{}',
    -- results example:
    -- {
    --   "total_checked": 25,
    --   "listed_on": [
    --     {"name": "Spamhaus SBL", "reason": "Detected sending spam", "listed_since": "2026-05-15"},
    --     {"name": "Barracuda BRBL", "reason": "Poor reputation", "listed_since": "2026-05-16"}
    --   ],
    --   "clean_on": ["SURBL", "SpamCop", "UCE Protect", "..."]
    -- }
    is_listed       BOOLEAN NOT NULL GENERATED ALWAYS AS (
        jsonb_array_length(COALESCE(results->'listed_on', '[]'::jsonb)) > 0
    ) STORED
);
CREATE INDEX idx_blacklist_domain ON blacklist_checks (domain_id);
CREATE INDEX idx_blacklist_ip ON blacklist_checks (ip_address);
CREATE INDEX idx_blacklist_listed ON blacklist_checks (is_listed) WHERE is_listed = true;
CREATE INDEX idx_blacklist_checked ON blacklist_checks (checked_at);
```

## Warm-Up Network Peers

```sql
CREATE TABLE warmup_peers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_address   VARCHAR(320) NOT NULL UNIQUE,
    provider        VARCHAR(50) NOT NULL,
    region          VARCHAR(2),              -- ISO 3166-1 alpha-2
    is_active       BOOLEAN NOT NULL DEFAULT true,
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats example:
    -- {
    --   "total_interactions": 12500,
    --   "last_active_at": "2026-05-19T08:00:00Z",
    --   "avg_response_time_hours": 2.3,
    --   "inbox_delivery_rate": 0.97
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_peers_provider ON warmup_peers (provider);
CREATE INDEX idx_peers_active ON warmup_peers (is_active) WHERE is_active = true;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | tenants, users, api_keys |
| Domains & Mailboxes | 2 | domains (with auth_health, compliance, reputation JSONB), mailboxes (with connection, health JSONB) |
| Authentication Records | 1 | auth_records (unified, type-discriminated with JSONB details) |
| DMARC Reports | 1 | dmarc_reports (rows stored as JSONB array) |
| Warm-Up Engine | 3 | warmup_schedules (with config, ai_state JSONB), warmup_daily_log (with provider_breakdown JSONB), warmup_peers |
| Email Events | 1 | email_events (with details, raw_payload JSONB) |
| Inbox Placement | 1 | placement_tests (with results JSONB containing full breakdown) |
| Reputation | 1 | reputation_scores (with breakdown JSONB) |
| Content Analysis | 1 | content_analyses (with analysis JSONB) |
| Alerts | 2 | alert_rules (with config JSONB), alert_instances (with context JSONB) |
| Blacklist Monitoring | 1 | blacklist_checks (with results JSONB, generated is_listed column) |
| **Total** | **17** | ~45% fewer tables than the normalised model |

---

## Key Design Decisions

1. **Unified `auth_records` table with a `record_type` discriminator** instead of separate tables per protocol. SPF, DKIM, DMARC, BIMI, ARC, and PTR records all share the same relational shell (domain_id, is_valid, checked_at) with protocol-specific fields in JSONB. This eliminates 4 tables compared to the normalised model and makes adding new protocol types (e.g., MTA-STS) trivial.

2. **DMARC report rows stored as a JSONB array** inside the parent report rather than in a separate child table. Most queries are per-report (display a single report) or aggregate (count pass/fail across reports). The `summary` JSONB field pre-computes the aggregates. For cross-report analysis, `jsonb_array_elements()` with LATERAL joins enables row-level queries when needed.

3. **Domain `auth_health` JSONB column** provides a denormalised current-state view of all authentication checks. Background workers update this JSONB whenever a new `auth_records` row is created. Dashboard queries read `auth_health` directly (a single-row fetch) rather than joining across multiple authentication tables.

4. **Mailbox `connection` JSONB** handles provider-specific connection details. OAuth-based providers (Gmail, Outlook) store token references and scopes. SMTP-based providers store host, port, and encrypted credentials. This avoids a provider_configs junction table or dozens of nullable columns.

5. **Placement test results as a single JSONB column** with nested summary, by_provider, spam_filters, and seed_results sections. This matches how the data is consumed in the UI (a single test result page) and avoids three separate tables (test_results, filter_results, seed_results).

6. **Reputation `breakdown` JSONB** stores per-provider metrics and IP-level scores alongside the relational core metrics (overall_score, inbox_rate, bounce_rate, complaint_rate). The relational columns enable fast time-series aggregation; the JSONB enables drill-down without additional tables.

7. **Blacklist `is_listed` as a generated column** derived from the JSONB `results->'listed_on'` array length. This gives a queryable boolean for filtered indexes while keeping the full blacklist detail in JSONB.

8. **`raw_payload` JSONB on email_events** preserves the original ESP webhook payload alongside parsed relational fields. This is invaluable for debugging provider-specific issues without re-requesting webhook data.

9. **GIN indexes on all major JSONB columns** enable containment queries (`@>`), existence queries (`?`), and jsonpath queries. The trade-off is higher write cost for GIN index maintenance, which is acceptable for the query patterns in this domain.

10. **Warm-up `ai_state` JSONB** stores ML model parameters and recommendations separately from configuration. This allows the AI engine to update its state without modifying the user-configured `config` JSONB, maintaining a clean separation between user intent and system-computed state.
