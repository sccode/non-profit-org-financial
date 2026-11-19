# Implementation Tasks

Each task is sized at about half a day or less. Each one is also a GitHub issue, labeled by phase.

## A. Foundations
1. Create the app skeleton, CI, and a deployable hello-world
2. Create the database schema and migrations
3. Add treasurer login with 2FA
4. Add the audit log helper
5. Connect Google Drive OAuth
6. Add a settings screen with a folder-ID field for each Drive folder

## B. Setup data
7. Projects: add, edit, archive
8. Categories: add, edit, archive

## C. Expense intake
9. Public expense form UI
10. Expense submit endpoint with validation
11. Receipt upload to Drive with the naming pattern and duplicate suffix
12. CAPTCHA and rate limiting on public forms
13. Confirmation email to the submitter

## D. Expense review
14. Expense list with filters (status, project, payment method)
15. Detail view showing the Drive receipt beside the entered amount
16. Approve, reject, and edit actions
17. "Mark paid via Zelle" button with date and confirmation number

## E. Donation intake and receipts
18. Public donation form with the conditional email field
19. Donation list and verify action
20. PDF receipt template
21. Send receipt email, log it, and add resend
22. Benevity CSV import with duplicate detection

## F. History import
23. Import the Processed Expenses sheet and link existing Drive receipts
24. Import the Processed Donations sheet

## G. Reconciliation
25. Detect new files in the Statements folder
26. Bank CSV parser (Chase format) with running-balance integrity check
27. Credit card CSV parser (needs a sample file)
28. Matching engine: exact amount plus date window; ignore internal transfers such as credit card autopay
29. Fuzzy name/memo matching
30. Many-to-one matching for Benevity payouts
31. Reconciliation screen: matched, unmatched, manual match
32. Month-close report saved to Drive

## H. Dashboard
33. Yearly project income/expense table with year selector
34. One category pie chart per project
35. All-years income, expense, and net cash flow report
36. CSV export for dashboard tables

## I. Launch
37. Run a past month in parallel with the Sheets process
38. Backups, monitoring, and the cutover checklist
