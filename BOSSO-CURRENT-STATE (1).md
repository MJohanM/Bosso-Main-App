# Bosso apps — real current state (supersedes the older BOSSO-CURRENT-STATE.md)

This replaces the earlier handoff doc, which had already gone stale — it described a v26
Checker that didn't match what was actually being worked from. Everything below was directly
confirmed against the actual files in this session, not carried forward from memory.

## The two apps — use these exact files, nothing else

1. **Checker app** — `bosso-checker-FIXED-v26.html`. Staff-facing, deep document analysis.
   This specific file is the only correct one. There were THREE different "v26" files floating
   around this project before this one: an old v24, a separately-built v26 made on top of that
   v24, and the real original v26. This file merges the real original v26 with every fix below.
   **No automated regression test file currently exists for it.** The original handoff
   mentioned `bosso-checker-regression-tests.js` with 54 tests — that file was never actually
   available in this session, despite being referenced. Building a real one should be the
   first priority in whatever environment picks this up next (see "Most urgent item" below).

2. **Precheck app** — `bosso-precheck-FIXED-v4.html`. Applicant-facing self-serve screen. Its
   v4 vetting logic was already correct going into this session (auto-decline only for an
   unreadable document or under-18, everything else is a watch note for the Checker app to
   review). Only its version banner was stale (said v3); that's fixed. No other changes.

## Checker app — what's actually in this file

- **Chronological-order tampering check** is direction-aware: it works out whether a given
  statement runs oldest-first or newest-first, and only flags a row that breaks that
  statement's own direction — not every row on a genuine newest-first statement.
- **Existing Debt & Recurring Payments** shows:
  - Real, bank-labelled DebiChecks and debit orders.
  - Other transactions the bank itself explicitly marks as stopped/disputed/returned.
  - Non-debicheck recurring debt (e.g. South African microlenders like Crediwise, Primeloans,
    Lendplus — real creditors that print as plain "LENDER NAME + reference number" on some
    banks' statements, FNB in particular, with none of the standard DebiCheck/debit-order
    wording at all).
  - It deliberately does NOT catch card/retail purchases (Pick n Pay, Woolworths, Uber, etc.)
    even when they coincidentally look like "name + long number" — a merchant denylist plus a
    6+ digit minimum reference length keeps those out. See "A real mistake" below for how
    fragile this balance turned out to be.
- **"Ask about this report" chat panel** — a staff member can ask follow-up questions about an
  already-generated report ("where's that Crediwise payment?") and get an answer grounded only
  in the report's already-extracted data, with a button to jump to and highlight the matching
  row where one exists. Has real multi-turn memory within one chat session (resets when a new
  report is generated, or when the AI provider — Gemini vs Claude — is switched mid-chat).
  **This has only been tested with simulated data and a stubbed network call in this session —
  it has never been run against a real Gemini or Claude API key.** Test it for real before
  trusting it.

## A real mistake made this session — read this before touching debt detection again

The person using this app asked, very clearly, more than once, for the debt/recurring-payment
report to stop showing card and retail purchases. While fixing that, at one point the entire
non-debicheck recurring-payment detection was disabled outright (reduced to an empty array) —
which also removed real, wanted detection of non-debicheck lenders like Crediwise. That was the
wrong fix for the complaint and had to be reverted. The actual correct, working state is:

- KEEP: non-debicheck recurring lender detection (`detectRecurringExternalPayments`).
- KEEP: the retail/card-purchase exclusion inside it (`CARD_MERCHANT_DENYLIST_PATTERN`, the 6+
  digit minimum reference length, `MASKED_CARD_PATTERN`).
- DO NOT disable the whole feature again as a way to fix a false-positive complaint — that
  overcorrects and removes something the person explicitly wants kept.

**If a future change touches this area, a regression test should assert BOTH directions at
once**: that a known non-debicheck lender pattern (e.g. "CREDIWISE 445217890123" appearing
twice) IS still flagged, and that a known retail-purchase pattern (e.g. "PICK N PAY SANDTON
118824" appearing twice) is NOT flagged. A test suite that only checks one direction would have
let this exact mistake through silently.

## Other fixes made this session (all confirmed against real or simulated test cases)

- FNB's card-purchase wording ("PURCH", not "pos") is now recognised by the shared
  `POS_PATTERN`/`MASKED_CARD_PATTERN`, which every purchase-exclusion check reads from.
- A direct prepaid electricity/airtime/water/data exclusion was added (previously only caught
  via an adjacent fee-line marker, which doesn't always appear).

## The Neon migration — DONE, confirmed working

The Render free Postgres database had expired (Render's free-tier databases are deleted 30 days
after creation). The backend (`bosso-handoff-api` on Render) has been migrated to a new,
Bosso-only Neon project (region: Virginia/US East — closest practical option to South Africa
among Neon's available regions, since Neon has none in Africa). `DATABASE_URL` was updated in
Render's Environment tab, the service redeployed cleanly (confirmed via Render's Logs tab — no
repeating crash-loop error), and a real test submission through the Precheck app was confirmed
to save and show up in the Checker app. The old, expired Render Postgres database should be
deleted if that hasn't been done yet — it's inert, just needs tidying up.

## Most urgent item for whatever picks this up next

**There is no real automated regression test suite right now.** Every fix this session was
verified with hand-written, throwaway test scripts in the course of making each change — real,
but not saved anywhere as a reusable file. Given how easily one fix silently undid another
working feature this session (see "A real mistake" above), a proper `bosso-checker-regression-
tests.js` that runs automatically after every change is the highest-value next step — not a new
feature, a safety net so this specific failure mode (a fix for A breaking B, caught only days
later in production) stops happening.

## Known unresolved items, stated plainly

- No regression test file exists (see above — this is the top priority).
- The Precheck app still has zero automated test coverage.
- No idea how many more bank-format-specific gaps like the FNB one exist for banks not yet
  tested against — this is an ongoing risk category, not a one-time fix.
- Whether Render's rate-limiting on `BACKEND_APP_KEY` is actually effective in production was
  never independently load-tested.
- The chat panel feature (above) has never been tested against a real API key.

## Why this document exists

Work on this project has been happening entirely through uploaded/downloaded files in chat,
with no git history and no single file either side can point to as "the real one." That's the
direct cause of at least one real regression this session (building a fix on a stale file,
which then had to be reconciled with a separately-fixed version). This project is moving to
Claude Code specifically to fix that — a real project folder (ideally with git) so there is
always exactly one current version, and so a test suite can run automatically instead of
relying on a human noticing a regression after the fact.
