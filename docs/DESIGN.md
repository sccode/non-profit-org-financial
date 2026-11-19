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
- **Bank and credit card transactions are imported from monthly CSV downloads.** The PDF statements are still kept in Drive for the record, but the app reads the CSV.

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
- **Need reimbursement.** Starts as *Pending*. The treasurer checks the amount against the receipt, approves it, pays by Zelle, then taps **Mark paid via Zelle** (records date and confirmation number).

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

1. The treasurer downloads the month's transactions as CSV from the bank and the credit card, and uploads them (with the PDF) to the by-year Statements folder in Drive.
2. The app lists new, unprocessed statement files.
3. The treasurer clicks **Run reconciliation** on a new file.
4. The app matches transactions to records:
   - Bank: Zelle reimbursements paid, cash/check donations received, Benevity payouts (one deposit matching many donations).
   - Credit card: expenses marked "paid by credit card".
5. Matching order: exact amount, then date window, then name/memo similarity.
6. Leftovers appear as unmatched on either side, and the treasurer can match them manually or add a note.
   - **Refunds:** a treasurer can resolve an unexpected incoming payment as a refund of a specific expense (for example, a personal purchase charged to the credit card and later repaid by Zelle). The refund offsets that expense. It is not counted as donation income, and the dashboard shows expenses net of refunds.
7. A month-close report is saved to Drive.

### 4.4 Bank CSV format (Chase checking sample)

Columns: `Details, Posting Date, Description, Amount, Type, Balance, Check or Slip #`. Newest rows come first. `Amount` is signed (debits negative). `Balance` is the running balance after each row.

What the sample shows, and how the parser and matcher handle it:

| Row kind | `Type` | How it appears | Handling |
|---|---|---|---|
| Zelle sent | `QUICKPAY_DEBIT`, `CHASE_TO_PARTNERFI` | `Zelle payment to <name> <confirmation#>` | Match to a reimbursement marked paid. Pull the confirmation number from the description to pre-fill or verify it. |
| Zelle received | `QUICKPAY_CREDIT` | `Zelle payment from <name> <confirmation#>` | Not expected in normal operation. Always lands in the review queue as an exception. The usual cause is a submitter repaying a personal purchase mistakenly put on the organization's credit card, so the treasurer resolves it as a **refund** linked to that expense (see below). |
| Benevity payout | `ACH_CREDIT` | Description contains `benevity` and a `Donation from ...` reference | Match one deposit to many imported Benevity donations. |
| Credit card payoff | `ACH_DEBIT` | `CHASE CREDIT CRD AUTOPAY` | Internal transfer, not an expense. Ignore for matching. The individual card purchases are matched from the credit card CSV instead. |

Notes:

- The recipient name in Zelle descriptions can differ from the name on the expense form (for example a first name only), and the same person can be paid several times on the same day with different amounts. Matching therefore keys on amount first, then date, then name as a tiebreaker only.
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
Statements/<year>/            treasurer uploads (PDF and CSV)
Reconciliation/<year>/        month-close reports, audit exports
```

- File names:
  - Expense receipts, existing pattern: `${expenseDate}_${projectName}_${category}_${sender}_$${amount}.${ext}`
  - Donation receipts: `yyyymmdd-donorname-$amount.pdf`
  - Reconciliation reports: `yyyymm-reconciliation.pdf` and `.csv`
- Duplicate names get a `-2`, `-3` suffix. Special characters are stripped from names.
- Submitters never get Drive access. The app server uploads on their behalf.
- The database stores the Drive file ID and link only.

## 7. Data model (simplified)

- **Project:** name (e.g. `202510 MidAutumnFestival`, `General Donation`), active flag
- **Category:** name, active flag
- **Expense:** submitted at, project, category, description, amount, submitter name, expense date, payment method, status, receipt Drive file ID, Zelle paid date and confirmation
- **Donation:** submitted at, project, donor name, donor email (optional), donation date, type, amount, description, source (form / Benevity), receipt requested, status, processed date
- **StatementFile:** Drive file ID, account, period, processed flag
- **Transaction:** statement file, date, description, amount, account
- **Match:** transaction to expense or donation (many-to-one allowed for Benevity)
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
- Drive folders private; no link sharing.
- No bank credentials stored. Only the uploaded CSV data is processed.
- Audit log of every approval, edit, and send.
- Donation receipt wording follows IRS requirements.

## 11. Open questions

- Credit card transactions: confirm the card offers CSV download and provide a sample (bank CSV sample received).
- Receipt numbering for donation receipts (sequential or none).
- Email notification to the treasurer on each new submission.
