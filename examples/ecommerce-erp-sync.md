# Failure Map Report

**System:** ecommerce-to-ERP Order Sync Integration **Date:** 2026-05-26 **Reviewed by:** failure-map

## Summary

The ecommerce-to-ERP sync carries high fragility in five areas: webhook reliability, status string mapping, refund propagation, inventory reconciliation, and silent failure paths. Most of these failures share a common trait - the sync appears healthy from both endpoints while something is quietly wrong in the middle. Six failure reports are documented below. The dominant theme is that this integration was built to handle the happy path well and the unhappy path poorly. The highest-priority structural fix is a dead-letter queue and alerting layer that catches records that stop moving.

---

## Failure Reports

### 1. Duplicate Orders from Webhook Retry

**Severity:** Critical **Domain:** Automation / Integration **Component:** ecommerce webhook handler, order creation endpoint **Fragility type:** Non-atomic compound operations

#### What happened

During a high-traffic sale event, ecommerce began retrying webhook deliveries for orders the handler had already received and processed. The handler's response was slow under load and returned a timeout before ecommerce received confirmation. ecommerce retried. The ERP created duplicate order records for roughly 340 orders. Fulfillment ran on both records for 60 of them before the duplicates were caught.

#### The change that caused it

Traffic volume increased significantly during a promotional event. The webhook handler had not been load-tested at sale-event volume. Response times degraded, ecommerce's retry window kicked in, and the handler had no idempotency check on incoming order IDs.

#### Why it broke

The order creation endpoint in the ERP was not idempotent. The handler did not check whether an order with the same ecommerce order ID had already been processed before writing a new record. At normal volume, responses were fast enough that retries almost never occurred. The problem was always there.

#### How it was caught

A fulfillment manager noticed duplicate tracking numbers being generated. No automated alert existed for duplicate order IDs in the ERP.

#### Impact

Double shipments on 60 orders. Return processing cost. Customer confusion. Several hours of manual cleanup across fulfillment and finance.

#### Hardening suggestions

1. Make the order creation handler idempotent: check for an existing record by ecommerce order ID before writing a new one. Return a 200 with the existing record if a duplicate is detected.
2. Add a webhook event log and deduplicate on event ID before processing.
3. Add an alert: if duplicate order IDs appear in the ERP within a rolling time window, fire a notification immediately.

---

### 2. Order Status String Mismatch Causes Sync to Stall

**Severity:** High **Domain:** Integration / Automation **Component:** Order status mapping layer between ecommerce and ERP **Fragility type:** Stringly-typed contracts

#### What happened

ecommerce introduced a new fulfillment status value in a platform update. The integration's status mapping table had no entry for the new value. Orders in that status were silently skipped - the sync handler logged no error, and the orders simply never moved to the ERP. Approximately 200 orders sat in limbo for four days before a fulfillment team member noticed orders were missing from the queue.

#### The change that caused it

ecommerce added a new fulfillment status string as part of a routine platform release. The integration was built against the status values that existed at build time and had no fallback behavior for unrecognized values.

#### Why it broke

The status mapping logic used a strict match with no default. An unrecognized status value caused the record to be silently dropped rather than routed to a review queue or flagged for investigation.

#### How it was caught

A fulfillment team member noticed order volume in the ERP seemed lower than expected. Manual comparison of ecommerce and ERP order counts surfaced the gap.

#### Impact

200 orders delayed by four days. Customer service escalations. Expedited shipping costs on time-sensitive orders.

#### Hardening suggestions

1. Add a default route for unrecognized status values: send them to a review queue instead of dropping them silently.
2. Add an alert: any unrecognized status value encountered during sync should fire a notification to the integration owner immediately.
3. Add a daily reconciliation check: compare total order counts between ecommerce and the ERP for the past 24-48 hours and alert on gaps above a defined threshold.

---

### 3. Refund Not Propagated to ERP

