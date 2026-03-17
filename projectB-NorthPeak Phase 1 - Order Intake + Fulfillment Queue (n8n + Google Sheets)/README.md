# NorthPeak Home Goods — Phase 1 Workflows (n8n)

> **Important:** this repository snapshot contains **Phase 1 only** of a larger multi-phase automation project.  
> It is **not** the full system and should be presented as a **single completed stage** of the broader NorthPeak operations automation build.

## Scope of this package

This package covers the first working stage of the project:

- order intake through webhook
- input normalization and validation
- duplicate detection
- order upsert into `12_Orders_Master`
- payment/hold eligibility logic
- fulfillment queue handoff into `13_Fulfillment_Queue`
- centralized ops logging in `19_Ops_Log`
- centralized internal alerts in `20_Alerts_Log`

## Included workflows

### Core / Intake
- `01_INTAKE__Order_Webhook_Normalizer`
- `02_ORDER__Payment_To_Fulfillment`

### Utilities
- `UTIL_Validate_Order`
- `UTIL_Check_Duplicate_Order`
- `UTIL_Find_Order_By_ID`
- `UTIL_Upsert_Sheet_Record`
- `UTIL_Update_Order_Status`
- `UTIL_Write_Ops_Log`
- `UTIL_Send_Internal_Alert`

## Not included in this package

The following stages are **not included yet**:

- shipment flow
- support flow
- inventory reservation / reorder
- returns flow
- monitoring / control flow beyond the utilities used in Phase 1

## Repository structure

```text
/workflows/phase1/
  01_INTAKE__Order_Webhook_Normalizer.json
  02_ORDER__Payment_To_Fulfillment.json
  UTIL_Validate_Order.json
  UTIL_Check_Duplicate_Order.json
  UTIL_Find_Order_By_ID.json
  UTIL_Upsert_Sheet_Record.json
  UTIL_Update_Order_Status.json
  UTIL_Write_Ops_Log.json
  UTIL_Send_Internal_Alert.json

/docs/
  PHASE1_REPORT.md
  PHASE1_TESTS.md
  IMPORT_ORDER.md
```

## Environment

- n8n v2.x
- Google Sheets credentials configured in n8n
- spreadsheet used by the workflows: `portfolio_project2_ops_case_upgraded`
- sub-workflows must be **Published** before dependent workflows can execute them via `Execute Workflow`

## Recommended import order

1. `UTIL_Send_Internal_Alert`
2. `UTIL_Write_Ops_Log`
3. `UTIL_Validate_Order`
4. `UTIL_Check_Duplicate_Order`
5. `UTIL_Find_Order_By_ID`
6. `UTIL_Upsert_Sheet_Record`
7. `UTIL_Update_Order_Status`
8. `02_ORDER__Payment_To_Fulfillment`
9. `01_INTAKE__Order_Webhook_Normalizer`

After import, publish all utility workflows first, then publish `02_ORDER__Payment_To_Fulfillment`, then publish `01_INTAKE__Order_Webhook_Normalizer`.

## Portfolio note

For portfolio presentation, describe this package as:

**“Phase 1 of a larger operations automation project: order intake, validation, deduplication, fulfillment queue handoff, and centralized logging/alerts in n8n.”**

Avoid presenting this repository as the full end-to-end NorthPeak system, because it currently documents only the first completed stage.
