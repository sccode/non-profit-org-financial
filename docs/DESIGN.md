# Non-Profit Financial App: Design

Status: draft for review. Last updated 2026-11-20. Architecture: Google-native (Apps Script + Google Sheets + Google Drive).

## 1. Goal

Replace the current Google Forms + Google Sheets + Apps Script workflow with one web app, built on Apps Script, Google Sheets, and Google Drive, that handles:

- Expense (reimbursement) and donation intake
- Treasurer review, payment tracking, and donation receipts
- Reconciliation of bank and credit card transactions against recorded expenses and donations
- A yearly dashboard of income and expenses per project

## 2. Scope and decisions

- **Google-native, no hosting.** Everything runs inside the treasurer's Google account: a Google Sheet holds the data, Apps Script runs the app, and Google Drive holds the files. There are no servers, hosting fees, or databases to maintain (section 9).
- **Single organization per deployment.** A future organization gets its own copy of the Sheet, script, and Drive folders, fully separate. No multi-tenancy.
- **Submitters need no account.** Expense and donation forms are public links, protected by a honeypot field, rate limiting, and an optional shared passcode.
- **One treasurer account:** the treasurer's Google account, with 2FA provided by Google.
- **Zelle payments stay manual.** The treasurer pays outside the app, then marks the expense paid.
- **English only.**
- **Files live in Google Drive**, in folders the treasurer configures. Existing files stay where they are.
- **Bank and credit card transactions are imported from monthly CSV downloads, uploaded directly in the web app.** The app never scans or polls Drive for statements. The official statement PDF is dropped in alongside the CSV. The app parses only the CSV, and archives both files to Drive.
- **Matching is assisted, not automatic.** The app suggests matches; the treasurer confirms each one. No fuzzy name matching.

## 3. Roles

| Role | Access | Can do |
|---|---|---|
| Submitter | Public link, no login | Submit expenses and donations |
| Treasurer | Google account (2FA from Google), private deployment | Manage projects and categories, review expenses, mark paid, verify donations, send receipts, import Benevity reports, run reconciliation, view dashboard |

## 4. Workflows

### 4.1 Expense

Form fields: project, category, description, amount, submitter name, expense date, receipt upload, payment method.

Payment method is one of:

- **Paid by credit card.** No reimbursement. Kept so it can be matched against the credit card transactions.
- **Need reimbursement.** Starts as *Pending*. The treasurer checks the amount against the receipt, approves it, pays by Zelle, then taps **Mark paid via Zelle**, which marks the record processed and records the date and the Zelle confirmation number. The confirmation number is optional, but when it is entered, reconciliation can match the bank row to this expense exactly, because the same number appears in the bank's Zelle description.

Status flow (reimbursement): Submitted, Approved (or Rejected), Paid, Reconciled.
Status flow (credit card): Submitted, Verified, Reconciled.

### 4.2 Donation

Form fields: project, donor name, optional email ("I need a receipt" reveals the email field), donation date, type (Cash, Check, Gift Card, and so on), amount, description.

- The treasurer verifies the amount against the bank transactions.
- If the donor gave an email, the app sends a PDF receipt and logs it.
- **Benevity donations** are imported from the monthly Benevity report (CSV). They are marked as no-receipt because Benevity issues its own.
- **Gift cards and other in-kind gifts** never appear in the bank account. They are flagged non-cash and excluded from bank matching.

Status flow: Submitted, Verified, Receipt sent (if applicable), Reconciled.

### 4.3 Reconciliation

1. The treasurer downloads the month's transactions as CSV, and the official statement as PDF, from the bank (and likewise from the credit card).
2. On the **Reconciliation page** there is a single drag-and-drop target: "Drop this month's CSV and official statement PDF here". The treasurer picks the account (bank or credit card), or the app detects it from the CSV header.
3. The app parses the CSV **in the browser, in memory**, and immediately opens the reconciliation screen. Nothing is written to Drive yet.
4. The app shows a **side-by-side reconciliation screen**: bank or card transactions on one side, candidate records on the other. It only **suggests** matches, and the treasurer confirms each with a click:
   - **Exact Zelle confirmation number:** a certain match, offered as a one-click "confirm all".
   - **Same amount within ±14 days** of the record's date.
   - Bank rows are checked against paid reimbursements, cash/check donations, and Benevity payouts (one deposit: the treasurer selects several donations and sees a running sum against the deposit). Credit card rows are checked against expenses marked "paid by credit card".
5. There is no fuzzy name or memo matching. Names are shown side by side so the treasurer can judge, but they never drive a match. At 10 to 100 transactions a month, a wrong automatic match costs more to unravel than a click costs to confirm.
6. Anything unmatched stays visible. The treasurer can match it manually, ignore it with a note, or mark it as an internal transfer (the app flags likely ones, such as credit card autopay).
   - **Refunds:** a treasurer can resolve an unexpected incoming payment as a refund of a specific expense (for example, a personal purchase charged to the credit card and later repaid by Zelle). The refund offsets that expense. It is not counted as donation income, and the dashboard shows expenses net of refunds.
