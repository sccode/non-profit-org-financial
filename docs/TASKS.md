# Implementation Tasks

Each task is sized at about half a day or less. Each one is also a GitHub issue, labeled by phase. Tasks 1 to 38 are issues #1 to #38. Tasks 39 and 40 are issues #40 and #41 (pull request #39 took that number).

Architecture: Google-native (Apps Script + Google Sheets + Google Drive). See [DESIGN.md](DESIGN.md), section 9.

## A. Foundations
1. Create the Apps Script project skeleton (TypeScript + clasp) with CI running typecheck and unit tests
2. Platform spike: two deployments (public vs treasurer-only), direct-to-Drive resumable upload, CAPTCHA options; record findings
3. Treasurer-only access: private web-app deployment plus an email allowlist check
4. Add the audit log helper
5. Sheet database: tabs, headers, and a data-access layer with locking
6. Add a settings screen with a folder-ID field for each Drive folder (Receipts, Donation Receipts, Statements)

## B. Setup data
7. Projects: add, edit, archive
8. Categories: add, edit, archive

## C. Expense intake
9. Public expense form UI (Apps Script HTML Service)
10. Expense submit endpoint with validation
11. Direct-to-Drive resumable receipt upload (server creates the session), with the naming pattern and duplicate suffix
12. Spam protection for public forms: honeypot, rate limit, optional shared passcode
13. Confirmation email to the submitter

## D. Expense review
14. Expense list with filters (status, project, payment method)
15. Detail view showing the Drive receipt beside the entered amount
16. Approve, reject, and edit actions
17. "Mark paid via Zelle" button: marks processed, with date and optional confirmation number

## E. Donation intake and receipts
18. Public donation form with the conditional email field
19. Donation list and verify action
20. PDF receipt generated from a Google Docs template
21. Send receipt email from Gmail, log it, add resend (watch the daily quota)
22. Benevity CSV import with duplicate detection

## F. History import
23. Import the Processed Expenses sheet and link existing Drive receipts
24. Import the Processed Donations sheet

## G. Reconciliation
25. Reconciliation page with a drag-and-drop target for the CSV and statement PDF; parse the CSV in memory
26. Bank CSV parser (Chase format) with running-balance integrity check
27. Credit card CSV parser (needs a sample file)
28. Match suggestions: exact Zelle confirmation number, else exact amount within ±14 days; flag likely internal transfers such as credit card autopay
29. Side-by-side reconciliation screen: transactions beside suggested records, click to confirm
30. Benevity payouts: select several donations against one deposit with a running-sum check
31. Exceptions: manual match, resolve as refund, or ignore with a note
32. On Finish, save transactions and matches to the Sheet, then upload the statement PDF, transactions CSV, and audit report to Statements/<year>/ with standardized names

## H. Dashboard
33. Yearly project income/expense table with year selector
34. One category pie chart per project (Chart.js)
35. All-years income, expense, and net cash flow report
36. CSV export for dashboard tables

## I. Launch
37. Run a past month in parallel with the Sheets process
38. Backups (scheduled Sheet copy to Drive), error alerts by email, and the cutover checklist

## J. Setup and operations
39. Org template: a copy-and-configure setup for another organization
40. Deployment guide: clasp setup and release steps
