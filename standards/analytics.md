# Analytics and SEO Standards

This document defines analytics, monitoring, and SEO configuration for Turbo Notify frontend applications.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GA_MEASUREMENT_ID` | Yes | Google Analytics 4 property ID |
| `CLARITY_PROJECT_ID` | Yes | Microsoft Clarity project ID |
| `SENTRY_DSN` | Yes | Sentry error tracking DSN |
| `SENTRY_ENV` | No | Environment name (production, staging) |
| `SENTRY_TRACES_SAMPLE_RATE` | No | Trace sampling rate (0.0-1.0) |
| `SENTRY_PROFILES_SAMPLE_RATE` | No | Profile sampling rate (0.0-1.0) |
| `NEXT_PUBLIC_SENTRY_DSN` | No | Exposed DSN for client-side tracking |
| `ANALYZE` | No | Enable bundle analysis |

**Example `.env.local`:**
```env
GA_MEASUREMENT_ID=G-XXXXXXXXXX
CLARITY_PROJECT_ID=xxxxxxxxxx
SENTRY_DSN=https://xxx@sentry.io/xxx
```

## Analytics Scripts Injection

### Landing Page

The `AnalyticsScripts` component (`src/shared/components/layout/analytics-scripts.tsx`) automatically loads Google Analytics and Microsoft Clarity when IDs are defined.

### Dashboard

Scripts are injected automatically in `_app.tsx` when environment variables are set.

## Event Tracking

Use the `trackEvent` utility for custom events:

```typescript
import { trackEvent } from '@/features/analytics';

trackEvent('button_clicked', {
  button_name: 'subscribe',
  plan_id: 'pro',
});
```

**Convention:** Use `snake_case` for all event parameters.

## Activation Funnel

Track user activation steps with dedicated utilities:

```typescript
import {
  activationStepStarted,
  activationStepCompleted,
  activationStepError
} from '@/features/analytics';

// User started a step
activationStepStarted('connect_whatsapp');

// Step completed successfully
activationStepCompleted('connect_whatsapp');

// Step failed
activationStepError('connect_whatsapp', 'qr_code_timeout');
```

These functions update GA, Sentry breadcrumbs, and Prometheus metrics.

## Consent Management

Update tracking consent based on user preference:

```typescript
import { updateConsent } from '@/features/analytics';

// User accepted tracking
updateConsent({ analytics: true, marketing: true });

// User declined
updateConsent({ analytics: false, marketing: false });
```

## Prometheus Metrics

Both Landing and Dashboard expose Prometheus metrics:

| Endpoint | Port |
|----------|------|
| Landing | `http://localhost:3010/api/metrics` |
| Dashboard | `http://localhost:3020/api/metrics` |

**Prometheus scrape config:**
```yaml
scrape_configs:
  - job_name: turbonotify-landing
    metrics_path: /api/metrics
    static_configs:
      - targets: ['localhost:3010']

  - job_name: turbonotify-dashboard
    metrics_path: /api/metrics
    static_configs:
      - targets: ['localhost:3020']
```

### Key Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `turbonotify_activation_step_total` | Counter | `step`, `status` | Activation funnel progress |
| `turbonotify_page_views_total` | Counter | `page`, `locale` | Page view counts |
| `turbonotify_api_requests_total` | Counter | `endpoint`, `method`, `status` | API request counts |

**Grafana Dashboard:** Create panels for `turbonotify_activation_step_total` grouped by `step` and `status`.

## SEO Configuration

### Site Sections

Define site structure in `src/shared/lib/site-sections.ts`. These feed:
- Sitemap generation
- `SiteNavigationElement` structured data
- Navigation components

### Robots.txt

The `src/app/robots.ts` file exposes the sitemap at `/robots.txt`, allowing search engines to discover all sections.

### Canonical URLs

All pages must include proper canonical tags with trailing slashes (per ADR 2025-09-19-seo-indexing).

---

## Related Documents

- [Observability Overview](/docs/observability/observability-overview.md)
- [Metrics Reference](/docs/observability/metrics-reference.md)
- [SEO Indexing ADR](/docs/reference/decisions/2025-09-19-seo-indexing.md)
