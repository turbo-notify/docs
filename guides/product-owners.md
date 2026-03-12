# Product Owners Guide

> Business context and technical constraints for Product Owners.

---

## What is Turbo Notify?

Turbo Notify is a **B2B SaaS platform** that provides WhatsApp Web session infrastructure to other businesses.

### Target Customers

| Segment | Use Case |
|---------|----------|
| **SaaS Companies** | Transactional notifications (orders, payments, appointments) |
| **AI Startups** | WhatsApp chatbots and assistants |
| **ERPs** | Customer communication automation |
| **Support Systems** | Help desk integrations |

### Value Proposition

**For customers who don't want to use the official WhatsApp Business API:**

| Official API | Turbo Notify |
|--------------|--------------|
| High cost per message | Flat monthly fee |
| Template approval required | Direct messaging |
| Long onboarding process | Quick API integration |
| Limited flexibility | Full control |

---

## How It Works

### High-Level Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Tenant's   │────▶│    Turbo     │────▶│   WhatsApp   │
│   Software   │ API │    Notify    │     │   Web        │
└──────────────┘     └──────────────┘     └──────────────┘
                              │
                              ▼
                     ┌──────────────┐
                     │   Tenant's   │
                     │   Webhook    │
                     └──────────────┘
```

1. **Tenant** sends API request to Turbo Notify
2. **Turbo Notify** delivers message via WhatsApp Web
3. **WhatsApp** delivers to recipient
4. **Events** (delivered, read) sent back to Tenant via webhook

### Core Entities

| Entity | Description |
|--------|-------------|
| **Tenant** | Customer organization (the SaaS, ERP, etc.) |
| **Session** | A WhatsApp number connected to the platform |
| **Message** | Text, image, document sent/received |
| **Webhook** | URL where we deliver events |

---

## Business Constraints

### What We Can Promise

- ✅ Reliable infrastructure for WhatsApp sessions
- ✅ Multiple session management per tenant
- ✅ Real-time event delivery via webhooks
- ✅ Message status tracking (sent, delivered, read)
- ✅ Media support (images, documents, audio, video)

### What We Cannot Promise

- ❌ "Your number will never be blocked"
  - WhatsApp can block any number at any time
  - We minimize risk but cannot eliminate it

- ❌ Official WhatsApp support
  - This is an unofficial integration
  - No SLA from WhatsApp

### Risk Positioning

**Correct positioning:**
> "Infrastructure that reduces risk and increases stability for WhatsApp sessions"

**Wrong positioning:**
> "Alternative to official API with guaranteed uptime"

---

## Product Features

### Session Management

| Feature | Description |
|---------|-------------|
| Create session | Connect a new WhatsApp number |
| QR authentication | Scan QR code to authenticate |
| Session status | Monitor connection health |
| Auto-reconnect | Automatic recovery on disconnection |

### Messaging

| Feature | Description |
|---------|-------------|
| Send text | Plain text messages |
| Send media | Images, documents, audio, video |
| Message status | Track sent/delivered/read |
| Receive messages | Incoming messages via webhook |

### Webhooks

| Feature | Description |
|---------|-------------|
| Configure URL | Set delivery endpoint |
| Event filtering | Choose which events to receive |
| Retry logic | Automatic retry on failure |
| Secret verification | HMAC signature for security |

### Dashboard

| Feature | Description |
|---------|-------------|
| Session overview | See all connected numbers |
| Message history | View sent/received messages |
| Usage metrics | Messages sent, delivery rates |
| Billing | Current plan, usage, invoices |

---

## Pricing Model

### Suggested Tiers

| Plan | Sessions | Messages/mo | Price |
|------|----------|-------------|-------|
| **Starter** | 1 | 1,000 | $X |
| **Pro** | 5 | 10,000 | $Y |
| **Business** | 20 | 50,000 | $Z |
| **Enterprise** | Unlimited | Custom | Contact |

### Billing Metrics

- **Sessions**: Number of connected WhatsApp numbers
- **Messages**: Outbound messages sent
- **Media**: Storage for media files (optional)

---

## Technical Constraints

### Session Limitations

| Constraint | Reason |
|------------|--------|
| 1 device per session | WhatsApp Web limitation |
| Re-scan needed periodically | Session expiration |
| Rate limits | Avoid WhatsApp blocks |

### Message Limitations

| Constraint | Value |
|------------|-------|
| Max message size | 4,096 characters |
| Max media size | 16 MB (images), 100 MB (documents) |
| Rate limit | ~60 messages/minute per session |

### Reliability Considerations

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Session disconnects | Brief message delay | Auto-reconnect |
| Worker failure | Sessions reallocated | Orchestrator |
| WhatsApp server issue | No control | Retry + monitoring |
| Number blocked | Session lost | Tenant notification |

---

## Competitive Landscape

### Direct Competitors

| Competitor | Positioning |
|------------|-------------|
| **Baileys-based** | Open source, DIY |
| **WAHA** | Self-hosted alternative |
| **Evolution API** | Similar managed service |
| **WPPConnect** | Brazilian market focus |

### Our Differentiation

- **Managed infrastructure** (vs. self-hosted)
- **Reliability focus** (vs. just features)
- **SaaS-friendly** (multi-tenant, webhooks)
- **Developer experience** (clean API, docs)

---

## Roadmap Considerations

### Phase 1 - MVP

Focus: Prove the core value works

- [ ] Session management (create, connect, disconnect)
- [ ] Text messaging
- [ ] Basic webhooks
- [ ] Simple dashboard

### Phase 2 - Production

Focus: Make it reliable for paying customers

- [ ] Media messaging
- [ ] Advanced webhooks (retry, filtering)
- [ ] Usage analytics
- [ ] Billing integration

### Phase 3 - Scale

Focus: Support larger customers

- [ ] Multiple sessions per tenant
- [ ] Connection strategies (always-on, scheduled)
- [ ] Team management
- [ ] API rate limiting per tenant

### Future Considerations

- WhatsApp Business API support (hybrid)
- Telegram/SMS channels
- Message templates
- Analytics dashboard

---

## Metrics to Track

### Business Metrics

| Metric | Description |
|--------|-------------|
| **MRR** | Monthly Recurring Revenue |
| **Churn** | Customer cancellation rate |
| **LTV** | Lifetime Value per customer |
| **CAC** | Customer Acquisition Cost |

### Product Metrics

| Metric | Description |
|--------|-------------|
| **Sessions Active** | Connected sessions count |
| **Messages/day** | Daily message volume |
| **Delivery Rate** | % messages delivered |
| **Uptime** | Session availability |

### Technical Health

| Metric | Description |
|--------|-------------|
| **Reconnect Rate** | Session stability |
| **Message Latency** | Time to deliver |
| **Webhook Success** | % webhooks delivered |
| **Error Rate** | API error percentage |

---

## Glossary

| Term | Definition |
|------|------------|
| **Session** | A WhatsApp Web connection |
| **Tenant** | A customer organization |
| **Worker** | Process managing sessions |
| **Webhook** | HTTP callback for events |
| **QR Auth** | QR code authentication |

For complete terminology, see [Glossary](../reference/glossary.md).

---

## Related Documentation

- [README](../README.md) - Platform overview
- [Architecture](../architecture/ecosystem-architecture.md) - Technical design
- [Glossary](../reference/glossary.md) - Terms and definitions
