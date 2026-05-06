# Standards & API Reference

> Project: Email Deliverability Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### IETF / RFC Standards

**RFC 5321 — Simple Mail Transfer Protocol (SMTP)**
- URL: https://datatracker.ietf.org/doc/html/rfc5321
- The foundational protocol defining how email is transmitted between mail servers. Specifies the envelope-level sender (5321.MAILFROM / return-path), the SMTP command sequence, and bounce handling. Understanding RFC 5321 is essential for diagnosing many deliverability failures, particularly those involving the envelope-from alignment required by SPF.

**RFC 5322 — Internet Message Format**
- URL: https://datatracker.ietf.org/doc/html/rfc5322
- Defines the structure and syntax of email messages: headers, body, and MIME encapsulation. Deliverability platforms must parse and validate messages against this standard when checking header anomalies, From: address formatting, and header injection risks.

**RFC 7208 — Sender Policy Framework (SPF)**
- URL: https://datatracker.ietf.org/doc/html/rfc7208
- Specifies how domain owners publish DNS TXT records listing IP addresses authorised to send email on their behalf. SPF is one of the three pillars of email authentication and is required by Google, Yahoo, and Microsoft for bulk senders. An email deliverability platform must be able to parse, validate, and diagnose SPF records and explain alignment failures.

**RFC 6376 — DomainKeys Identified Mail (DKIM) Signatures**
- URL: https://datatracker.ietf.org/doc/html/rfc6376
- Defines cryptographic signing that allows the receiving server to verify that an email was sent by an authorised party and was not altered in transit. DKIM signatures are attached as message headers; platforms must validate key length (minimum 1024-bit, recommended 2048-bit), selector publication in DNS, and header canonicalisation methods (relaxed/simple).

**RFC 7489 — Domain-based Message Authentication, Reporting and Conformance (DMARC)**
- URL: https://datatracker.ietf.org/doc/html/rfc7489
- Builds on SPF and DKIM to allow domain owners to publish policies (none / quarantine / reject) specifying how receivers should handle messages that fail authentication. DMARC also enables aggregate (rua) and forensic (ruf) reporting. This RFC is being superseded by DMARCbis; current implementations must remain compatible with both versions.

**DMARCbis — Draft IETF Standards Track Update to DMARC (draft-ietf-dmarc-dmarcbis)**
- URL: https://datatracker.ietf.org/doc/draft-ietf-dmarc-dmarcbis/
- The forthcoming update (at draft revision 41 as of early 2025) that promotes DMARC from an Informational RFC to a Proposed Standard. Key changes include: removal of the pct, rf, and ri tags; addition of np, psd, and t tags; replacement of the Public Suffix List with a DNS Tree Walk algorithm for organisational domain determination; and clarification of record inheritance rules. Records will continue to use v=DMARC1. Deliverability platforms should track this draft and prepare for the transition.

**RFC 8617 — The Authenticated Received Chain (ARC) Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc8617
- Published as Experimental in 2019. ARC allows intermediate mail handlers (mailing lists, forwarding services) to sign and preserve authentication results as messages pass through multiple hops, solving the DMARC-break problem that arises when forwarded email loses SPF/DKIM alignment. Google has deployed ARC in Gmail. Deliverability platforms that analyse forwarding chains or mailing list scenarios should understand and surface ARC header chains.

**RFC 8058 — Signaling One-Click Functionality for List Email Headers**
- URL: https://datatracker.ietf.org/doc/html/rfc8058
- Defines the List-Unsubscribe-Post header that enables one-click unsubscribe. Required by Google and Yahoo for bulk senders since February 2024 and by Microsoft since May 2025. Deliverability platforms should validate that outbound messages from bulk senders include a compliant List-Unsubscribe header with a functioning HTTPS POST endpoint.

**RFC 8616 — Email Authentication for Internationalized Mail**
- URL: https://datatracker.ietf.org/doc/html/rfc8616
- Extends SPF, DKIM, and DMARC to support internationalised email addresses (Unicode domains and localparts). Relevant for platforms operating in non-ASCII sending environments.

### Emerging & Quasi-Standards

