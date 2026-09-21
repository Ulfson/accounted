# Proposal: file-based customer and supplier subledger import

Status: proposed product specification, not an implemented import feature.

Target: the hosted service at accounted.se / app.accounted.se. This proposal does
not require a self-hosted installation, database access, or a paid connection to
the source accounting provider.

## Problem

A company migrating its bookkeeping through SIE can still lack the individual
customer and supplier invoices needed for payment matching and aging reports.
Recreating those invoices through the ordinary issue or registration flow risks
recording revenue, expenses, VAT, receivables, or payables a second time.

The repository already has file import for customer and supplier master records.
Those records alone do not supply an invoice subledger. Provider migration also
has paths for importing invoice entities and linking existing registration
vouchers. Users need a file-based entry point to the same register-only capability
without an active source-provider integration subscription.

## Proposed user flow

1. Under Import, select Customer subledger or Supplier subledger. Show the active
   company name and organization number throughout the flow.
2. Choose a snapshot date and upload CSV or XLSX. Offer downloadable templates
   and column mapping using the existing register-import components. An optional
   versioned JSON format can support integrations using the same import service.
3. Match each customer or supplier to an existing master record. Offer an explicit
   review step for missing records, reusing the existing master-record import.
   Do not infer an organization number from a customer or supplier number.
4. Preview original invoice amounts, currency, remaining balances, proposed
   registration-voucher links, duplicates, and blocking errors. Keep imported
   invoice numbers; do not consume the normal invoice numbering sequence.
5. Reconcile the snapshot against the corresponding posted general ledger at that
   date. Explain any difference before confirmation.
6. Confirm the selected company, snapshot date, number of invoices, and totals.
   Clearly state that this creates subledger records and no journal entries.
7. Import the reviewed records and return a durable result with imported, skipped,
   and rejected rows, reasons, reconciliation totals, and a batch identifier.
   Open invoices then become available to the ordinary payment-matching flow.

PDF subledger reports and invoice PDFs should also be supported as a reviewed
input path. PDF extraction produces editable candidate data, never automatically
approved financial facts. Attach the original invoice PDFs to the corresponding
records, preserving provenance. Structured CSV/XLSX import is the proposed first
implementation; assisted PDF extraction follows through the same validation and
confirmation flow. Users with only PDFs can transcribe or extract into the
provided template in the first release. A PDF upload alone must not be presented
as successful subledger import.

## Data contract

| Field | Requirement |
| --- | --- |
| Ledger kind | Customer or supplier, fixed for the batch |
| Snapshot date | Required batch-level date |
| Source system and source record ID | Preserve when supplied; do not fabricate source IDs |
| Counterparty | Explicitly resolved company-owned customer or supplier |
| Original invoice number | Required; preserve as text including leading zeros |
| Invoice date and due date | Required, valid calendar dates |
| Document kind | Invoice or credit note |
| Currency | Required; no silent SEK default for unidentified currency |
| Original net, VAT, and gross | Preserve supported values and validate their relationship |
| Outstanding balance | Required, signed consistently with document kind, at snapshot date |
| SEK carrying values | Required for reconciliation of foreign-currency balances; preserve source values |
| Original voucher reference | Series, number, and fiscal year when available |
| Payment reference | Preserve OCR/reference as text when supplied |
| Payment history | Optional, with actual dates, amounts, and existing voucher references |
| Source documents | Optional attachments and original filenames |
| Invoice lines and VAT treatment | Preserve when available; missing details must remain explicitly unknown |

Do not derive original VAT from the remaining amount of a partially paid invoice.
Do not invent a payment date from the invoice date, import date, or a zero balance.
If the native data model cannot safely represent incomplete invoice details, keep
that row staged and require completion instead of manufacturing a zero-VAT line.

Validate money using the project's shared money conventions. Reject invalid dates,
non-finite values, inconsistent signs, unsupported VAT treatments, and unsupported
credit/payment relationships with row-level explanations. Do not silently round,
normalize, or drop a discrepancy out of the preview.

## Safety and consistency requirements

- The importer writes register records, invoice items, document links, and import
  history only. It must never issue an invoice, send it to a counterparty, post a
  registration/payment voucher, or generate automatic reversal entries.
- Reuse the core registration-voucher linking logic where applicable. A link must
  resolve unambiguously within the company and fiscal year and pass the existing
  amount/account checks. An uncertain match remains visibly unresolved.
- Existing posted entries, journal lines, lock state, and voucher sequences remain
  unchanged. Historical invoice dates do not authorize changes to locked periods.
- Missing SIE bookkeeping is reported as a separate prerequisite; importing a
  subledger must never quietly create the missing bookkeeping.
