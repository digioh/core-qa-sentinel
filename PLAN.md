# Product / Feature QA Agent — Plan

Status: **routine created and enabled 2026-10-14.** Playbook: `core.md`.
Goal: a Claude Code desktop routine that picks Product / Feature QA tasks from the Asana **QA** board, runs the engineer's test plan in Chrome, and reports in Hussain's style. Experiment: can this replace manual core QA, or only assist it?

---

## 1. Current state (as found 2026-10-14)

| Item | State |
|---|---|
| QA board (`1203348184098714`) | Holds both client Final QA and core / product QA. Sections: Upcoming, Ready, In progress, Review and update, Complete, Blocked |
| Product Release QA board (`1209283663848605`) | Already **archived**. Core tasks are still multi-homed there (sections New task / In progress / Submitted / Completed) |
| QA Type field (`1219235193315472`) | Product / Feature QA = `1219235193315473`, Final QA = `1219235193315474`. An Asana rule sets **Final QA** on every task added to the QA board (since ~Oct 8) — Hussain is fixing the rule |
| QA Type backfill | **Done 2026-10-14**: 23 core tasks set to Product / Feature QA (10 open, 13 completed). Not changed: `1218717326667659` (Khizer, digioh.com JS fix) and `1211307337371261` (TEMPLATE - Testing REAL Campaigns) — judgment calls |
| Final QA automation | `qa-sentinel` routine (`D:\Work\claude\qa-sentinel`), every 3 h, 14:00–24:00 UTC, Ready section only, 6-agent fan-out, posts `--For QAer-- Claude <date>` sub-task. Working; Ready is currently empty |
| Core QA automation | None |

Open core tasks today: ID-15778, 15690, 15443, 15675, 15672, 15780, 15765, 14849, 15089, plus `1218692434149265` (AI campaigns publish as file — "not testable yet" until Pablo pings).

---

## 2. How core QA works today (what the agent must copy)

**Task creation.** Pablo Feldman (sometimes Alexander Schwarzgruber, or a "Software Factory" stub Pablo later rewrites) creates `QA: ID-xxxxx <summary>` as a subtask of the engineering ticket, assigned to Hussain. Automation adds it to QA → Upcoming.

**Description template (very consistent):**
1. `Parent: ID-xxxxx . PR: <Azure DevOps URL>`
2. Where to test — env (`mcphqdev2.digioh.com` dev2 / `account.digioh.com` prod / connectors-ui-dev / Shopify admin), login, account to impersonate, Chrome normal window
3. What changed (before/after)
4. Before you start — setup, feature flags in Admin > Edit User, URL params (`?boxqamode=1`, `&digioh_conditions_monitor=1`)
5. `Test 1 … Test N` — numbered steps + explicit `PASS:` / `FAIL:` lines
6. If a test fails — what to attach
7. Not covered here

**Gotchas the agent must handle**
- Later comments override the plan (e.g. Pablo moved all Smart Rules QA from dev2 to prod account 45281 on 1218595965611392). Read **all comments, including sibling QA tasks of the same parent**, before running.
- Gates in the plan: "start once parent is Ready for QA", "run only once ID-15675 is in production (10/14 release)". Check the parent ticket state; if gated → skip that test and say so.
- Explicit escape hatches: "if X is not offered, skip this test and say so in your comment".

**Recurring test accounts:** 46338 praxis2 (main), 45281 Digioh Quiz Training + `digioh-managed-store.myshopify.com` (Smart Rules runtime, campaign 336251), 46341 praxisreadonly (no-flags regression), 46388 (Smart Rules, do not touch in prod), 46377 Klaviyo, 45521 Pablo, 44849/45470 GA, 45434 Iterable, 43060 support.

**Hussain's report style (exact)**
- All pass: `It is looking good.`
- Findings: one line each, no header, no greeting:
  `Test 3, point 7: The badge disappears, but "+ Add" under display rule is still showing. We should update it to "Edit": https://snipboard.io/iEZHIC.jpg`
- Questions: end with `Is this expected?`
- Blocked: `Unable to start, <reason>. <link to related task>`
- Retest fail: `Issues are not fixed` + list of `Test N, point M` refs + any new finding
- Then a **separate** comment containing only `@Pablo Feldman`
- Close (only after Pablo resolves/reclassifies): `Marking this as complete.`
- Plain text, no bold/headers/emojis.

**Bookkeeping Hussain does:** Issues & Queries = number of findings (0 on pass). Moves section: Product Release QA → In progress → Submitted; QA board → In progress → Review and update. Marks complete himself at the end (automations then sync Status, Engineering "Done", etc.). Never sets Status by hand.

**Pablo's replies:** per finding — "real bug, fixed in dev2, re-test" / "working as designed, plan was confusing" / "tracked in <ticket>". Usually 1–2 rounds.

---

## 3. Proposed design