**BIMI — Brand Indicators for Message Identification (IETF Draft)**
- Working Group URL: https://bimigroup.org/ · IETF draft: https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/
- Allows domain owners to display a verified brand logo next to authenticated messages in supporting mail clients (Gmail, Yahoo, Apple Mail, Fastmail). Requires DMARC at quarantine or reject policy. Two certificate types are accepted: Verified Mark Certificates (VMC, requiring trademark validation, issued by DigiCert, Entrust, Sectigo, GlobalSign, SSL.com) and Common Mark Certificates (CMC, accepted by Gmail since September 2024, requiring one year of logo use without trademark). Logo must be in SVG Tiny P/S format. Deliverability platforms should check BIMI DNS record validity, DMARC policy level, and certificate chain.

### Regulatory Frameworks

**CAN-SPAM Act (US, 15 U.S.C. §7701)**
- URL: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- Requires sender identification, accurate subject lines, a physical address, and a functioning unsubscribe mechanism. Opt-out requests must be honoured within 10 business days. FTC maximum civil penalties increased to $53,088 per violation as of January 2025. Email deliverability platforms operating in or for US senders must surface CAN-SPAM compliance gaps.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr.eu/
- Applies to processing of personal data (including email addresses) of EU residents. Requires explicit consent for marketing emails, the right to erasure, and transparent data use disclosure. As of 2025, reliance on "legitimate interest" for cold email outreach to EU individuals is being increasingly restricted. Deliverability platforms that handle subscriber lists must expose unsubscribe rates and suppression list management features to support GDPR compliance.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- Grants California residents the right to opt out of sale of personal data and to request deletion. Overlaps with GDPR in list-hygiene requirements for email platforms.

---

## Similar Products — Developer Documentation & APIs

### SendGrid (Twilio)
- **Description:** Market-leading email delivery platform supporting both transactional and marketing email at scale. Sends from servers closest to recipients with multiple dedicated IPs for volume stability.
- **API Documentation:** https://www.twilio.com/docs/sendgrid/api-reference
- **SDKs/Libraries:** Python, Node.js, Ruby, PHP, Java, Go, C# — https://github.com/sendgrid
- **Developer Guide:** https://docs.sendgrid.com/for-developers/sending-email/api-getting-started
- **OpenAPI Spec:** Available at https://github.com/twilio/sendgrid-oai
- **Standards:** REST/JSON, OpenAPI 3.x, SMTP relay
- **Authentication:** API Key (Bearer token in Authorization header); Basic Authentication no longer accepted

### Mailgun (Sinch)
- **Description:** Developer-first transactional email API with detailed logs, email validation API, and inbox placement features. Offers US and EU regional endpoints.
- **API Documentation:** https://documentation.mailgun.com/docs/mailgun/api-reference/api-overview
- **SDKs/Libraries:** Python, Ruby, PHP, Java, C#, Node.js, Go — https://documentation.mailgun.com/docs/mailgun/sdk/introduction
- **Developer Guide:** https://documentation.mailgun.com/
- **Standards:** REST/JSON, SMTP relay
- **Authentication:** HTTP Basic Auth with API key; HTTPS required

### Postmark
- **Description:** Transactional-email specialist with best-in-class inbox placement (98.7% in independent testing) and sub-10-second delivery. Exposes a free SpamAssassin-based spam-check API.
- **API Documentation:** https://postmarkapp.com/developer/api/overview
- **SDKs/Libraries:** Multiple languages — https://postmarkapp.com/developer
- **Developer Guide:** https://postmarkapp.com/developer
- **Webhooks:** Bounce, delivery, click, spam complaint, open — https://postmarkapp.com/developer/webhooks/webhooks-overview
- **Spam Check API:** https://spamcheck.postmarkapp.com/doc/
- **Standards:** REST/JSON, SMTP relay
- **Authentication:** Server API token in X-Postmark-Server-Token header

### Amazon SES (Simple Email Service)
- **Description:** AWS cloud-based email sending service with built-in Virtual Deliverability Manager (VDM), reputation dashboard, and predictive inbox placement tests. Tightly integrated with CloudWatch for alerting.
- **API Documentation:** https://docs.aws.amazon.com/ses/latest/APIReference-V2/Welcome.html
- **SDKs/Libraries:** AWS SDK for all major languages — https://aws.amazon.com/developer/tools/
- **Developer Guide:** https://docs.aws.amazon.com/ses/latest/dg/
- **Virtual Deliverability Manager:** https://docs.aws.amazon.com/ses/latest/dg/vdm.html
- **Standards:** AWS REST/JSON (SigV4 signed requests), SMTP relay
- **Authentication:** AWS IAM (SigV4); SMTP credentials derived from IAM user