7. When the treasurer clicks **Finish**, the app saves the transactions and matches to the Sheet, then uploads the **statement PDF**, the **transactions CSV**, and the generated **audit report** to `Statements/<year>/` in Drive with standardized names. Files are uploaded on Finish, not on drop, so an abandoned session leaves nothing behind in Drive.

Re-uploading a month: transactions already saved are detected (same date, amount, description, and running balance) and skipped, so a second upload of overlapping dates does not create duplicates.

### 4.4 Bank CSV format (Chase checking sample)

Columns: `Details, Posting Date, Description, Amount, Type, Balance, Check or Slip #`. Newest rows come first. `Amount` is signed (debits negative). `Balance` is the running balance after each row.

What the sample shows, and how the parser and matcher handle it:

| Row kind | `Type` | How it appears | Handling |
|---|---|---|---|
| Zelle sent | `QUICKPAY_DEBIT`, `CHASE_TO_PARTNERFI` | `Zelle payment to <name> <confirmation#>` | Suggest the reimbursement whose recorded confirmation number equals the one in the description (exact). If none, suggest paid reimbursements with the same amount within ±14 days. |
| Zelle received | `QUICKPAY_CREDIT` | `Zelle payment from <name> <confirmation#>` | Not expected in normal operation. Always lands in the review queue as an exception. The usual cause is a submitter repaying a personal purchase mistakenly put on the organization's credit card, so the treasurer resolves it as a **refund** linked to that expense (see below). |
| Benevity payout | `ACH_CREDIT` | Description contains `benevity` and a `Donation from ...` reference | Match one deposit to many imported Benevity donations. |
| Credit card payoff | `ACH_DEBIT` | `CHASE CREDIT CRD AUTOPAY` | Internal transfer, not an expense. Ignore for matching. The individual card purchases are matched from the credit card CSV instead. |

Notes:

- The recipient name in Zelle descriptions can differ from the name on the expense form (for example a first name only), and the same person can be paid several times on the same day with different amounts. Matching therefore never relies on names: it uses the confirmation number when recorded, otherwise amount and date, and the treasurer confirms.
- The running balance is used as an integrity check: each row's balance should equal the previous row's balance plus its amount. A mismatch (missing or duplicated rows) is flagged before matching.
- Descriptions contain bank identifiers (trace numbers, ORIG IDs). The app stores them but does not display them by default.
- The credit card CSV format is not yet known. It needs its own parser.

## 5. Dashboard (yearly)

A year selector at the top. For the selected year:

- **Project table:** income, expense, and net for every project.
- **Pie charts:** one per project, splitting that project's expenses by category.

Separate **all-years report:** income, expense, and net cash flow per year.

CSV export for each table.

## 6. Google Drive

- The script runs as the treasurer, who approves its Drive, Sheets, and Gmail permissions once. Because the existing folders were not created by the script, it needs the broad Drive permission. The folder IDs are entered in Settings.
- Layout (existing folders are reused, subfolders created only if missing):

```
Receipts/<year>/              expense receipts (existing)
Donation Receipts/<year>/     PDFs sent to donors
Statements/<year>/            statement PDFs, transaction CSVs, and audit reports, uploaded by the app
```

- File names:
  - Expense receipts, existing pattern: `${expenseDate}_${projectName}_${category}_${sender}_$${amount}.${ext}`
  - Donation receipts: `yyyymmdd-donorname-$amount.pdf`
  - Statement PDFs: `yyyymm-<account>-statement.pdf`, e.g. `202610-Checking-statement.pdf`
  - Transaction CSVs: `yyyymm-<account>-transactions.csv`, e.g. `202610-Checking-transactions.csv`
  - Audit reports: `yyyymm-<account>-audit-report.pdf`, e.g. `202610-Checking-audit-report.pdf`
- Duplicate names get a `-2`, `-3` suffix. Special characters are stripped from names.
- Submitters never get Drive access. Receipt photos can be 5 to 15 MB, so the browser uploads them **directly to Drive with a resumable upload**: the script (running as the treasurer) validates the form, creates the Drive upload session (file name, folder, and type set by the script), and hands the browser only that session URL. The file never passes through the script, which avoids Apps Script's request-size limits and lets an upload resume after a dropped mobile connection. If the platform spike (task 2) finds that Apps Script pages cannot do this (a browser origin restriction is the likely problem), the fallback is to shrink photos in the browser and send them to the script in chunks. If a submitter uploads a file and never submits the form, the orphaned file is flagged for cleanup.
- The Sheet stores the Drive file ID and link only.

## 7. Data model (one Sheet tab per record type)

