# LedgerLens AI

An n8n workflow that turns an uploaded invoice PDF into structured accounting data.

![LedgerLens AI workflow](screenshots/LedgerLens%20AI.png)

## What it does

1. User uploads an invoice PDF through an n8n form.
2. The PDF text is extracted.
3. GPT-5.6 Luna extracts vendor, invoice number, dates, currency, line items, totals, and payment terms.
4. The invoice is logged to Google Sheets.
5. The submitter gets a confirmation email.
6. Finance gets a notification email with the extracted data.

## Current tech

- n8n Forms + file upload
- PDF text extraction
- OpenAI GPT-5.6 Luna with strict structured output
- Google Sheets accounting log
- Gmail notifications

## Setup

1. Import `LedgerLens AI.json`.
2. Set environment variables:

   | Variable | Purpose |
   |---|---|
   | `INVOICE_SHEET_ID` | Google Sheet ID |
   | `FINANCE_EMAIL_TO` | Finance inbox |

3. Connect OpenAI, Google Sheets, and Gmail credentials.
4. Create a sheet tab `Invoices` with headers:

   ```
   Submitted By, Submitted At, Vendor Name, Vendor Email, Vendor Address, Invoice Number, Invoice Date, Due Date, Currency, Subtotal, Tax, Total, Payment Terms, Line Items, Notes, Status
   ```

5. Activate the workflow and share the form URL.