- Import an outstanding balance as a snapshot, not as newly recorded payments.
  An already posted payment can be linked only after an unambiguous match. A later
  bank match books only a genuinely unbooked payment through the existing engine.
- Detect exact repeats using stable source identity and canonical content, scoped
  to company and ledger kind. Customer invoice-number collisions require review.
  Supplier invoice numbers are scoped to the resolved supplier, since different
  suppliers can use the same number. Conflicting content never overwrites an
  existing invoice or its payment history automatically.
- Preview performs no writes. Execution rechecks permissions, company ownership,
  duplicates, and relevant balances after preview. Reject stale previews and
  require renewed review when their financial basis changes.
- Commit a bounded batch atomically with its items, links, and history. If larger
  jobs are needed, use durable per-record receipts and explicit partial progress.
  Retries and simultaneous submissions must not create duplicate records.
- Record actor, company, snapshot date, source file checksum, approved normalized
  data, outcome, and any unresolved reconciliation items in the import history.
  Do not log invoice contents or attach real customer data to public diagnostics.
- Hosted session routes use the normal MFA and write-authorization wrapper. Any
  API entry point uses company-scoped permissions, validation, and idempotency.
  A privileged import worker must still validate every referenced tenant object.

## Reconciliation

Compare signed outstanding SEK balances with the relevant control-account balances
at the snapshot date, usually customer receivables account '1510' and supplier
payables account '2440'. Allow reviewed account mapping for companies that use
additional control accounts. Use the existing ledger report semantics, including
opening balances and posted movements, rather than summing all historical years.

Show totals separately by ledger kind, control account, and currency. Present
foreign-currency valuation differences separately. Incomplete coverage, missing
vouchers, unmatched historical payments, and credit notes must remain visible.
The default should block completion on an unexplained reconciliation difference.
Maintainers should decide whether a separately authorized, audited exception is
needed for intentionally partial migrations before implementing that option.

## Existing building blocks

- Master-record upload, mapping, and review: `app/(dashboard)/import/page.tsx`,
  `components/import/`, and `lib/import/shared/`.
- Master-record routes: `app/api/import/customers/` and
  `app/api/import/suppliers/`.
- Provider invoice mapping and orchestration:
  `extensions/general/arcim-migration/lib/entity-mapper.ts` and
  `extensions/general/arcim-migration/lib/migration-orchestrator.ts`.
- Register-only voucher linking:
  `lib/invoices/link-migrated-registration-vouchers.ts`.
- Invoice-row completion with history: `lib/invoices/complete-invoice-rows.ts`.

Extract reusable provider-independent validation and persistence into core when
necessary. Both provider and file adapters should call that shared service. Core
must not import extension code. Reuse existing durable migration primitives only
where their authorization and source-identity assumptions fit file imports; do
not fabricate a provider consent to make a file look like an API migration.

## Acceptance criteria for implementation

- Both customer and supplier imports are reachable in the hosted import UI, with
  Swedish and English strings, without provider credentials or subscription.
- Full invoices, partial payments, credit notes, foreign currency, leading-zero
  references, and historical dates are either imported faithfully or rejected
  explicitly when unsupported. No financial information is silently invented.
- Same-number invoices from different suppliers remain distinct. Exact retries
  skip safely; conflicting duplicates and concurrent imports cannot overwrite or
  duplicate financial state.
- Before/after database checks prove no journal entries, journal lines, payment
  vouchers, or numbering sequences changed during import. Registration links
  point only to the correct existing company-owned vouchers.
- Importing a remaining balance followed by matching an already posted payment
  cannot post that payment twice. Matching a new payment uses normal bookkeeping.
- Reconciliation includes opening balances, fiscal-year boundaries, configured
  control accounts, credit notes, and SEK valuation of foreign-currency balances.
- Tests cover unauthorized/read-only access, cross-company references, malformed
  files, stale previews, missing/ambiguous vouchers, transaction failure, retries,
  and concurrency. Database invariants are covered by real Postgres tests on the
  permitted staging environment, not mocks alone.
- Documentation clearly distinguishes released structured-file support from any
  future assisted PDF extraction, and explains unresolved-data recovery.

## Alternatives and delivery

Master-record import alone cannot restore invoice balances. A mandatory provider
connection excludes users who cannot justify the source provider's integration
subscription. Ordinary invoice creation followed by reversal entries introduces
unnecessary bookkeeping and is not the proposed migration path. A bespoke SQL
script is inaccessible to hosted users and bypasses the normal review workflow.

Recommend a shared, reviewed register-only importer, first exposed through the
hosted CSV/XLSX wizard, then extended with API/JSON and assisted PDF adapters.
Maintainers need to approve the scope and reconcile-exception policy, implement
and validate the feature, and deploy it to the hosted service. Merging this
specification alone does not enable invoice import or change any company data.
