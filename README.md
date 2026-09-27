# LedgerLens AI

When a submitter uploads a PDF invoice, extract and record it for finance review, then stop.

## Production operations

**Purpose:** When a submitter uploads a PDF invoice, extract and record it for finance review, then stop.

- **Scope:** The submitted PDF, the Invoices tab, and the two configured mailboxes. No payment, approval, deletion, or accounting ledger posting.
- **Success:** One Invoices row keyed by Invoice Key; confirmation and finance email after the row is recorded.
- **Inputs:** n8n form PDF and email; OpenAI extraction; environment values for sheet and finance inbox.
- **Philippine use:** Amounts retain the invoice currency; PHP is not assumed for foreign invoices. Dates and totals require human finance review, including Philippine VAT treatment.

### Configure and run

1. Import this workflow and `Failure Alert.json` into n8n. Connect OpenAI, Google Sheets and Gmail credentials. For a webhook workflow, also connect a Header Auth credential and configure the caller to send it. Use a dedicated Google account with access only to the named spreadsheet and mail account.
2. Create the tabs and exact headers: Invoices: Invoice Key, Run ID, Submitted By, Submitted At, Vendor Name, Vendor Email, Vendor Address, Invoice Number, Invoice Date, Due Date, Currency, Subtotal, Tax, Total, Payment Terms, Line Items, Notes, Status.
3. Set required environment values: `INVOICE_SHEET_ID, FINANCE_EMAIL_TO`. For CareerCompass, use `JOBHUNT_SEARCH_URL=https://hn.algolia.com/api/v1/search_by_date?query=%22Ask%20HN%3A%20Who%20is%20hiring%22&tags=story&hitsPerPage=30` and `JOBHUNT_ITEM_API_BASE=https://hacker-news.firebaseio.com/v0/item` as starting URLs. Set `LEDGERLENS_ALERT_EMAIL_TO` to the operator's real inbox.
4. Optional config: None. Set `LEDGERLENS_ENABLED=true` to enable runs. Dry run is on unless `LEDGERLENS_DRY_RUN=false` is explicitly set. The kill switch is `LEDGERLENS_ENABLED=false`; deactivate the workflow too for an immediate stop.
   On self-hosted n8n, set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` or Code nodes cannot read these values. Run these workflows on a dedicated instance; keep API secrets in n8n credentials and expose only workflow config through environment values.
   Set `N8N_CONCURRENCY_PRODUCTION_LIMIT=1` on that dedicated instance to serialize Sheet lookups and writes. Set `N8N_FORMDATA_FILE_SIZE_MAX=10` to cap PDF form uploads at 10 MiB.
5. Open this workflow's n8n Settings and select the imported Failure Alert workflow as **Error Workflow**. Send its Gmail node from an account the operator monitors. Keep execution retention appropriate for sensitive data.
6. Run `python smoke_test.py` after every edit; GitHub Actions runs it on push and pull request. With dry run on, run the n8n workflow once and confirm no Google/AI/Gmail nodes executed. Then use a test sheet, test inbox, and representative input before setting `LEDGERLENS_DRY_RUN=false`.

Event: PDF form submission. Owner: finance operations. If the instance is offline, scheduled events are missed and webhook callers must retry with the same request ID. Manually replay missed scheduled runs after recovery. Review this workflow each quarter; retire it when its owner, input source, or business process no longer exists.

### Reliability, audit, and failure response

- **Reruns:** Vendor name plus invoice number forms Invoice Key. Existing rows stop a sequential rerun before Gmail. Google Sheets lookup plus upsert is sequential idempotency; Sheets has no atomic uniqueness constraint, so concurrent runs can still duplicate rows. Keep one active run per workflow and reconcile after interruptions.
- **Partial failure:** If an email fails after the row is written, a rerun skips that email. The operator must inspect n8n history and resend manually. Never blindly retry a Gmail node after a timeout.
- **Timeouts/retries:** The workflow has a 300 second execution limit; HTTP calls have 15 second timeouts and three attempts with a two second wait. AI calls use a 30 second timeout. n8n's generic retry also retries some permanent HTTP errors, so disable that per node if your source returns persistent 4xx responses. Writes are not automatically retried because an ambiguous timeout can follow a successful write.
- **Audit:** n8n execution history and JSON console logs include timestamp, execution ID, and result without raw customer content. Sheet rows carry business keys and run IDs where available. Inspect failed executions and the `Failure Alert` email. The alert workflow depends on n8n being up; use an external uptime monitor for instance outages.
- **Security:** Store OAuth/API secrets in n8n credentials, not JSON or git. Protect webhook URLs with authenticated ingress and rate limits. Restrict who can view execution history, Sheets, and Gmail. Do not feed sensitive customer or financial data to OpenAI without your organization's approval and retention policy.
- **Exit status/health:** `python smoke_test.py` exits 0 on pass and nonzero on failure. In n8n, the execution status is the scheduler signal; monitor failed and missing scheduled executions. The latest timestamp in the relevant sheet is the simple status indicator.

### Go-live preflight

- [ ] Operator and failure inbox assigned; Error Workflow selected and alert tested.
- [ ] Dry run checked on test input; no live side effect occurred.
- [ ] Required variables and least-privilege credentials configured; no secrets in the export.
- [ ] Same request/lead/invoice rerun checked in a test sheet; no second mail or draft.
- [ ] Network timeout and failure alert checked; partial-write recovery rehearsed.
- [ ] `LEDGERLENS_ENABLED=false` demonstrated as the kill switch.

**Production gate:** Offline smoke tests do not prove n8n import compatibility or live Google/Gmail/OpenAI behavior. Complete the test-account run above and review the Sheets concurrency and ambiguous-email limits before activating on real data.

These exports target self-hosted n8n. n8n Cloud may block `$env` access inside Code nodes; verify compatibility before importing there.
