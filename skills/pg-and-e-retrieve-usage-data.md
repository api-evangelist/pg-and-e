---
name: pg-and-e-retrieve-usage-data
description: >-
  Retrieve authorized PG&E electric and gas interval usage, meter readings and billing summaries
  from Share My Data, including the asynchronous bulk/notification path for many customers at once.
api: PG&E Share My Data (NAESB ESPI 1.1 / Green Button Connect My Data)
generated: '2026-09-17'
method: generated
source: >-
  https://www.pge.com/assets/pge/docs/save-energy-and-money/energy-savings-programs/Supported-APIs.pdf
operations:
  - GET /GreenButtonConnect/espi/1_1/resource/Subscription/{SubscriptionID}/UsagePoint
  - GET /GreenButtonConnect/espi/1_1/resource/Subscription/{SubscriptionID}/UsagePoint/{UsagePointID}
  - GET /GreenButtonConnect/espi/1_1/resource/Subscription/{SubscriptionID}/UsagePoint/{UsagePointID}/MeterReading
  - GET /GreenButtonConnect/espi/1_1/resource/Subscription/{SubscriptionID}/UsagePoint/{UsagePointID}/UsageSummary
  - GET /GreenButtonConnect/espi/1_1/resource/ReadingType/{ReadingTypeID}
  - GET /GreenButtonConnect/espi/1_1/resource/LocalTimeParameters
  - GET /GreenButtonConnect/espi/1_1/resource/Batch/Subscription/{SubscriptionID}
  - GET /GreenButtonConnect/espi/1_1/resource/Batch/Bulk/{BulkID}
  - GET /GreenButtonConnect/espi/1_1/resource/Batch/RetailCustomer/{RetailCustomerID}
---

# Retrieve usage data from PG&E Share My Data

Prerequisite: `pg-and-e-onboard-and-authorize`. You need a client certificate, a customer
`access_token`, and the `SubscriptionID` (= `AuthorizationID`).

Everything below returns **ESPI 1.1 Atom XML**, validated against PG&E's published XSD bundle. There
is no JSON representation.

## One customer, interactively

```
GET .../resource/Subscription/{SubscriptionID}/UsagePoint                         access_token
GET .../resource/Subscription/{SubscriptionID}/UsagePoint/{UsagePointID}          access_token
GET .../resource/Subscription/{SubscriptionID}/UsagePoint/{UsagePointID}/MeterReading
GET .../resource/Subscription/{SubscriptionID}/UsagePoint/{UsagePointID}/UsageSummary
```

A `UsagePoint` is one service agreement — one electric or one gas meter. A customer with both
commodities has at least two, so never assume a single usage point.

`UsageSummary` gives billing-period totals, and **dollar cost only if the customer granted Bill Info
(function block 16)**. Check the scope string before promising a customer their bill amount.

## Narrow the window

Function block 37 is always granted, so these query parameters are always available on the resources
that accept them:

- `published-min`, `published-max`
- `updated-min`, `updated-max`

All are Zulu timestamps. Use `updated-min` for daily deltas — that is what it is for. Note that many
resources are documented as "request parameters ignored"; parameters apply to the meter-reading and
usage-summary reads.

## Interpret the numbers correctly

- `GET .../resource/ReadingType/{ReadingTypeID}` gives units, multiplier and commodity. **Read it.**
  An interval value is meaningless without it.
- `GET .../resource/LocalTimeParameters` gives the local time offset. PG&E states it publishes a
  single LocalTimeParameters ID.
- PG&E defines one `IntervalBlock` per day. A month of 15-minute data is ~30 blocks, not one.

## Many customers: use bulk, not a loop

Iterating your whole customer base through the per-customer resources will hit the 1 request/second
per-vendor ceiling immediately. The asynchronous path exists for exactly this:

1. PG&E POSTs an **ESPI Notification** to the notification URI you registered (Green Button function
   block 39, push model) when data is ready.
2. You then GET the bulk resource with your `client_access_token`:

```
GET .../resource/Batch/Bulk/{BulkID}                       client_access_token
GET .../resource/Batch/BulkRetailCustomerInfo/{BulkID}     client_access_token
GET .../resource/Batch/Subscription/{SubscriptionID}       access_token
```

The notification is **not signed and carries no delivery id**. Verify by fetching the bulk resource
over your own mutual-TLS connection rather than trusting the POST, and deduplicate on your side.

## Failure modes, in the order you will meet them

| What you see | What it means |
|---|---|
| `400 "Invalid Certificate"` | Client certificate missing/invalid. Transport, not auth. |
| `405 {"error":"invalid_request","error_description":"GET not permitted"}` | You GET the token endpoint. POST it. |
| `400 {"error":"invalid_request","error_description":"Missing grant_type"}` | Token request has no grant_type. |
| `404 No listener for endpoint: /path` | Path not routed. ESPI resources are under `/GreenButtonConnect`; OAuth endpoints are not. |
| Empty feed | The customer did not authorize that function block, or the window has no data. Re-read the scope string. |

## Limits

1 req/s per vendor; 2,000/hour and 20,000/24h per Client ID (daily reset 5 p.m. PT). No rate-limit
headers are returned. Batch daily rather than polling.
