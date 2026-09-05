---
name: barclays-account-information-access
description: Set up a UK Open Banking account-access consent with Barclays and read a customer's accounts, balances and transactions.
api: Barclays Account and Transactions API (v4.0)
generated: '2026-09-04'
method: generated
source: openapi/barclays-account-and-transactions-openapi.yml
operations:
  - CreateAccountAccessConsents
  - GetAccountAccessConsentsConsentId
  - GetAccounts
  - GetAccountsAccountId
  - GetAccountsAccountIdBalances
  - GetAccountsAccountIdTransactions
  - DeleteAccountAccessConsentsConsentId
---

# Read Barclays accounts, balances and transactions

Barclays implements the UK Open Banking Read/Write API. Nothing here is reachable
anonymously: you need an FCA/eIDAS-authorised AISP role, an Open Banking-issued mTLS
transport certificate, and an access token. The base is
`https://telesto.api.barclays/open-banking/<version>`.

## Before you start

- Register on the Barclays API Exchange (https://drm.developer.barclays.com/s/registration)
  and onboard through the Dynamic Client Registration API (v1.10).
- Every request below carries `Authorization`, and the FAPI headers
  `x-fapi-auth-date`, `x-fapi-customer-ip-address`, `x-fapi-interaction-id` and
  `x-customer-user-agent`. Keep `x-fapi-interaction-id` stable per logical request — it is
  the only correlation identifier Barclays echoes back on this family.

## Steps

1. **Create the consent.** `CreateAccountAccessConsents` — `POST /account-access-consents`
   with the `Permissions` you actually need (`ReadAccountsDetail`, `ReadBalances`,
   `ReadTransactionsDetail`, …) and an `ExpirationDateTime`. Use a client-credentials token
   (`TPPOAuth2Security`, scope `accounts`). You get back a `ConsentId` with status `AWAU`.
2. **Send the customer to authorise.** Redirect the PSU through the authorization-code flow
   (`PSUOAuth2Security`, scope `accounts`) carrying the `ConsentId` as the intent. The PSU
   completes SCA at Barclays.
3. **Confirm the consent is live.** `GetAccountAccessConsentsConsentId` —
   `GET /account-access-consents/{consentId}`. Proceed only when `Status` is `AUTH`.
   `AWAU` means still awaiting authorisation; `RJCT` means the customer declined.
4. **List the accounts.** `GetAccounts` — `GET /accounts` with the PSU-scoped token.
   Read `Data.Account[].AccountId`; every downstream resource hangs off it.
5. **Read balances.** `GetAccountsAccountIdBalances` — `GET /accounts/{accountId}/balances`.
6. **Read transactions.** `GetAccountsAccountIdTransactions` —
   `GET /accounts/{accountId}/transactions`. Page with `Links.Next` until it is absent;
   `Meta.TotalPages` tells you how many pages exist, and `Meta.FirstAvailableDateTime` /
   `Meta.LastAvailableDateTime` bound the history you can ask for. Narrow with
   `fromBookingDateTime` / `toBookingDateTime`.
7. **Revoke when done.** `DeleteAccountAccessConsentsConsentId` —
   `DELETE /account-access-consents/{consentId}`. This is the only reversal this API offers,
   and it stops future access rather than undoing past reads.

## Errors and limits

- Errors come back as `OBErrorResponse1` — `{ Id, Code, Message, Errors[] }`, where each
  `Errors[]` entry is `OBError1 { ErrorCode, Message, Path, Url }`. `ErrorCode` values are
  `UK.OBIE.*` strings. This is not RFC 9457 `application/problem+json`.
- `403` here almost always means the consent does not carry the permission you used, not
  that the token is bad. Re-read the consent before retrying.
- `429` is declared but Barclays publishes no limit, no window and no `Retry-After` header.
  Back off exponentially and treat the number as unknowable in advance.
- These reads have no idempotency key and need none. The writes in
  `barclays-domestic-payment.md` do.