### Mailtrap
- **Description:** Email infrastructure platform combining a sandbox (fake SMTP for development testing) with a production sending API and deliverability tools. API provides spam scoring, HTML checks, BCC tracking, and analytics.
- **API Documentation:** https://docs.mailtrap.io/developers
- **SDKs/Libraries:** Official SDKs for major languages — https://docs.mailtrap.io/
- **Developer Guide:** https://docs.mailtrap.io/getting-started/email-api-smtp
- **Standards:** REST/JSON, SMTP
- **Authentication:** API token in Authorization header

### GlockApps
- **Description:** Dedicated inbox placement testing service that checks deliverability across major ISPs (Gmail, Outlook, Yahoo, etc.) and spam filters. API available on enterprise plans.
- **API Documentation:** https://glockapps.com/api-documentation-v2/
- **Developer Guide:** https://glockapps.com/api-documentation-v2/
- **Standards:** REST/JSON
- **Authentication:** apiKey query parameter (from account settings); GET access from Enterprise plan; POST from Large Enterprise plan only
- **Rate Limits:** Depends on plan tier

### Validity Everest
- **Description:** Enterprise email deliverability platform covering inbox placement testing, authentication monitoring, list hygiene, and reputation tracking. Offers a Postman collection for API exploration.
- **API Documentation:** https://developer.everest.validity.com/
- **Postman Collection:** Available for download from the developer portal
- **Standards:** REST (JSON, XML, CSV, Serialized output)
- **Authentication:** X-API-KEY header (key from account settings)
- **Rate Limits:** 500 requests per minute (increases available on request)

### MXToolbox
- **Description:** DNS and email diagnostic toolset covering blocklist lookups, SPF/DKIM/DMARC validation, MX record checks, and monitoring. REST API exposes the SuperTool lookup engine.
- **API Documentation:** https://mxtoolbox.com/c/products/mxtoolboxapi · https://knowledgebase.mxtoolbox.com/home/api-reference
- **Standards:** REST/JSON
- **Authentication:** API Key (requested through account settings; free tier limited to example.com lookups)
- **Available Methods:** Lookup (blocklist, smtp, mx, dns, spf, dkim, dmarc), Monitor, Monitor Tags, Usage

### MailReach
- **Description:** Email warm-up and deliverability testing platform with a REST API for connecting mailboxes, scheduling warm-up, tracking deliverability signals, and triggering Slack/webhook alerts. Works with Gmail, Outlook, Mailgun, Brevo, Amazon SES, SendGrid, and any SMTP provider.
- **API Documentation:** https://www.mailreach.co/email-warmup-api
- **Standards:** REST/JSON
- **Authentication:** API key
- **Integration:** Zapier, Make (formerly Integromat), direct REST

### Warmy
- **Description:** AI-driven email warm-up platform with an API for programmatic control of warm-up schedules, account management, and deliverability reporting. Designed for SaaS platforms and agencies managing multiple mailboxes.
- **API Documentation:** https://www.warmy.io/product/api
- **Standards:** REST/JSON
- **Authentication:** API key

---

## Notes

**DMARCbis transition:** The DMARCbis draft is in IETF Last Call as of early 2025 and is expected to be published as a formal Proposed Standard during 2025. Deliverability platforms should implement support for new tags (np, psd, t) and the DNS Tree Walk algorithm before the RFC is finalised, as major ISPs may begin honouring the new semantics early. The pct tag (used to roll out policies gradually) is being removed; platforms that currently advise customers to use pct for gradual enforcement will need to update their guidance.

**BIMI certificate evolution:** Google's acceptance of Common Mark Certificates (CMC) since September 2024 substantially lowers the barrier to BIMI adoption by removing the trademark requirement. Deliverability platforms should support validation of both VMC and CMC certificate chains and surface BIMI readiness as part of authentication health checks.

**No open standard for warm-up networks:** Email warm-up (simulated sending to build sender reputation) has no formal standard or RFC. All warm-up tools operate proprietary seed networks. Interoperability between warm-up providers is not possible at the protocol level, making vendor lock-in a structural feature of the market and representing an opportunity for an open-standard approach.

**Spam filtering algorithms:** The SpamAssassin rule set (Apache, Apache-2.0 licence) remains the most widely adopted open-source spam scoring framework. Postmark exposes it via a public API. Other ISP-specific filtering algorithms (Gmail's TensorFlow-based classifiers, Microsoft's SmartScreen) are closed and undocumented; behaviour must be inferred empirically.
