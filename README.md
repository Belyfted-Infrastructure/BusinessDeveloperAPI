# Belyfted Business API Reference

Version: `v1`

This document describes the proposed public Business API built around the
banking capabilities already present in Belyfted. It is a human-readable API
reference and implementation guide. It does not mean that every documented
route is already publicly deployed.

## Contents

- [Environments](#environments)
- [Authentication](#authentication)
- [Request signing](#request-signing)
- [Common headers](#common-headers)
- [Responses and errors](#responses-and-errors)
- [Pagination](#pagination)
- [Idempotency](#idempotency)
- [Customer onboarding](#customer-onboarding)
- [Customer management](#customer-management)
- [File uploads](#file-uploads)
- [Bank accounts](#bank-accounts)
- [Bank-account restrictions](#bank-account-restrictions)
- [Direct-debit mandates](#direct-debit-mandates)
- [Reports and transactions](#reports-and-transactions)
- [Local transfers](#local-transfers)
- [Belyfted transfers](#belyfted-transfers)
- [Recipients](#recipients)
- [Collections](#collections)
- [Webhooks](#webhooks)
- [Scopes](#scopes)
- [HTTP status codes](#http-status-codes)
- [Implementation status](#implementation-status)

## Environments

| Environment | Base URL |
|---|---|
| Test | `https://sandbox-api.belyfted.com/v1` |
| Live | `https://api.belyfted.com/v1` |

Test credentials only work on the sandbox server. Live credentials only work
on the live server. Customers, accounts, transfers, recipients, idempotency
keys, and webhook deliveries are isolated by merchant and environment.

## Authentication

Send the Authentication Profile token in every protected request:

```http
Authorization: Belyfted YOUR_API_TOKEN
```

The Authentication Profile must be:

- Active and approved.
- Unexpired.
- Created for the current API environment.
- Permitted to access the requested endpoint.
- Assigned the scope required by the endpoint.
- Called from an allowed IP address when an IP allowlist is configured.

Public resource authorization is merchant-scoped. A resource belonging to a
different merchant or environment must be returned as not found.

## Request signing

Requests containing a body must include:

```http
DigitalSignature: BASE64_RSA_SHA256_SIGNATURE
```

Generate the signature by signing the exact raw HTTP request body with the
private key corresponding to the public key in the Authentication Profile.
Belyfted verifies the signature with that public key.

Important rules:

- Use RSA with SHA-256.
- Base64-encode the binary signature.
- Sign the final body after JSON serialization.
- Do not reformat JSON after signing it.
- Whitespace, property order, and escaping affect the signature.
- Requests without a body do not require `DigitalSignature`.

## Common headers

```http
Authorization: Belyfted YOUR_API_TOKEN
Accept: application/json
Content-Type: application/json
X-Request-Id: OPTIONAL_UUID
Idempotency-Key: REQUIRED_FOR_MUTATIONS
DigitalSignature: REQUIRED_WHEN_A_BODY_IS_PRESENT
```

Rate-limit response headers should include:

```http
RateLimit-Limit: 100
RateLimit-Remaining: 73
RateLimit-Reset: 1790254800
```

## Responses and errors

### Successful response

```json
{
  "status": "success",
  "request_id": "4493bf08-4042-4b05-811f-1b2e1d392622",
  "data": {}
}
```

### Collection response

```json
{
  "status": "success",
  "request_id": "4493bf08-4042-4b05-811f-1b2e1d392622",
  "data": [],
  "pagination": {
    "has_more": true,
    "next": "opaque-cursor"
  }
}
```

### Error response

```json
{
  "status": "error",
  "request_id": "d21f9e3f-1188-449c-aa78-eb0817dd9e04",
  "errors": [
    {
      "code": "VALIDATION_ERROR",
      "message": "The amount must be greater than zero.",
      "field": "amount.amount",
      "details": {
        "minimum": "0.00"
      }
    }
  ]
}
```

Sensitive values such as token hashes, private keys, identity document
contents, provider credentials, and internal database IDs must never be
returned. Account numbers, document numbers, and tax identifiers should be
masked except where full values are necessary for a specific operation.

## Pagination

Collection endpoints use cursor pagination.

| Parameter | Description |
|---|---|
| `page[size]` | Number of records. Default `25`; maximum `100`. |
| `page[after]` | Opaque cursor returned by the preceding response. |
| `sort` | Sort field. Prefix with `-` for descending order. |

Example:

```http
GET /v1/customers?page[size]=25&page[after]=CURSOR&sort=-created_at
```

## Idempotency

Every mutating request should include a unique key:

```http
Idempotency-Key: 6aa747f1-08fc-4fab-871c-b2dff5827e28
```

Rules:

- Keys are isolated by merchant, environment, method, and route.
- Keys should be retained for at least 48 hours.
- Retrying the same request returns the original response.
- A replayed response includes `Idempotency-Replayed: true`.
- Reusing a key with a different body returns `409`.
- A concurrent request using the same key returns `409`.

## Customer onboarding

### Submit a business KYB application

```http
POST /v1/customer-onboardings/businesses
```

Required scope: `onboarding_api`  
Successful status: `202 Accepted`

```json
{
  "external_reference": "BUSINESS-20001",
  "business_name": "Example Technologies Limited",
  "company_number": "12345678",
  "registration_type": "private_limited_company",
  "date_of_incorporation": "2020-06-15",
  "country_of_incorporation": "GB",
  "company_status": "active",
  "business_description": "Software development and payment technology.",
  "mcc": "5734",
  "sic_code": "62012",
  "business_size": "small",
  "website": "https://example.com",
  "registered_address": {
    "line1": "20 Example Road",
    "line2": null,
    "city": "London",
    "state": "England",
    "postal_code": "EC1A 1BB",
    "country": "GB"
  },
  "contact": {
    "email": "compliance@example.com",
    "phone": "+442012345678"
  },
  "annual_turnover": {
    "amount": "250000.00",
    "currency": "GBP"
  },
  "directors": [
    {
      "roles": ["director", "beneficial_owner"],
      "ownership_percentage": "75.00",
      "first_name": "Grace",
      "last_name": "Johnson",
      "date_of_birth": "1985-08-21",
      "nationality": "GB",
      "email": "grace.johnson@example.com",
      "phone": "+442011112222",
      "address": {
        "line1": "30 Director Avenue",
        "city": "London",
        "state": "England",
        "postal_code": "SW1A 1AA",
        "country": "GB"
      },
      "is_us_citizen": false,
      "tax_residencies": [
        {
          "country": "GB",
          "tax_identification_number": "AB123456C"
        }
      ],
      "identity_document": {
        "file_id": "3dd28cfa-b626-4dcc-a34c-a60a617da150",
        "document_type": "passport",
        "expires_at": "2031-08-21"
      }
    }
  ],
  "company_documents": [
    {
      "file_id": "7835c557-d19d-45fb-91b4-bd45e35ab147",
      "document_type": "certificate_of_incorporation",
      "expires_at": null
    }
  ],
  "purpose_of_account": ["receive_customer_payments", "supplier_payments"],
  "source_of_funds": ["business_revenue"],
  "expected_origin_countries": ["GB", "NG"],
  "attestations": {
    "information_accurate": true,
    "onboarding_checks_authorised": true
  },
  "metadata": {
    "crm_reference": "CRM-BUSINESS-001"
  }
}
```

### Submit an individual KYC application

```http
POST /v1/customer-onboardings/individuals
```

Required scope: `onboarding_api`  
Successful status: `202 Accepted`

Important fields include:

- External reference, first name, optional middle name, and last name.
- Date of birth, nationality, email, and phone.
- Employment status and annual income.
- Residential address.
- Identity document and selfie file identifiers.
- US citizenship and tax residencies.
- Purpose of account and source of funds.
- Expected origin countries and initial funding.
- International and cash-payment expectations.
- Accuracy and onboarding-check attestations.

### Retrieve onboarding information

| Method | Endpoint | Description |
|---|---|---|
| GET | `/customer-onboardings/{onboardingId}` | Get an onboarding application. |
| GET | `/customer-onboardings/{onboardingId}/decision` | Get the current onboarding decision. |

Required scope: `manage_customer_api`

Onboarding statuses:

- `pending`
- `submitted`
- `reviewing`
- `approved`
- `declined`
- `failed`

## Customer management

| Method | Endpoint | Description |
|---|---|---|
| GET | `/customers` | List individual and business customers. |
| GET | `/customers/{customerId}` | Get a customer. |
| GET | `/customers/{customerId}/bank-accounts` | List the customer's accounts. |
| POST | `/customers/{customerId}/bank-accounts` | Open an account for the customer. |

Required scope: `manage_customer_api` for customer reads and
`bank_account_api` for account operations.

Customer filters:

- `type=individual|business`
- `status`
- `search`
- `external_reference`
- `created_from`
- `created_to`

Customer statuses:

- `pending`
- `active`
- `rejected`
- `failed`
- `blocked`
- `suspended`

## File uploads

```http
POST /v1/files
Content-Type: multipart/form-data
```

| Field | Required | Description |
|---|---:|---|
| `purpose` | Yes | Purpose of the uploaded document. |
| `file` | Yes | Document file. |

Supported purposes include:

- `identity_document_front`
- `identity_document_back`
- `selfie`
- `certificate_of_incorporation`
- `proof_of_address`
- `additional_company_document`

```json
{
  "status": "success",
  "request_id": "6d5a49e5-787a-4900-b326-a26d854c81ad",
  "data": {
    "file_id": "13239786-d507-47ee-82db-06bfcc70c663",
    "document_type": "passport",
    "expires_at": "2030-04-12"
  }
}
```

Uploaded documents must be stored privately and accessed only through
merchant-scoped opaque identifiers.

## Bank accounts

### Products

| Method | Endpoint | Description |
|---|---|---|
| GET | `/bank-account-products` | List available products. |
| GET | `/bank-account-products/{productId}` | Get a product. |

Product filters:

- `customer_type`
- `currency`
- `country`

A product describes its customer types, country, currency, local account
details, transfer capabilities, direct-debit support, virtual-account support,
and statement support.

### Open an account

```http
POST /v1/bank-accounts
```

Required scope: `bank_account_api`

```json
{
  "customer_id": "e2e54e54-f1e4-48ef-92de-a591295fbc05",
  "product_id": "gb-global-gbp-business",
  "display_name": "Operating account",
  "metadata": {
    "department": "operations"
  }
}
```

Successful status: `202 Accepted`. Account opening is asynchronous, and the
initial status is normally `opening`.

### Account operations

| Method | Endpoint | Description |
|---|---|---|
| GET | `/bank-accounts` | List accounts. |
| POST | `/bank-accounts` | Open an account with a product. |
| GET | `/bank-accounts/{bankAccountId}` | Get an account. |
| PATCH | `/bank-accounts/{bankAccountId}` | Update mutable properties. |
| DELETE | `/bank-accounts/{bankAccountId}` | Close an account. |
| GET | `/bank-accounts/{bankAccountId}/deposit-details` | Get funding details. |

Bank-account statuses:

- `opening`
- `open`
- `locked`
- `frozen`
- `deactivated`
- `closing`
- `closed`
- `failed`

An account cannot be closed when it has a non-zero, locked, or reserved
balance, or when it has pending transfers or direct-debit payments.

### Deposit details

```http
GET /v1/bank-accounts/{bankAccountId}/deposit-details
```

```json
{
  "status": "success",
  "request_id": "9acac2ae-dda0-49ce-b860-192956022f08",
  "data": {
    "account_holder_name": "Example Technologies Limited",
    "bank_name": "Example Bank",
    "account_number": "12345678",
    "sort_code": "040004",
    "iban": null,
    "bic": null,
    "reference": "BLF-100001",
    "currency": "GBP"
  }
}
```

## Bank-account restrictions

| Method | Endpoint | Description |
|---|---|---|
| GET | `/bank-accounts/{accountId}/restrictions` | List restrictions. |
| POST | `/bank-accounts/{accountId}/restrictions` | Apply a restriction. |
| GET | `/bank-accounts/{accountId}/restrictions/{restrictionId}` | Get a restriction. |
| PATCH | `/bank-accounts/{accountId}/restrictions/{restrictionId}` | Update a restriction. |
| POST | `/bank-accounts/{accountId}/restrictions/{restrictionId}/lift` | Lift a restriction. |

Restriction types:

- `lock`
- `deactivate`
- `freeze`
- `terminate`

```json
{
  "type": "freeze",
  "reason": "Account activity is under compliance review.",
  "metadata": {
    "case_reference": "CASE-1005"
  }
}
```

The implementation should persist restriction history instead of relying only
on the current wallet status.

## Direct-debit mandates

Direct-debit mandate functionality requires a new backing domain
implementation.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/bank-accounts/{accountId}/direct-debit-mandates` | List mandates. |
| GET | `/direct-debit-mandates/{mandateId}` | Get a mandate. |
| PATCH | `/direct-debit-mandates/{mandateId}` | Suspend, resume, or cancel. |
| GET | `/direct-debit-mandates/{mandateId}/payments` | List mandate payments. |

Mandate statuses:

- `pending`
- `active`
- `suspended`
- `cancelled`
- `expired`

```json
{
  "action": "suspend",
  "reason": "Customer requested a temporary suspension."
}
```

## Reports and transactions

### Statements

| Method | Endpoint | Description |
|---|---|---|
| POST | `/bank-accounts/{accountId}/statements` | Generate a statement. |
| GET | `/statements/{statementId}` | Check generation status. |
| GET | `/statements/{statementId}/download` | Download the statement. |

```json
{
  "from": "2026-08-01",
  "to": "2026-08-31",
  "format": "pdf"
}
```

Statement statuses are `processing`, `ready`, and `failed`.

### Analytics

```http
GET /v1/bank-accounts/{bankAccountId}/analytics?from=2026-08-01&to=2026-08-31
```

Analytics include opening balance, closing balance, total credits, total
debits, credit count, and debit count.

### Transactions

| Method | Endpoint | Description |
|---|---|---|
| GET | `/bank-accounts/{accountId}/transactions` | List transactions. |
| GET | `/bank-accounts/{accountId}/transactions/{transactionId}` | Get a transaction. |
| GET | `/bank-accounts/{accountId}/transactions/{transactionId}/receipt` | Download a receipt. |

Use `direction=credit` for inbound transactions and `direction=debit` for
outbound transactions.

## Local transfers

| Method | Endpoint | Description |
|---|---|---|
| POST | `/transfers/local` | Initiate a local transfer. |
| GET | `/transfers/local` | List local transfers. |
| GET | `/transfers/local/{transferId}` | Get a local transfer. |
| GET | `/transfers/local/{transferId}/status` | Track a transfer. |
| GET | `/transfers/local/{transferId}/receipt` | Download the receipt. |
| POST | `/transfers/local/{transferId}/cancel` | Cancel an eligible transfer. |

Required scope: `local_transfer_api`

```json
{
  "bank_account_id": "1c2a702b-58b3-4497-baba-06573f53ee7a",
  "recipient_id": "a37b9d30-0a06-4e26-87fb-ff36035592a8",
  "amount": {
    "amount": "250.00",
    "currency": "GBP"
  },
  "delivery_method": "faster_payments",
  "payment_reference": "INV-10012",
  "source_of_funds": "business_revenue",
  "sending_purpose": "supplier_payment",
  "metadata": {
    "invoice_id": "INV-10012"
  }
}
```

Successful status: `202 Accepted`.

Transfer statuses:

- `pending`
- `processing`
- `held`
- `completed`
- `failed`
- `cancelled`

A completed or failed transfer cannot be cancelled.

## Belyfted transfers

| Method | Endpoint | Description |
|---|---|---|
| POST | `/transfers/belyfted` | Initiate a Belyfted transfer. |
| GET | `/transfers/belyfted` | List Belyfted transfers. |
| GET | `/transfers/belyfted/{transferId}` | Get a transfer. |
| GET | `/transfers/belyfted/{transferId}/status` | Track a transfer. |
| GET | `/transfers/belyfted/{transferId}/receipt` | Download the receipt. |
| POST | `/transfers/belyfted/{transferId}/cancel` | Cancel an eligible transfer. |

Required scope: `belyfted_transfer_api`

```json
{
  "bank_account_id": "1c2a702b-58b3-4497-baba-06573f53ee7a",
  "recipient_id": "0acdc85e-c0bb-4b71-bb99-b89aa83dfa94",
  "amount": {
    "amount": "100.00",
    "currency": "GBP"
  },
  "payment_reference": "REF-10002",
  "source_of_funds": "business_revenue",
  "sending_purpose": "supplier_payment",
  "scheduled_at": null,
  "schedule_frequency": "once"
}
```

## Recipients

| Method | Endpoint | Description |
|---|---|---|
| GET | `/recipient-forms/local` | Get local-recipient fields. |
| GET | `/recipient-forms/belyfted` | Get Belyfted-recipient fields. |
| GET | `/recipients` | List recipients. |
| POST | `/recipients` | Create a recipient. |
| GET | `/recipients/{recipientId}` | Get a recipient. |
| PATCH | `/recipients/{recipientId}` | Update a recipient. |

Required scope: `beneficiary_api`

Filter recipients with `type=local` or `type=belyfted`.

### Create a local recipient

```json
{
  "type": "local",
  "recipient_category": "business",
  "display_name": "Example Supplier",
  "account_name": "Example Supplier Limited",
  "account_number": "12345678",
  "sort_code": "040004",
  "currency": "GBP",
  "country": "GB"
}
```

### Create a Belyfted recipient

```json
{
  "type": "belyfted",
  "belyfted_id": "BLY-100023",
  "display_name": "Example Recipient",
  "currency": "GBP"
}
```

Full account numbers should not be returned after creation. Subsequent
responses should contain masked values.

## Collections

### Confirmation of Payee

```http
POST /v1/confirmation-of-payee
```

Required scope: `beneficiary_api`

```json
{
  "account_name": "Example Supplier Limited",
  "account_number": "12345678",
  "sort_code": "040004",
  "account_type": "business"
}
```

Possible results are `match`, `close_match`, `no_match`, and `not_supported`.

### Reference data

```http
GET /v1/reference-data/{collection}
```

Supported collections:

- `delivery-methods`
- `recipient-types`
- `supported-banks`
- `source-of-funds`
- `sending-purposes`
- `supported-business-countries`
- `supported-individual-countries`
- `business-registration-types`
- `business-mccs`
- `employment-statuses`
- `document-types`
- `tax-residence-jurisdictions`
- `expected-origin-countries`
- `primary-account-purposes`
- `origin-of-wealth`
- `supported-bank-accounts`

Optional filters include `country`, `currency`, `recipient_type`, and
`search`. Public reference-data identifiers should be stable strings, not
internal database IDs.

## Webhooks

### Belyfted public key

```http
GET /v1/webhook/public-key
```

This endpoint returns the Belyfted public key for the current API environment.
The merchant uses it to verify webhook signatures.

### Webhook endpoint management

| Method | Endpoint | Description |
|---|---|---|
| GET | `/webhook-endpoints` | List endpoints. |
| POST | `/webhook-endpoints` | Register an endpoint. |
| GET | `/webhook-endpoints/{endpointId}` | Get an endpoint. |
| PATCH | `/webhook-endpoints/{endpointId}` | Update an endpoint. |
| DELETE | `/webhook-endpoints/{endpointId}` | Remove an endpoint. |
| POST | `/webhook-endpoints/{endpointId}/test` | Send a test event. |

Live webhook URLs must use HTTPS.

```json
{
  "url": "https://merchant.example.com/webhooks/belyfted",
  "description": "Production banking events",
  "subscribed_events": [
    "customer.onboarding.updated",
    "bank_account.opened",
    "bank_account.transaction.credit",
    "transfer.completed",
    "transfer.failed"
  ]
}
```

### Webhook payload and signature

```json
{
  "Type": "transfer.completed",
  "Version": "1.0",
  "Nonce": "27b7866c-abee-4e08-9985-f53110a84cf7",
  "Payload": {
    "id": "30e99d64-b653-48af-bcce-51f719ea9a15",
    "status": "completed"
  }
}
```

Webhook headers:

```http
DigitalSignature: BASE64_SIGNATURE
x-key-id: KEY_IDENTIFIER
x-delivery-attempt: 1
x-retry-reason: OPTIONAL_DESCRIPTION
```

Belyfted signs the exact JSON body with the private key configured for the
current environment.

### Webhook acknowledgement

The merchant must return HTTP `200` with exactly:

```json
{"nonce":"27b7866c-abee-4e08-9985-f53110a84cf7"}
```

The exact acknowledgement body must be signed with the merchant private key.
The signature is returned in the response header:

```http
DigitalSignature: BASE64_SIGNATURE
```

Belyfted verifies this signature using the public key in the merchant's
Authentication Profile.

### Retry policy

A delivery is retried when:

- The connection fails or times out.
- The endpoint returns a non-200 status.
- The acknowledgement body is malformed.
- The acknowledgement signature is missing or invalid.
- The acknowledgement nonce does not match.

Retry every 15 minutes for up to 24 hours. Stop after a valid signed
acknowledgement, or mark the delivery `exhausted` after the final attempt.

### Delivery history and resend

| Method | Endpoint | Description |
|---|---|---|
| GET | `/webhook-deliveries` | List delivery attempts. |
| GET | `/webhook-deliveries/{deliveryId}` | Get delivery details. |
| POST | `/webhooks/resend` | Resend using the webhook idempotency key. |

```json
{
  "webhook_idempotency_key": "WH-20260925-01HXYZ"
}
```

Successful status: `202 Accepted`. A delivery that is already pending returns
`409 Conflict`.

### Recommended events

Customer events:

- `customer.onboarding.submitted`
- `customer.onboarding.updated`
- `customer.onboarding.approved`
- `customer.onboarding.declined`

Bank-account events:

- `bank_account.opening`
- `bank_account.opened`
- `bank_account.updated`
- `bank_account.restricted`
- `bank_account.restriction_lifted`
- `bank_account.closing`
- `bank_account.closed`
- `bank_account.transaction.credit`
- `bank_account.transaction.debit`

Transfer events:

- `transfer.pending`
- `transfer.processing`
- `transfer.held`
- `transfer.completed`
- `transfer.failed`
- `transfer.cancelled`

Direct-debit events:

- `direct_debit.mandate.created`
- `direct_debit.mandate.updated`
- `direct_debit.payment.completed`
- `direct_debit.payment.failed`

## Scopes

| Scope | Capability |
|---|---|
| `onboarding_api` | Submit individual and business onboarding applications. |
| `manage_customer_api` | View customers and onboarding decisions. |
| `bank_account_api` | Manage accounts, restrictions, reports, and transactions. |
| `beneficiary_api` | Manage recipients and perform Confirmation of Payee. |
| `local_transfer_api` | Create and retrieve local transfers. |
| `belyfted_transfer_api` | Create and retrieve Belyfted transfers. |

Production authorization should deny access by default. An Authentication
Profile must contain the exact required scope or the explicit wildcard `*`.
An empty scope list must not mean unrestricted access.

## HTTP status codes

| Status | Meaning |
|---:|---|
| `200` | Request completed successfully. |
| `201` | Resource created. |
| `202` | Asynchronous operation accepted. |
| `204` | Resource removed. |
| `400` | Malformed request. |
| `401` | Authentication or digital-signature failure. |
| `403` | Scope, environment, endpoint, or IP restriction. |
| `404` | Resource not found inside the merchant/environment boundary. |
| `409` | Resource-state or idempotency conflict. |
| `422` | Semantic validation failure. |
| `429` | Authentication Profile rate limit exceeded. |
| `500` | Unexpected internal failure. |
| `502` | Upstream banking provider failure. |
| `503` | Capability temporarily unavailable. |

Common error codes include:

- `AUTHENTICATION_TOKEN_INVALID`
- `AUTHENTICATION_TOKEN_EXPIRED`
- `AUTHENTICATION_PROFILE_REVOKED`
- `AUTHENTICATION_PROFILE_NOT_APPROVED`
- `ENVIRONMENT_MISMATCH`
- `IP_ADDRESS_NOT_ALLOWED`
- `ENDPOINT_NOT_ALLOWED`
- `SCOPE_NOT_GRANTED`
- `DIGITAL_SIGNATURE_MISSING`
- `DIGITAL_SIGNATURE_INVALID`
- `VALIDATION_ERROR`
- `CUSTOMER_NOT_FOUND`
- `ONBOARDING_NOT_FOUND`
- `BANK_ACCOUNT_NOT_FOUND`
- `BANK_ACCOUNT_RESTRICTED`
- `RECIPIENT_NOT_FOUND`
- `TRANSFER_NOT_FOUND`
- `INSUFFICIENT_FUNDS`
- `IDEMPOTENCY_KEY_REQUIRED`
- `IDEMPOTENCY_KEY_REUSED`
- `WEBHOOK_DELIVERY_NOT_FOUND`
- `WEBHOOK_DELIVERY_PENDING`

## Implementation status

| Capability | Current position |
|---|---|
| Individual and business onboarding | Existing workflow; public adapter required. |
| Customer listing and details | Existing capability; public adapter required. |
| Bank-account opening and retrieval | Existing wallet/account capability; normalization required. |
| Bank-account restrictions | Status actions exist; persistent restriction history required. |
| Deposit details | Existing capability. |
| Statements, analytics, and transactions | Existing capability. |
| Local transfers | Existing step workflow; atomic public adapter required. |
| Belyfted transfers | Existing step workflow; atomic public adapter required. |
| Recipients | Existing partial capability; stable get/update API required. |
| Confirmation of Payee | Existing capability. |
| Reference collections | Mostly existing; response normalization required. |
| Direct-debit mandates | New domain capability required. |
| Signed webhook delivery and resend | Existing capability. |
| Multiple public webhook endpoints | Public management adapter required. |

## Implementation boundary

The public API should use dedicated public controllers and application services
that invoke existing domain services. Dashboard controllers, Sanctum sessions,
transaction PIN middleware, UI encryption envelopes, and step-based form state
must not become part of the public contract.

Every request should pass through the following controls in order:

1. Resolve the environment from the hostname.
2. Assign or validate the request ID.
3. Authenticate the Authentication Profile.
4. Confirm that the profile is active and approved.
5. Confirm that its environment matches the hostname.
6. Enforce the IP and endpoint allowlists.
7. Enforce the required API scope.
8. Verify the digital signature when a body is present.
9. Enforce the Authentication Profile rate limit.
10. Reserve the idempotency key for mutations.
11. Validate the request and resolve merchant-scoped resources.
12. Invoke the existing application/domain service.
13. Persist the audit log and idempotent response.
14. Enqueue relevant webhook events.
15. Return the normalized public response.
