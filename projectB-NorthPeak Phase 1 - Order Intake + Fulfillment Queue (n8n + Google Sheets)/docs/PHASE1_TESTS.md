# PHASE1_TESTS.md

## Test matrix for Phase 1

### Scenario 1 — Order not found
**Input:** `02_ORDER__Payment_To_Fulfillment` receives an `order_uid` that does not exist in `12_Orders_Master`  
**Expected result:**
- no fulfillment queue row created
- alert logged to `20_Alerts_Log`
- ops log written to `19_Ops_Log`
- workflow ends gracefully

### Scenario 2 — Pending / not paid
**Input:** order exists, but `payment_status != paid`  
**Expected result:**
- no fulfillment queue row created
- order is not auto-promoted
- ops log records the skip

### Scenario 3 — Paid but on hold
**Input:** order exists, `payment_status = paid`, `hold_review = yes`  
**Expected result:**
- no fulfillment queue row created
- order is not auto-promoted
- ops log records the skip
- alert may be logged depending on branch configuration

### Scenario 4 — Paid and eligible
**Input:** order exists, `payment_status = paid`, `hold_review = no`, order is in early stage  
**Expected result:**
- row created or updated in `13_Fulfillment_Queue`
- order status updated to `queued` where applicable
- ops log records success

### Scenario 5 — Re-run / idempotency
**Input:** rerun the same eligible order through `02_ORDER__Payment_To_Fulfillment`  
**Expected result:**
- queue state is not reset
- `queued_at` is not overwritten without reason
- existing progress fields are preserved
- ops log still records the execution result

## Manual validation checklist

- [ ] webhook responds correctly on intake
- [ ] invalid intake payload is rejected or logged correctly
- [ ] duplicate order does not create a second master row
- [ ] paid/no-hold order reaches fulfillment queue
- [ ] pending or hold orders are skipped
- [ ] logs are written to `19_Ops_Log`
- [ ] alerts are written to `20_Alerts_Log` where expected
- [ ] utility workflows are published and callable
