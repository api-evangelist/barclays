---
name: barclays-domestic-payment
description: Initiate a single immediate domestic payment from a Barclays account over UK Open Banking, including the funds check and the idempotency rules.
api: Barclays Payment Initiation API (v4.0)
generated: '2026-09-04'
method: generated
source: openapi/barclays-payment-initiation-openapi.yml
operations:
  - CreateDomesticPaymentConsents
  - GetDomesticPaymentConsentsConsentId
  - GetDomesticPaymentConsentsConsentIdFundsConfirmation
  - CreateDomesticPayments
  - GetDomesticPaymentsDomesticPaymentId
  - GetDomesticPaymentsDomesticPaymentIdPaymentDetails
---

# Initiate a domestic payment at Barclays

This is a PISP flow. It moves real money and **Barclays publishes no operation that
reverses an executed payment** — read the reversibility note at the bottom before you
call step 4.

## Steps

1. **Create the payment consent.** `CreateDomesticPaymentConsents` —
   `POST /domestic-payment-consents`, client-credentials token
   (`TPPOAuth2Security`, scope `payments`). Send `x-idempotency-key`: this operation
   declares it and it is the mechanism that stops a retry creating a second consent.
   Set `Data.Initiation.InstructedAmount`, `CreditorAccount` (with a
   `SchemeName` from `OBInternalAccountIdentification4Code` — e.g.
   `UK.OBIE.SortCodeAccountNumber`), and a `LocalInstrument` from
   `OBInternalLocalInstrument1Code` (`UK.OBIE.FPS`, `UK.OBIE.CHAPS`, …) if you need a
   specific rail.
2. **Have the customer authorise it.** Authorization-code flow (`PSUOAuth2Security`,
   scope `payments`) with the `ConsentId` as the intent; the PSU completes SCA.
3. **Check status and funds.**
   `GetDomesticPaymentConsentsConsentId` — `GET /domestic-payment-consents/{consentId}` —
   must return `Status: AUTH`. Then
   `GetDomesticPaymentConsentsConsentIdFundsConfirmation` —
   `GET /domestic-payment-consents/{consentId}/funds-confirmation` — returns
   `FundsAvailableResult.FundsAvailable`. This is the closest thing to a dry run these
   contracts offer: there is no validate-only mode.
4. **Execute.** `CreateDomesticPayments` — `POST /domestic-payments`, referencing the
   `ConsentId`, with `Data.Initiation` byte-identical to the consent. **Send the same
   `x-idempotency-key` on every retry of the same payment.** A retry without it can pay
   twice.
5. **Confirm.** `GetDomesticPaymentsDomesticPaymentId` —
   `GET /domestic-payments/{domesticPaymentId}` — and read `Status` against
   `ExternalPaymentTransactionStatus1Code`: `ACSC`/`ACCC` settled, `ACSP` in progress,
   `PDNG` pending, `RJCT` rejected. `GET /domestic-payments/{id}/payment-details` gives the
   fuller status history.

## Reversibility — read this

`DELETE /domestic-payment-consents/{consentId}` does not exist. The only DELETE on this
family is on the VRP consent. Once step 4 returns and the status reaches `ACSC` or `ACCC`,
Barclays publishes no cancel, refund, void or reverse operation, and no window inside which
one would be accepted. Treat step 4 as final and get the confirmation in step 3 first.

## Idempotency, honestly

`x-idempotency-key` is declared on all 15 write operations of this API and on the 4 VRP
writes — 19 of the 64 mutating operations Barclays publishes. Every other write in the
estate (Barclays Bank Ireland payments, and every Barclaycard US write) declares no replay
protection at all. Barclays publishes no key-retention window, so do not assume how long a
key stays deduplicated.
