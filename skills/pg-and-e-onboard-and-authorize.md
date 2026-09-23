---
name: pg-and-e-onboard-and-authorize
description: >-
  Register as a PG&E Share My Data third party, obtain both OAuth token classes over mutual TLS, and
  read what a customer actually authorized out of the ESPI function-block scope string.
api: PG&E Share My Data (NAESB ESPI 1.1 / Green Button Connect My Data)
generated: '2026-09-17'
method: generated
source: >-
  https://www.pge.com/en/save-energy-and-money/energy-saving-programs/smartmeter/third-party-companies.html
operations:
  - GET /GreenButtonConnect/espi/1_1/resource/Authorization
  - GET /GreenButtonConnect/espi/1_1/resource/Authorization/{AuthorizationID}
  - GET /GreenButtonConnect/espi/1_1/resource/ApplicationInformation/{ApplicationInformationID}
  - DELETE /GreenButtonConnect/espi/1_1/resource/Authorization/{AuthorizationID}
  - POST /datacustodian/oauth/v2/token
---

# Onboard to PG&E Share My Data and authorize a customer

Operation paths are quoted from PG&E's published supported-API reference
(`Supported-APIs.pdf`). The in-repo OpenAPI is a partial scaffold and is not the source of truth for
this skill.

## 0. You cannot skip registration

Share My Data is approval-gated. Before any call works you need, per PG&E's own registration page:

- a 9-digit U.S. Employer Identification Number (EIN)
- business and technical contacts
- a notification URI PG&E can POST to
- **a TLS 1.2 X.509 certificate from a recognized SSL provider. Self-signed is rejected**, and
  submitting one delays approval
- eligible standing with the California Public Utilities Commission

Register at <https://sharemydata.pge.com/>. Questions go to ShareMyData@pge.com.

## 1. Every request carries your client certificate

Mutual TLS is enforced at the gateway, including on the token endpoint. Without the certificate
you get, verbatim:

```
HTTP/1.1 400
"Invalid Certificate"
```

That is a *transport* failure. No bearer token will clear it. If you see it, your TLS client config
is wrong — stop and fix that before debugging OAuth.

## 2. Get a third-party token (client_credentials)

```
POST https://api.pge.com/datacustodian/oauth/v2/token
grant_type=client_credentials
```

This yields the **client_access_token**. Use it for your own resources: the `Authorization` feed,
`ApplicationInformation`, `ReadServiceStatus`, and every `Batch/Bulk/...` resource.

Errors here are RFC 6749 objects. A `GET` on this endpoint returns
`405 {"error":"invalid_request","error_description":"GET not permitted"}`; a request with no
`grant_type` returns `400 {"error":"invalid_request","error_description":"Missing grant_type"}`.

## 3. Get a customer token (authorization_code)

Send the customer to `https://api.pge.com/datacustodian/oauth/v2/Authorize` with your `client_id`.
They authenticate on PG&E's site and pick what to share. Exchange the returned code at the token
endpoint for a per-customer **access_token** plus a refresh token.

Rehearse this against the test environment first —
`https://api.pge.com/datacustodian/test/oauth/v2/authorize` and `.../test/oauth/v2/token`. Both
answer anonymously with parameter-validation errors, so you can check request shape before you hold
a certificate.

## 4. Read the scope — it is a function-block string, not scope names

The `scope` returned with the token looks like:

```
FB=1_3_8_13_14_18_19_31_32_35_37_38_39_40_4_5_10_15_16_46_47;AdditionalScope=Usage_Billing_Basic_Account_ProgramEnrollment;IntervalDuration=900_3600;BlockDuration=Daily;HistoryLength={...};AccountCollection={...};BR={ThirdPartyID};dataCustodianId=PGE
```

Parse the `FB=` list. The blocks that tell you what the customer actually chose are the
customer-selectable ones: **4** (interval usage), **5** (electric interval), **10** (gas), **15**
(usage summary), **16** (usage summary with cost), **46** and **47** (retail customer info). The
rest are always returned and tell you nothing about consent. Full mapping in
`scopes/pg-and-e-scopes.yml`.

Also read `IntervalDuration` — it tells you whether you were granted 15-minute data, hourly, or
both. Do not assume 15-minute granularity.

## 5. Enumerate what you hold

```
GET /GreenButtonConnect/espi/1_1/resource/Authorization                    (client_access_token)
GET /GreenButtonConnect/espi/1_1/resource/Authorization/{AuthorizationID}  (client_access_token)
```

Responses are ESPI Atom XML, not JSON. **`AuthorizationID`, `SubscriptionID` and `RetailCustomerID`
are the same value** at the customer level — do not go looking for a mapping.

## 6. Revocation is one-way

```
DELETE /GreenButtonConnect/espi/1_1/resource/Authorization/{AuthorizationID}  (client_access_token)
```

This is the only write operation on the API. **There is no API call that restores it.** The customer
must grant a new authorization through the OAuth flow, producing a new `AuthorizationID`. PG&E
documents no undo window. Treat it as irreversible and confirm with a human first.

## Limits

1 request/second per vendor across all your Client IDs; 2,000 calls/hour and 20,000 calls/24h per
Client ID, the daily counter resetting at 5 p.m. PT. **No `RateLimit-*` or `Retry-After` headers are
returned** — you must count for yourself.
