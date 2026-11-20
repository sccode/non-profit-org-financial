# Non-Profit Financial App: Design

Status: draft for review. Last updated 2026-11-19.

## 1. Goal

Replace the current Google Forms + Google Sheets + Apps Script workflow with one web app that handles:

- Expense (reimbursement) and donation intake
- Treasurer review, payment tracking, and donation receipts
- Reconciliation of bank and credit card transactions against recorded expenses and donations
- A yearly dashboard of income and expenses per project

## 2. Scope and decisions

- **Single organization per deployment.** Future organizations get their own fully separate deployment and database. No multi-tenancy.
- **Submitters need no account.** Expense and donation forms are public links, protected by CAPTCHA and rate limiting.
- **One treasurer account**, with login and 2FA.
- **Zelle payments stay manual.** The treasurer pays outside the app, then marks the expense paid.
- **English only.**
- **Files live in Google Drive**, in folders the treasurer configures. Existing files stay where they are.
- **Bank and credit card transactions are imported from monthly CSV downloads, uploaded directly in the web app.** The app never scans or polls Drive for statements. The official statement PDF is dropped in alongside the CSV. The app parses only the CSV, and archives both files to Drive.
- **Matching is assisted, not automatic.** The app suggests matches; the treasurer confirms each one. No fuzzy name matching.

## 3. Roles

| Role | Access | Can do |
|---|---|---|
| Submitter | Public link, no login | Submit expenses and donations |
| Treasurer | Login + 2FA | Manage projects and categories, review expenses, mark paid, verify donations, send receipts, import Benevity reports, run reconciliation, view dashboard |

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
3. The app parses the CSV **in memory** and immediately opens the reconciliation screen. Nothing is written to Drive yet.
4. The app shows a **side-by-side reconciliation screen**: bank or card transactions on one side, candidate records on the other. It only **suggests** matches, and the treasurer confirms each with a click:
   - **Exact Zelle confirmation number:** a certain match, offered as a one-click "confirm all".
   - **Same amount within ±14 days** of the record's date.
   - Bank rows are checked against paid reimbursements, cash/check donations, and Benevity payouts (one deposit: the treasurer selects several donations and sees a running sum against the deposit). Credit card rows are checked against expenses marked "paid by credit card".
5. There is no fuzzy name or memo matching. Names are shown side by side so the treasurer can judge, but they never drive a match. At 10 to 100 transactions a month, a wrong automatic match costs more to unravel than a click costs to confirm.
6. Anything unmatched stays visible. The treasurer can match it manually, ignore it with a note, or mark it as an internal transfer (the app flags likely ones, such as credit card autopay).
   - **Refunds:** a treasurer can resolve an unexpected incoming payment as a refund of a specific expense (for example, a personal purchase charged to the credit card and later repaid by Zelle). The refund offsets that expense. It is not counted as donation income, and the dashboard shows expenses net of refunds.
7. When the treasurer clicks **Finish**, the app saves the transactions and matches to the database, then uploads the **statement PDF**, the **transactions CSV**, and the generated **audit report** to `Statements/<year>/` in Drive with standardized names. Files are uploaded on Finish, not on drop, so an abandoned session leaves nothing behind in Drive.

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

- Treasurer authorizes Drive access once (OAuth, narrow `drive.file` scope) and enters the folder IDs in Settings.
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
- Submitters never get Drive access. Receipt photos can be 5 to 15 MB, so the browser uploads them **directly to Drive with a resumable upload**: the server validates the form, creates the Drive upload session (file name, folder, and type set by the server, using the treasurer's authorization), and hands the browser only that session URL. The file never passes through the app server, which avoids serverless request-size limits and lets an upload resume after a dropped mobile connection. If a submitter uploads a file and never submits the form, the orphaned file is flagged for cleanup.
- The database stores the Drive file ID and link only.

## 7. Data model (simplified)

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

## 8. Existing data

Two existing Sheet tabs are imported once:

- **Processed Expenses:** Timestamp, Project, Category, Description/Reason, Amount, Expensed By, Date of Expense, Receipt Link, Payment Method
- **Processed Donations:** Submit Timestamp, Project, Vendor Name, Vendor Email, Donation Date, Type, Amount, Description, Processed Date

The receipt links already point at Drive files, so the import links them directly with no copying or renaming.

## 9. Tech stack (suggested)

- Next.js (TypeScript) with Postgres
- Google Drive API (OAuth) for files
- Email via Resend or SendGrid
- PDF generation for donation receipts and reports
- Hosting: Vercel or Render, with managed Postgres and nightly backups

## 10. Security and privacy

- Treasurer login with 2FA; all treasurer routes require it.
- Public forms: CAPTCHA, rate limiting, file type and size limits.
- Receipt upload session URLs are tied to one file in the configured folder and expire.
- Drive folders private; no link sharing.
- No bank credentials stored. Only the uploaded CSV data is processed.
- Audit log of every approval, edit, and send.
- Donation receipt wording follows IRS requirements.

## 11. Open questions

- Credit card transactions: confirm the card offers CSV download and provide a sample (bank CSV sample received).
- Receipt numbering for donation receipts (sequential or none).
- Stack: keep Next.js + Postgres + Vercel (section 9), or move to a lighter, Google-native setup. Under review.
- Email notification to the treasurer on each new submission.