- **Project:** name (e.g. `202510 MidAutumnFestival`, `General Donation`), active flag
- **Category:** name, active flag
- **Expense:** submitted at, project, category, description, amount, submitter name, expense date, payment method, status, receipt Drive file ID, Zelle paid date and confirmation
- **Donation:** submitted at, project, donor name, donor email (optional), donation date, type, amount, description, source (form / Benevity), receipt requested, status, processed date
- **StatementUpload:** account, period, Drive file IDs for the statement PDF, transactions CSV, and audit report, finished-at. The CSV rows are also saved in Transaction.
- **Transaction:** statement upload, date, description, amount, running balance, account
- **Match:** transaction to expense or donation (many-to-one allowed for Benevity), how it was suggested (confirmation number, amount and date, or manual), confirmed by the treasurer
- **Refund:** an incoming transaction linked to the expense it offsets, with a note
- **EmailLog:** receipt and confirmation emails sent
- **AuditLog:** who did what and when
- **Settings:** organization name, EIN, receipt wording, Drive folder IDs

## 8. Existing data

Two existing Sheet tabs are imported once:

- **Processed Expenses:** Timestamp, Project, Category, Description/Reason, Amount, Expensed By, Date of Expense, Receipt Link, Payment Method
- **Processed Donations:** Submit Timestamp, Project, Vendor Name, Vendor Email, Donation Date, Type, Amount, Description, Processed Date

The receipt links already point at Drive files, so the import links them directly with no copying or renaming. The app uses a new spreadsheet; the old one is left untouched as a backup until cutover.

## 9. Architecture (Google-native)

Everything runs inside the treasurer's Google account.

| Piece | What it is |
|---|---|
| Data | One Google Sheet per organization, one tab per record type (section 7). Readable and fixable by hand. |
| Code | One Apps Script project bound to that Sheet, written in TypeScript and pushed with `clasp`. The source lives in this repo. |
| Public site | Web-app deployment A: runs as the treasurer, open to anyone with the link. Serves only the expense form, the donation form, and the upload-session endpoint. |
| Treasurer site | Web-app deployment B: only the treasurer's Google account can open it, plus an email allowlist check in code. Serves review, reconciliation, settings, and the dashboard. |
| Files | Google Drive folders (section 6). |
| Email and PDFs | Gmail for receipts and confirmations. Donation receipts are PDFs made from a Google Docs template; audit reports are PDFs made from HTML. |
| Charts | Chart.js in the browser, fed pre-aggregated rows by the script. |
| Browser-side logic | CSV parsing, match suggestions, and the reconciliation screen run in the browser. The script reads and writes the Sheet. |

**Why two deployments.** Apps Script sets "who it runs as" and "who can open it" per deployment. The forms must be open to people with no Google login, and the treasurer screens must not be, so they are separate deployments of the same project. The platform spike (task 2) confirms this setup works as expected.

**Limits and how the design handles them**

- **Execution time:** each call is capped at a few minutes. Processing a month of tens of rows is far below that.
- **Email quota:** about 100 recipients a day on a free Google account, more on Workspace. That is plenty for donation receipts. If a batch exceeds the quota, the rest wait for the next day and the treasurer is told.
- **Concurrent writes:** a lock around Sheet writes keeps two simultaneous submissions from colliding.
- **Sheet size:** a few thousand rows a year is fine. Older years can be moved to an archive file.
- **Testing:** the logic (CSV parsing, balance check, match suggestions, file naming, dashboard totals) is plain TypeScript with unit tests that run in CI without Google. The Apps Script parts are thin wrappers around it.
- **Deploying:** `clasp push` from a computer with Node installed, or a manual copy into the Apps Script editor (task 40).
- **Backups:** a scheduled script copies the Sheet to a Drive folder weekly (task 38).
- **New organization:** copy the template Sheet (its script comes along), set the folder IDs, and create the two deployments (task 39).

## 10. Security and privacy

- Treasurer pages are a separate deployment that only the treasurer's Google account can open (2FA comes from Google), plus an email allowlist check in code.
- The public deployment runs with the treasurer's authority, so it exposes only fixed, validated actions (submit an expense, submit a donation, create an upload session) and never reads or returns existing records.
- Public forms: honeypot field, rate limit, optional shared passcode, file type and size limits. A real CAPTCHA is awkward inside Apps Script pages; the spike (task 2) checks whether one works.
- Receipt upload session URLs are tied to one file in the configured folder and expire.
- Drive folders private; no link sharing.
- No bank credentials stored. Only the uploaded CSV data is processed.
- Audit log of every approval, edit, and send.
- Donation receipt wording follows IRS requirements.

## 11. Open questions

- Credit card transactions: confirm the card offers CSV download and provide a sample (bank CSV sample received).
- Receipt numbering for donation receipts (sequential or none).
- Existing Apps Script code: your current scripts already send donation receipts and move processed rows. Sharing that code (and the receipt template or wording) lets us reuse the logic instead of rewriting it.
- Deployment: can the treasurer run `clasp` on a computer with Node installed, or should releases be a manual copy into the Apps Script editor?
- Email notification to the treasurer on each new submission.
