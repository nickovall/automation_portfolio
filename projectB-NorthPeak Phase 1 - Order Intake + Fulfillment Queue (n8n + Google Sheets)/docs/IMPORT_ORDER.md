# IMPORT_ORDER.md

## Import order

Import and publish workflows in this order:

1. `UTIL_Send_Internal_Alert`
2. `UTIL_Write_Ops_Log`
3. `UTIL_Validate_Order`
4. `UTIL_Check_Duplicate_Order`
5. `UTIL_Find_Order_By_ID`
6. `UTIL_Upsert_Sheet_Record`
7. `UTIL_Update_Order_Status`
8. `02_ORDER__Payment_To_Fulfillment`
9. `01_INTAKE__Order_Webhook_Normalizer`

## Important n8n note

These workflows use `Execute Workflow`.

In n8n v2.x, dependent sub-workflows must usually be **Published** before they can be executed reliably from parent workflows.

## After import

- connect Google Sheets credentials
- confirm Spreadsheet ID points to the correct spreadsheet
- verify referenced sheet names exist
- publish all utility workflows
- publish `02_ORDER__Payment_To_Fulfillment`
- publish `01_INTAKE__Order_Webhook_Normalizer`
- run manual smoke tests before presenting the repo as complete

## Recommended smoke test order

1. test `UTIL_Write_Ops_Log`
2. test `UTIL_Send_Internal_Alert`
3. test `UTIL_Find_Order_By_ID`
4. test `02_ORDER__Payment_To_Fulfillment`
5. test webhook intake on `01_INTAKE__Order_Webhook_Normalizer`