### 3.1 Routing — DECIDED: separate routine
New routine **`core-qa-sentinel`**, separate from `qa-sentinel` (which stays untouched and keeps handling Final QA).

```
QA board task, incomplete
├── QA Type = Final QA             → qa-sentinel (existing, unchanged)
└── QA Type = Product / Feature QA → core-qa-sentinel (new)
```

| Setting | Value |
|---|---|
| Model | **Sonnet** (routine and any subagents) |
| Schedule | Cron `47 1-22/3 * * *` (local PKT), time gate **14:00–24:00 UTC** — same window and cadence as qa-sentinel, offset 1.5 h |
| Working dir | `D:\Work\claude\core-qa-sentinel`, repo `digioh/core-qa-sentinel` (playbook `core.md`, reports in `reports-core/<ID>/`) |
| Browser | Claude in Chrome (Hussain's logged-in Chrome) |

Browser collision — DECIDED: separate by time, no change to qa-sentinel.

| qa-sentinel fires (PKT) | core-qa-sentinel fires (PKT) | UTC | In 14–24 UTC window? |
|---|---|---|---|
| 18:17 | 19:47 | 14:47 | core only (sentinel 13:17 UTC is outside) |
| 21:17 | 22:47 | 17:47 | both |
| 00:17 | 01:47 | 20:47 | both |
| 03:17 | 04:47 | 23:47 | both |

- Each routine gets a 1.5 h slot before the other fires. qa-sentinel runs median ~42 min, worst seen ~90 min; core runs are expected to be shorter (one task, one browser leg).
- Residual risk: a qa-sentinel run longer than ~90 min overlaps core. Guard (core side only): at start, call `list_task_runs("asana-qa-sentinel")`; if the latest run is `running`, stop silently and wait for the next slot.
- Core must also finish before the next qa-sentinel fire: hard stop at ~80 min (write what's done, mark the rest NOT RUN, post the sub-task).
- Note: cron is local Pakistan time (UTC+5); qa-sentinel has ~5 min jitter.

### 3.2 Pickup rules (core)
- QA Type = Product / Feature QA, incomplete, assigned to Hussain, section Upcoming or Ready (or In progress for retest — see 3.6).
- No `(QA run by Claude)` marker comment yet for the current round.
- Not gated (parent ready / release date reached / no "not testable yet" comment without a follow-up ping).
- One task per run, oldest first.

### 3.3 Run steps
1. Read task, all comments, parent ticket, sibling QA tasks' latest comments.
2. Parse plan into a test list `{test, steps[], pass, fail, env, account, flags, params}` → save `reports-core/<ID>/plan.json`.
3. (No section moves — parent stays as is.)
4. Setup (see §5 permissions): login is already in Chrome; impersonate account; set flags; create/duplicate campaigns as the plan says.
5. Execute each test literally in Chrome (Claude in Chrome tools). Evidence per step: screenshot on any deviation, Conditions Monitor text, console errors, Agent reply text.
6. Grade each test PASS / FAIL / QUERY / SKIPPED(reason).
7. Cleanup as the plan says (restore rules, archive copies, revert flags).
8. Upload screenshots (§4), create the `QA run by Claude` sub-task with the findings comment, re-fetch to verify it posted.
9. Save to `reports-core/<ID>/` (e.g. `reports-core/ID-15765/`):
   - `plan.json` — parsed test list
   - `report.txt` — exact text posted to the sub-task
   - `*.png` — full-size screenshots (kept locally even after snipboard upload)
   - `run-log.md` — per-test PASS / FAIL / QUERY / SKIPPED / NOT RUN with reasons and snipboard URLs
   - Retest rounds go in `retest-YYYY-MM-DD/` so round 1 is never overwritten.

### 3.4 What it posts — DECIDED: sub-task, no mentions
The agent **only creates a sub-task** on the QA task. It never comments on the parent and **never mentions Pablo (or anyone)**. Hussain reviews it and posts his own findings on the parent, mentioning Pablo himself.

- Sub-task name: `QA run by Claude YYYY-MM-DD` (round 2+: `QA run by Claude YYYY-MM-DD (retest)`). Also the pickup marker — a task with one for the current round is skipped.
- Sub-task left **open**, **unassigned**.
- One comment on the sub-task, Hussain's format, plain text, no @-mentions:
```
Test 2, point 3: When we click escape, it closes the agent window. https://snipboard.io/xxxx.jpg
Test 6, point 3: Image preview shows a broken thumbnail while the first upload is in progress: https://snipboard.io/yyyy.jpg
Test 5: Is this expected? The "How often" row moves to the top after save. https://snipboard.io/zzzz.jpg
Skipped: Test 4 (needs a physical microphone), Test 7 (gated on 10/14 production release).
Passed: Test 1, 3, 8.
```
  or `It is looking good.` + Skipped line if any.
- Does **not** touch the parent: no section moves, no Issues & Queries, no completion. Hussain does those after review.

### 3.5 What it skips (says so in the comment)
Physical microphone, OS clipboard / Explorer paste, Firefox/Safari, real Android/iPhone, Shopify admin login, Google Ads account creation, HubSpot/ESP back-office checks, nightly jobs / pixel logs / real rates (engineering-only), multi-user concurrency tests.

### 3.6 Retest rounds (phase 2)
Trigger: Pablo comments after the last Claude/Hussain comment with "fixed / re-test". Agent re-runs only the referenced tests and creates a new `QA run by Claude <date> (retest)` sub-task with `Issues are not fixed` + refs, or `It is looking good now.` Still never closes, never mentions.

---

## 4. Screenshots — snipboard.io upload

Idea (Hussain): capture → open snipboard.io → paste → use the URL in the comment.

**VERIFIED 2026-10-14** — test upload: https://snipboard.io/xeA7lR.jpg (digioh.com homepage).

Claude cannot put an image on the OS clipboard, but Claude in Chrome's **`upload_image`** drops a fresh screenshot into snipboard's hidden file input. Working flow:
1. On a deviation: `computer.screenshot` (or `zoom` on the region) with **scale 1** (the 0.5 test image was only 518 px wide — too small for UI detail) → `imageId`.
2. Keep a snipboard tab open. `navigate` it to `https://snipboard.io`, `find` "pick a file" → file input ref.
3. `upload_image(imageId, snipboardTabId, ref)` — within a few minutes of the screenshot.
4. Wait ~3 s; the tab's `location.href` becomes `https://snipboard.io/<id>.jpg` ("Saved as:" on the page). Read it with `javascript_tool` → put it in the finding line.
5. Navigate the snipboard tab back to `https://snipboard.io` for the next upload.

Fallback if snipboard is down: text-only finding + local file in `reports-core/<ID>/` for Hussain to attach (Asana connector cannot upload attachments).

Note: snipboard links are public-by-URL (same as today's practice). Avoid screenshots that show credentials or customer PII.

---

## 5. Risky actions — permission policy

| Action | dev2 (`mcphqdev2.digioh.com`) | Production (`account.digioh.com`) |
|---|---|---|
| Impersonate test accounts listed in the plan | Allowed | Allowed (read/test only) |
| Toggle Admin > Edit User feature flags | **Allowed** (revert at cleanup) | Not allowed → skip test, ask |
| Create / duplicate / edit test campaigns | Allowed | Only on accounts the plan names (45281, 46388), else skip |
| Publish to QA / publish live | Publish-to-QA allowed | Live publish not allowed unattended → skip, ask |
| Agent chat "approve write" | Allowed | Only on the plan's test account |
| Login / entering passwords, creating external accounts | Never — Chrome must already be logged in | Never |

These standing permissions must be written into the playbook, scoped exactly like this table.

---

## 6. Rollout

1. **Prereqs:** Hussain fixes the QA Type rule. ~~Snipboard spike~~ done. Confirm Claude in Chrome stays connected while the routine runs (app must be open; routines only fire while the desktop app is running).
2. **Hand run first:** run the playbook manually on one task (e.g. ID-15765 Reporting dropdown — HQ UI, dev2, low risk) to pre-approve tool permissions and check output.
3. **Live from day one, review-gated by design:** because the agent only writes a `QA run by Claude` sub-task and never mentions anyone or touches the parent, Hussain's review is built in. Hussain does his own QA in parallel for ~2 weeks and compares per finding: matched / missed / false positive / skipped.
   Pilot set: ID-15778, 15690, 15443, 15675, 15672, 15780, 15765.
4. **Decide after the pilot:** keep as assistant (Hussain still QAs), or let Hussain skip his own pass and just review/forward the sub-task.

Success bar (suggested): ≥90% of Hussain's findings matched, ≤1 false positive per task, every skipped test explicitly listed.

---

## 7. Where a human stays in the loop
- Bug vs confusing plan (Pablo overrules several findings).
- AI-wording tolerance in Agent chat tests.
- Hardware / other-browser / real-device tests.
- Production live publishes and anything on real customer stores.
- Closing the task.

---

## 8. Decisions

Decided (2026-10-14):
- Separate routine `core-qa-sentinel`.
- Output = `QA run by Claude <date>` sub-task with findings; agent **never mentions Pablo** or anyone; Hussain posts his own findings on the parent.
- Model: Sonnet.
- Schedule: same window/cadence as qa-sentinel, offset 1.5 h (`47 1-22/3 * * *` PKT, 14:00–24:00 UTC) to avoid browser collision; core skips if qa-sentinel is still running.
- Sub-task unassigned.
- Screenshots: snipboard.io via `upload_image` (verified).
- Risky actions: allowed on dev2 per §5; production changes skipped.

Open:
1. What to do with Final QA's human layer (one-line comment + Issues & Queries sub-tasks) — automate later, separately?
2. Classification of `1218717326667659` and the TEMPLATE task.
