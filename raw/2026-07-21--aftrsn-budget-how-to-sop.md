---
type: raw
title: "AFTRSN_Budget_How-To_SOP"
source_url: "dropbox:/apps/lumina ai os/lumina vps/lumina ai/obsidian/lumina inbox/aftrsn_budget_how-to_sop.md"
captured: 2026-07-21
vault: brands
brand: AFTRSN
immutable: true
---

# 📘 Budget — How to enter & track (AFTER SUN PEOPLE)

*Standard operating procedure for the **Budget & Finance** database and the automatic per-edition accounting. Keep it simple: you only ever record **lines**; every total calculates itself.*

---

## The one rule to remember

**Never type a total.** You record individual money lines in **Budget & Finance**, link each one to its **Edition**, and the edition adds everything up on its own (Spent, Revenue, Profit, % of budget used). If a number looks wrong, fix the *line*, not the total.

There are three layers:

1. **Budget & Finance** — the ledger. One row = one movement of money (a cost or an income).
2. **Editions** — each event automatically sums its own lines (Spent Σ, Revenue Σ, Planned Σ).
3. **Indicators** — the edition then computes Profit, % used, and the "spent / cap" readout by formula.

---

## A) Set the envelope (once per edition)

Before tracking, give the edition its ceiling.

1. Open the **Editions** database → open the edition (e.g. *EP4*).
2. Fill **Budget Cap** = the maximum you allow yourself to spend for this event (in CHF).
3. That's it. The bar / % on the edition now measures spending against this cap.

> The Budget Cap is a decision, not a calculation — you type it. Everything else is automatic.

---

## B) Enter an EXPENSE line

Do this every time money goes out (artist fee, rental, ads, etc.).

1. Open **Budget & Finance** → **+ New**.
2. **Line item** — short name (e.g. `DJ fee — MAIAN`).
3. **Type** → **Expense**.
4. **Category** — pick the right one (Main Artist(s), Staff & Rental, Advertising, Production, …).
5. **Edition** — link the event this belongs to. **This link is what makes the total work — never skip it.**
6. **Planned** — the amount you budgeted.
7. **Actual** — the real amount, once known (leave empty until then).
8. **Status** — move it along as reality changes (see section E).

`Variance` fills itself (Actual − Planned): negative = under budget, positive = over.

---

## C) Enter a REVENUE line

Do this for money coming in (bar sales, ticket sales, sponsor cash, …).

1. **+ New** → **Type** → **Revenue**.
2. **Line item** (e.g. `Bar sales`), **Category** (Product Sales, Ticket Sales, Sponsors…), **Edition** (link it!).
3. For volume-based income, use **Quantity** × **Unit Price** — `Auto Amount` computes the product for you (e.g. 985.5 drinks × 0.20 = 197.10).
4. Put the realised figure in **Actual**; keep the forecast in **Planned**.

---

## D) The Status workflow (per line)

Move each line as it progresses. This is how you tell "planned" money from "real" money.

- **Planned** — budgeted, not committed yet.
- **Committed** — ordered / agreed, invoice pending.
- **Approved** — validated by the founder, ready to pay.
- **Paid** — money actually moved. ← counts as fully real.
- **Needs fix** — something's off, put it here so it's visible.

Rule of thumb: **Actual + Paid = real money**. Keep Actual accurate, because that's what feeds the edition's Spent and Revenue.

---

## E) Read the automatic indicators (on the Edition)

Open any edition and read — you never edit these, they're computed:

- **Spent Σ** — sum of all its expense lines (real).
- **Revenue Σ** — sum of all its income lines (real).
- **Planned Σ** — sum of the planned expenses.
- **Profit** — Revenue Σ − Spent Σ.
- **Budget Used** — Spent Σ ÷ Budget Cap (how much of the envelope is gone).
- **Budget Display** — the quick `spent / cap` text.

**Worked example — Mini Café, 26.06:** its lines (three artists at 250 planned, equipment 800 paid, bar 197.10 income…) automatically produce **Spent 950**, **Revenue 197.10**, **Profit −752.90**. Nobody added those up — the edition did.

---

## F) Useful views (no setup needed)

On **Budget & Finance**, switch tabs:

- **By Edition** — every line grouped per event, with Planned vs Actual side by side. Your main tracking screen.
- **By Category** — where the money goes across all events.
- **Revenue** — income only, with Quantity × Unit Price.

On **Editions** → **Pipeline** / **By Format** you see each event's Profit and budget at a glance.

---

## G) Don't confuse: task **Cost** vs Budget & Finance

- The green **Cost** bar on a **task card** is a **quick manual estimate** you type on that task — handy to eyeball the weight of a job. It is **not** accounting.
- **Budget & Finance** is the **single source of truth** for real money. When something is actually spent or earned, it must exist as a line here.

If a figure matters financially, it lives in Budget & Finance — full stop.

---

## H) Weekly / per-event routine (5 minutes)

1. Add any new expense/revenue **lines** that appeared this week (link the Edition!).
2. Update **Actual** and **Status** on lines that moved (Committed → Approved → Paid).
3. Open the edition and glance at **Budget Used** and **Profit** — react if the envelope is filling up.
4. Anything unclear → set the line to **Needs fix** so it's not forgotten.

---

## Quick checklist for one line

- [ ] Line item named clearly
- [ ] Type set (Expense / Revenue)
- [ ] Category chosen
- [ ] **Edition linked** ← the critical one
- [ ] Planned filled; Actual filled once known
- [ ] Status current (…→ Paid when done)

*Do this and the totals, profit, and budget bars take care of themselves.*
