# ADR: SEO Adjustments for Correct Indexing

**Date:** 2025-09-19
**Status:** Accepted
**Context:** Google Search Console issues

---

## Problem

Google Search Console reported that static assets (`/_next/static/**`), favicons, and the dashboard (`https://dashboard.turbonotify.com`) were being crawled from the landing page sitemap.

Additionally, valid pages appeared with inconsistent canonicals (without trailing `/`) relative to the redirects forced by Next.js `trailingSlash` configuration, generating statuses like "Crawled, not indexed" and "Page with redirect".

## Decisions

1. **Restrict static assets in `robots.txt`**
   - Block crawling of `_next/static`, `/static`, and favicon directory
   - Keep only relevant routes for indexing

2. **Sanitize sitemap**
   - Remove external URLs (dashboard/login)
   - Normalize paths with trailing `/` to avoid unnecessary redirects

3. **Standardize canonicals**
   - Apply helper to ensure trailing `/` on all published routes
   - Include language variants (landing, pricing, faq, earn, docs, legal pages)

## Consequences

- Google stops wasting crawl budget on static files and dashboard resources
- Pages that should rank now expose canonicals aligned with actual served URLs
- Sitemap restricted to indexable routes, reducing noise in validation cycles

## References

- Screenshots of the problem: `docs/decision-history/2025-09-18-expected-seo-result.png`