**Severity:** High **Domain:** Integration / Business Process **Component:** ecommerce refund webhook handler, ERP credit memo creation **Fragility type:** Silent failure path

#### What happened

A batch of refunds processed in ecommerce over a weekend failed to create corresponding credit memos in the ERP. The refund webhook was firing, but the ERP's endpoint was intermittently returning errors due to a session timeout issue. The handler logged the errors but did not retry or queue for reprocessing. Finance discovered the gap during month-end reconciliation when the ecommerce refund total and the ERP credit memo total diverged.

#### The change that caused it

An ERP update had shortened the default session timeout for API connections. The integration was using a connection pool that wasn't refreshing sessions on the correct interval.

#### Why it broke

The refund handler had no retry logic and no dead-letter queue. Failed webhook events were logged but not reprocessed. The integration's health dashboard showed no errors because the handler caught the exceptions - it just didn't do anything useful with them.

#### How it was caught

Month-end reconciliation. Approximately three weeks after the failures began.

#### Impact

Understated liability in the ERP for the period. Finance had to manually create credit memos for all affected refunds. Audit trail gaps and unnecessary stress during close.

#### Hardening suggestions

1. Add retry logic with exponential backoff to the refund handler. Failed events should retry at least three times before moving to a dead-letter queue.
2. Implement a dead-letter queue for all webhook event types. Any event that exhausts retries should be held for manual review, not silently dropped.
3. Add a daily reconciliation check between ecommerce refund totals and ERP credit memo totals. Alert on gaps above a defined dollar threshold.

---

### 4. Inventory Oversell During Sync Lag

**Severity:** High **Domain:** Integration / Automation **Component:** Inventory sync between ERP and ecommerce **Fragility type:** Non-atomic compound operations / Invisible invariants

#### What happened

During a product launch, inventory sold faster than the sync could propagate inventory decrements back to ecommerce. The sync ran on a 15-minute polling interval. In that window, multiple customers purchased units that were no longer available, resulting in 47 oversold units across three SKUs.

#### The change that caused it

The sync interval had not been reviewed since the integration was first deployed. Launch traffic was significantly higher than typical daily volume. No one adjusted the sync frequency or enabled near-real-time inventory updates for the launch window.

#### Why it broke

The system was designed assuming a polling interval acceptable under normal traffic. No alert fired when inventory moved faster than the sync could follow. ecommerce's inventory levels were not treated as a managed surface that needed adjustment during high-velocity events.

#### How it was caught

Customer orders for out-of-stock items. Support escalations when fulfillment couldn't ship.

#### Impact

47 oversell units. Customer experience damage. Manual order cancellation and re-offer process.

#### Hardening suggestions

1. Switch to webhook-driven inventory updates instead of polling for high-velocity SKUs or during high-traffic events.
2. Add an inventory buffer threshold: set ecommerce inventory levels slightly below actual available stock to create a safety margin for sync lag.
3. Build a pre-launch checklist that includes reviewing and adjusting sync frequency and inventory buffer settings before any promotional or launch event.

---

### 5. Tax Calculation Drift Between Platforms

**Severity:** Medium **Domain:** Integration / Business Process **Component:** Tax calculation logic in ecommerce vs. ERP **Fragility type:** Business rule encoding / Data trust drift

#### What happened

A tax rate update was applied in the ERP's tax configuration but not in ecommerce's tax settings. For six weeks, the two systems calculated sales tax differently for a specific product category in one state. Orders appeared to process normally. The discrepancy was discovered during a sales tax audit, requiring amended filings.

#### The change that caused it

Finance updated the ERP tax configuration to reflect a state rule change. No corresponding update was made to ecommerce because the process assumed ecommerce tax settings were managed separately by a different team.

#### Why it broke

Tax configuration lived in two places with no single source of truth and no reconciliation between them. The systems could diverge silently because no alert monitored for tax calculation differences between matched orders.

#### How it was caught

