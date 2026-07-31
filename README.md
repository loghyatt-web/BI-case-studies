# Diagnosing a Silent Data-Integrity Defect: Invoice Fan-Out in a Financial Summary Report

*A case study in grain control and correctness reasoning in a BI reporting environment (IBM Cognos Analytics / Bullhorn Canvas).*

---

## Context

A financial summary report for a staffing-industry client aggregated payable amounts and billable amounts across placements. The report was already in use by the client's payroll team and had not been flagged by its consumers.

The bill-side totals were correct. The pay-side totals were quietly overstated for a subset of placements. They inflated in a way that never triggered an obvious error, because every number still looked plausible.

I did not build the report. I was asked to find out why some pay figures didn't reconcile.

## The symptom

Nothing was visibly broken. There was no error message, no null, no zero — just a total that was higher than it should have been. The tell was reconciliation: a placement's pay total didn't match the known-correct value when I checked it against the source payable records.

That gap is the whole point of the case. A number that is wrong but plausible is more dangerous than one that is obviously broken, because it survives a glance and flows straight into a leadership dashboard.

## The diagnosis

I isolated the report into its query layers and rebuilt each one as a throwaway list, comparing totals against a known-correct value at every stage. This localized the inflation to a single join: the pay stream was joined to the invoice table on a billable-charge key alone.

The problem is a classic fan-out at the wrong grain. A single billable charge can appear on multiple separate invoices — each with its own invoice ID. Because the join matched on the billable-charge key alone, one charge that was invoiced across three invoices expanded into three rows. When the pay stream fanned out this way:

- **Bill amounts** split correctly across the expanded rows (they sum back to the right total).
- **Pay amounts** replicate in full onto every expanded row, so a single payment gets counted once per invoice it touched.

A toy example makes it concrete. One worker paid \$100 on a single charge that happened to be billed across three separate invoices:

| Invoice ID | Bill Amount | Pay Amount (as stored after join) |
|---|---|---|
| INV-001 | \$40 | \$100 |
| INV-002 | \$35 | \$100 |
| INV-003 | \$25 | \$100 |
| **Sum** | **\$100** ✅ | **\$300** ❌ |

Bill sums back to \$100. Pay inflates to \$300 — a 3× overstatement driven entirely by how many invoices the charge happened to be split across.

The defect is deterministic, which is what made it diagnosable: it affects every placement whose charges are billed across multiple invoices, and *never* affects placements with one-to-one invoicing. That pattern — some placements wrong, some perfect, split cleanly on invoicing structure — was the fingerprint that confirmed the root cause.

## The fix

The join couldn't simply be removed — the report legitimately needed to show each invoice, so the duplicate rows had to stay. The problem was only that pay was being summed across them. So the fix was to count each pay event exactly once regardless of how many invoice rows it fanned into, using a ranking approach:

1. Build a **composite key that uniquely identifies each distinct pay event** — the payable charge plus its transaction date, earn code, transaction origin, and the precise date-and-time it was added — and carry it through every query layer, so each real pay event could be told apart from its fan-out duplicates with certainty.
2. Apply a **ranking window function partitioned by that composite key** — within each pay event, the first row gets rank 1 and its fan-out clones get 2, 3, and so on.
3. Keep the pay value only on the rank-1 row and null it on the duplicates, so when the outer query sums pay, each payment is counted once while the invoice rows themselves remain visible.

The composite key was the important detail. An earlier attempt fingerprinted rows on just amount and earn code, which risked collapsing two legitimately separate pay lines that happened to share those values. Identifying each pay event by the full set of fields — charge, date, earn code, origin, and add-timestamp — removed that risk: the report could now distinguish real distinct pay events from mechanical join duplicates with certainty. I also made the ranking deterministic with an explicit ordering, so the row that "wins" the pay value is the same on every run.

I documented one honest tradeoff alongside the fix: this corrects the totals without changing the join, but it leaves the duplicate rows in place with pay nulled — so per-row pay values are meaningful only on the winning row. The structurally purer alternative (pre-aggregating invoice to one row per charge before the join) was also on the table; the ranking fix was chosen because it kept the invoice-level detail the report needed while still making the totals correct.

After the fix, the pay value stays on one row per pay event and is nulled on the fan-out duplicates. The total is now correct, but note that the per-row pay figure only carries meaning on the winning row:

| Invoice ID | Bill Amount | Pay Amount (after fix) |
|---|---|---|
| INV-001 | $40 | $100 |
| INV-002 | $35 | *(null)* |
| INV-003 | $25 | *(null)* |
| **Sum** | **$100** | **$100** ✅ |

I delivered the diagnosis and remediation options as a stakeholder-facing document written in plain language, so decision-makers who don't read SQL could understand what was wrong, why totals had been unreliable, and what each fix would cost.

## Why it matters

Financial totals that feed leadership decisions have to be correct at the grain — not just plausible at a glance. Fan-out and multi-fact join errors are the most dangerous class of BI defect precisely because they're silent: the report runs, the numbers render, and they're wrong by a clean multiple that nobody notices until something is reconciled against a source of truth.

The durable lesson isn't the specific join. It's the discipline: validate totals against known-correct values rather than trusting that a query that runs is a query that's right.

---

*All identifying details — client, schema specifics, and figures — have been removed or replaced with illustrative values. This write-up describes the reasoning, not any client's data.*
