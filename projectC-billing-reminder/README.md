# Invoice and Payment Reminder Automation

This project automates invoice reminders and tracks payment statuses. It reads invoice data
from a table, filters for unpaid and soon-to-be-due invoices, sends reminder emails, and
produces weekly reports.

## Key Features
- Cron or scheduling trigger for daily/weekly execution.
- Filtering logic to identify overdue or soon-to-be-due invoices.
- Automated email reminders using an SMTP or third-party email node.
- Status updates and count of reminders sent.
- Weekly summary report generation.

## Usage
1. Prepare a table of invoices with fields: `invoice_id`, `client_name`, `email`, `amount`,
   `due_date`, `status`, `reminders_sent`.
2. Use a scheduling node to run the workflow regularly.
3. Filter unpaid invoices and those nearing their due dates.
4. Send personalized reminders via email.
5. Update the table with status and number of reminders sent.
6. Generate and email a weekly summary report.
