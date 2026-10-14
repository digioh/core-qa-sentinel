# Product QA — core-qa-sentinel

A Claude Code desktop routine that QAs **Product / Feature QA** tasks (HQ, Digioh Agent, Smart Rules, display rules editor, runtime) from the Asana **QA** board. It runs the engineer's test plan in Chrome and posts findings in Hussain's style as a sub-task.

Sister project: [`qa-sentinel`](../qa-sentinel) handles client **Final QA** (quizzes and campaigns). The two never touch each other's tasks. They are split by the Asana **QA Type** field.

> Status: **routine created 2026-10-14, enabled**. The playbook is [core.md](core.md); design and decisions are in [PLAN.md](PLAN.md).

---

## How it works

```
Every 3 h (offset from qa-sentinel), 14:00–24:00 UTC only
  │
  ├─ qa-sentinel still running?  → stop silently
  ├─ Find ONE task on the QA board:
  │     QA Type = Product / Feature QA, incomplete,
  │     no "QA run by Claude" sub-task for this round,
  │     not gated (parent ready / release date reached)
  │     none → stop silently
  │
  ├─ Read the task, all comments, the parent ticket and sibling QA tasks
  ├─ Parse "Test 1 … Test N" with PASS: / FAIL: lines → plan.json
  ├─ Run each test in Chrome (Claude in Chrome, Hussain's logged-in session)
  │     screenshot every deviation → upload to snipboard.io → URL
  ├─ Clean up (revert flags, restore rules, archive copies)
  ├─ Create sub-task "QA run by Claude YYYY-MM-DD" (open, unassigned)
  │     with one findings comment, then re-fetch to verify
  └─ Save everything to reports-core/<ID>/
```

Hussain reviews the sub-task, then posts his own findings on the parent and mentions Pablo himself.

---

## What it posts

Plain text with no @-mentions, one line per finding:

```
Test 2, point 3: When we click escape, it closes the agent window. https://snipboard.io/xxxx.jpg
Test 5: Is this expected? The "How often" row moves to the top after save. https://snipboard.io/yyyy.jpg
Skipped: Test 4 (needs a physical microphone), Test 7 (gated on 10/14 production release).
Passed: Test 1, 3, 8.
```

When every test passes, the comment is `It is looking good.`, followed by the Skipped line if any tests were skipped.

---

## Hard rules

- **Only creates** a `QA run by Claude <date>` sub-task and its comment. It never edits, moves or completes the parent, never sets fields, and never comments on the parent.
- **Never @-mentions anyone**, Pablo included.
- On **dev2 (`mcphqdev2.digioh.com`)** it may impersonate test accounts, toggle Admin > Edit User flags, create and duplicate test campaigns, and publish to QA. It reverts all of this at cleanup.
- On **production (`account.digioh.com`)** it makes no flag changes and no live publishes. Any test that needs them is skipped and listed.
- It never logs in or enters passwords. Chrome must already be signed in.
- It skips and lists any test that needs a microphone, OS clipboard paste, Firefox or Safari, a real phone, a Shopify admin login, Google Ads account creation, or engineering-only data.
- It hard-stops at about 80 minutes. Whatever is unfinished is marked NOT RUN, and it still posts.

---

## Routine settings

| Setting | Value |
|---|---|
| Routine ID | `core-qa-sentinel` (task file: `%USERPROFILE%/.claude/scheduled-tasks/core-qa-sentinel/SKILL.md`) |
| Model | Sonnet (`model: claude-sonnet-5-5` in SKILL.md) |
| Cron | `47 1-22/3 * * *` (local PKT; fires 19:47, 22:47, 01:47, 04:47 within the window) |
| Time gate | 14:00–24:00 UTC |
| qa-sentinel | `17 */3 * * *`, same window. The 1.5 h offset avoids browser collisions |
| Browser | Claude in Chrome |
| Requires | Claude desktop app open, Chrome connected and signed in to HQ (dev2 + prod) |

---

## Folder layout

```
core-qa-sentinel/
├── README.md            this file
├── PLAN.md              design, findings, decisions
├── core.md              the playbook the routine reads
└── reports-core/
    └── ID-15765/
        ├── plan.json        parsed test list
        ├── report.txt       exact text posted to the sub-task
        ├── *.png            full-size screenshots
        ├── run-log.md       per-test result, reasons, snipboard URLs
        └── retest-YYYY-MM-DD/   later rounds, same files
```

`reports-core/` should be **git-ignored**. Screenshots and logs can contain account IDs, customer campaign data and internal HQ screens.

---

## Asana reference

| Thing | GID |
|---|---|
| QA project | `1203348184098714` |
| Product Release QA project (archived) | `1209283663848605` |
| QA Type field | `1219235193315472` |
| QA Type: Product / Feature QA | `1219235193315473` |
| QA Type: Final QA | `1219235193315474` |
| Issues & Queries field | `1208918841853152` |

---

## Screenshots: snipboard.io

Verified on 2026-10-14 with a test upload: https://snipboard.io/xeA7lR.jpg

1. Take a full-size screenshot (scale 1).
2. Upload it with `upload_image` to the "pick a file" input on a snipboard.io tab.
3. Wait about 3 seconds, then read the image URL from the snipboard tab's address bar.
4. Navigate the tab back to `https://snipboard.io` for the next upload.

If snipboard is down, it posts a text-only finding and keeps the PNG in `reports-core/<ID>/`.

---

## Running it

- **By hand (first time):** open a Claude Code session in this folder and say "follow core.md for task <gid>". The first run also pre-approves tool permissions, which is what keeps unattended runs from stalling.
- **Scheduled:** after the routine is created, it shows in the desktop app's **Routines** sidebar. Use "Run now" there to trigger it on demand.
- **Pause:** flip it off in the Routines sidebar.