An external sales tax audit. The discrepancy had been accumulating for six weeks.

#### Impact

Amended tax filings. Audit exposure. Finance team time to investigate and remediate.

#### Hardening suggestions

1. Define a single source of truth for tax configuration and document it explicitly. Either ecommerce or the ERP owns tax rates - not both.
2. Add a post-order reconciliation check: compare the tax amount recorded in ecommerce and the ERP for the same order. Alert on divergence above a defined tolerance.
3. Build a tax configuration change checklist: any change to tax settings in either system triggers a review of the other.

---

### 6. Sync Failure with No Alert and No Queue

**Severity:** Critical **Domain:** Automation / Integration **Component:** Integration middleware, error handling layer **Fragility type:** Silent failure path / Recovery path missing

#### What happened

The integration middleware lost its authentication token to the ERP after a scheduled credential rotation. The rotation was done as part of routine security maintenance. The middleware began failing silently - it could not authenticate, so it stopped processing. No alert fired. No queue built up. Orders continued arriving in ecommerce with no corresponding movement in the ERP for approximately 18 hours, until a fulfillment lead noticed the ERP order queue had gone quiet.

#### The change that caused it

A scheduled credential rotation - routine, well-intentioned - replaced the API credentials the integration was using without updating the middleware configuration. No one knew the middleware was a dependent system.

#### Why it broke

The credential rotation process had no step to update dependent systems. The middleware had no health check that verified ERP connectivity independently of order volume. A quiet period looked the same as a failed state.

#### How it was caught

Human observation. A fulfillment lead noticed no new orders had appeared in the ERP queue for an unusually long time.

#### Impact

18 hours of unfulfilled orders. Customer order status not updated. Expedited fulfillment scramble and a backlog that took hours to clear.

#### Hardening suggestions

1. Add a heartbeat check to the integration middleware: if no orders have been successfully processed in a configurable window - 30 minutes during business hours, for example - fire an alert.
2. Add the integration middleware to the credential rotation checklist as a dependent system. Every rotation must include a middleware credential update and a post-rotation connectivity test before the rotation is marked complete.
3. Implement a dead-letter queue so that all orders arriving during a sync outage are held and reprocessed in sequence when connectivity is restored.

---

## Themes and Recommendations

Two structural issues drive almost every failure here.

The integration has no resilience layer. Four of the six reports follow the same pattern: the happy path works, the unhappy path disappears. There is no dead-letter queue, no retry logic, no heartbeat check, and no alert that fires when the system stops moving. Adding a resilience layer - retry, queue, and heartbeat - would address the majority of the critical risk in this integration at once.

There is no ongoing reconciliation. Three failures would have been caught in hours instead of weeks with a daily reconciliation check comparing key metrics between ecommerce and the ERP: order counts, refund totals, tax amounts, inventory levels. Reconciliation is not glamorous, but it is the fastest way to detect silent failures before they compound.

Build the resilience layer first. Add reconciliation checks second. Those two changes address five of the six failure reports.

---

## What Should Become a Test, SOP, Alert, or Refactor

| Action Type | Item |
| --- | --- |
| Alert | Heartbeat check: fire if no orders successfully processed in defined window |
| Alert | Unrecognized status values encountered during sync |
| Alert | Duplicate order IDs detected in ERP |
| Alert | Tax calculation divergence above defined tolerance |
| SOP | Credential rotation checklist must include middleware update and post-rotation connectivity test |
| SOP | Pre-launch checklist: review sync frequency and inventory buffer settings |
| SOP | Tax configuration change process: any change to one system triggers review of the other |
| Refactor | Add idempotency check to order creation handler |
| Refactor | Add default route for unrecognized status values (review queue, not silent drop) |
| Refactor | Implement dead-letter queue for all webhook event types |
| Refactor | Add retry logic with exponential backoff to refund handler |
| Refactor | Daily reconciliation: order counts, refund totals, inventory levels |
