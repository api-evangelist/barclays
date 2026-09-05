---
name: barclays-open-data-lookup
description: Read Barclays ATM locations, branch locations, published product terms and FCA service metrics from the OBIE open-data APIs.
api: Barclays ATM Locator (2.2), Branch Locator (2.2), Product Details (2.4), FCA Service Metrics (v1)
generated: '2026-09-04'
method: generated
source: openapi/barclays-atm-locator-openapi.yml, openapi/barclays-branch-locator-openapi.yml, openapi/barclays-product-details-openapi.yml, openapi/barclays-fca-service-metrics-openapi.yml
operations:
  - getAtms
  - getBranches
  - get_personal-current-accounts
  - get_business-current-accounts
  - get_commercial-credit-cards
  - get_unsecured-sme-loans
  - get__fca-service-metrics_pca
  - get__fca-service-metrics_bca
---

# Read Barclays open data

These four APIs are the CMA/OBIE open-data surface. They carry no consent model and no
PSU. They are the only Barclays APIs an agent can plan against without an authorised TPP
role — but note that Barclays publishes no anonymous base URL for them either, so you still
onboard through the API Exchange before calling.

## What is there

- `getAtms` — `GET /atms`. Barclays ATM estate with services and opening hours.
  `HEAD /atms` is also declared if you only want to check freshness.
- `getBranches` — `GET /branches`. Branch estate with addresses and hours.
- Product Details serves four product classes, each as its own path:
  `GET /personal-current-accounts`, `GET /business-current-accounts`,
  `GET /commercial-credit-cards`, `GET /unsecured-sme-loans`.
- FCA Service Metrics serves the FCA-mandated service-quality figures:
  `GET /fca-service-metrics/pca` (personal current accounts) and
  `GET /fca-service-metrics/bca` (business current accounts).

## Conventions that only apply here

- These are the only Barclays APIs that support conditional requests. Send
  `If-None-Match` (or `If-Modified-Since`) and honour the `Etag` and `Cache-Control`
  response headers — the datasets change slowly and a 304 costs you nothing.
- Responses use the OBIE open-data media types
  `application/prs.openbanking.opendata.v2.2+json` and
  `application/prs.openbanking.opendata.v1.0+json`. Send a matching `Accept` header or you
  will get a `406`.
- There is no pagination envelope on these APIs and no idempotency concern: they are all
  reads.
