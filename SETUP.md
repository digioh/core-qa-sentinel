# Setup — running core-qa-sentinel by hand

This gets you from a fresh clone to a supervised first run. Nothing here is tied to one person: the agent works as **you** (your Asana, Gmail and Chrome sessions) and only picks up QA tasks **assigned to you**.

## What it does and how

1. Finds one Product / Feature QA task assigned to you on the Asana QA board (oldest first, Upcoming or Ready).
2. Reads the task, every comment, the engineering ticket and sibling QA tasks, then turns the test plan into a list of tests.
3. Runs each test in your Chrome, following the plan step by step and comparing against its PASS / FAIL lines. It takes a screenshot of every problem and uploads it to snipboard.io.
4. Undoes anything it changed (flags, test campaigns) when done.
5. Creates a `QA run by Claude <date>` sub-task under your QA task with one plain-text comment listing findings, queries, skipped tests and passed tests, then moves your QA task to In progress.

It is an assistant, not a replacement for you. **It only reports; it never tells engineering anything.** You confirm its findings, then post your own. Full rules are in [core.md](core.md); overview in [README.md](README.md).

---

## 1. Prerequisites

- **Claude desktop app** (Code tab) on Windows or macOS, signed in to your own Claude account.
- **Google Chrome** with the **Claude in Chrome** extension installed and connected.
- Chrome already **signed in** to HQ on both hosts: `https://mcphqdev2.digioh.com` (dev2) and `https://account.digioh.com` (production), with an account that can impersonate test accounts and edit user flags on dev2. The agent never logs in or types passwords.
- Access to the Asana **QA** board and at least one Product / Feature QA task **assigned to you**.
- Git.

## 2. Clone

```bash
git clone https://github.com/digioh/core-qa-sentinel.git
cd core-qa-sentinel
```

Open that folder as the project in the Claude desktop app (Code tab). The agent uses the session's working directory, so any location works. Run output goes to `reports-core/<ID>/`, which is created automatically and is git-ignored.

## 3. Connectors (live in your Claude account, not in the repo)

Connect these in the desktop app (Settings > Connectors) with **your own** accounts:

| Connector | Used for |
|---|---|
| **Asana** | Reading the QA board and tasks, creating the run sub-task and its comment, moving the QA task to In progress. Identifies you with `get_me`. |
| **Gmail** | Sending you one "needs attention" email when a task is blocked. Use the same person as your Asana account, since the email goes to your Asana address. |
| **Claude in Chrome** | Driving the browser. Must be connected, see section 1. |

Optional: the **scheduled-tasks** and **session management** tools are used for a collision check against a sister routine (skipped automatically if it doesn't exist) and a token-usage line in the run notes.

### `.mcp.json` (only if you can't use the Asana connector)

The connector is the normal route and needs no file. If you must use a personal access token instead:

```bash
cp .mcp.json.example .mcp.json
```

Paste **your own** Asana personal access token over the placeholder. `.mcp.json` is git-ignored: never commit it, never paste the token into chat, docs or a ticket. This token route is untested with `core.md`, whose tool names assume the connector, so prefer the connector.

## 4. First supervised run

Do the first run yourself, watching, so you can approve tool permissions once. Unattended runs stall on permission prompts otherwise.

1. Make sure you have a Product / Feature QA task assigned to you in **Upcoming** or **Ready** on the QA board. (Other people's tasks are ignored.)
2. In the Code tab session for this folder, paste exactly:

   ```
   Read core.md and follow it exactly to QA one task. Use the Asana connector for Asana and Claude in Chrome for the browser. Only pick up tasks assigned to me. Stop silently if there is no eligible task.
   ```

   To target one task: `Read core.md and follow it exactly for Asana task <task gid>.`
3. When a permission prompt appears, choose **Always allow** for each of these (the agent can't run without them):
   - **Asana:** `get_me`, `get_tasks`, `get_task`, `get_task_stories`, `get_my_tasks`, `create_tasks`, `add_comment`, `update_tasks`
   - **Gmail:** `send_message`
   - **Claude in Chrome:** `tabs_context_mcp`, `tabs_create_mcp`, `tabs_close_mcp`, `navigate`, `computer`, `read_page`, `find`, `get_page_text`, `form_input`, `javascript_tool`, `read_console_messages`, `resize_window`, `upload_image`, `browser_batch`
   - **Files in this folder:** write, edit and shell commands inside the repo (for `reports-core/`)
   - **Optional:** `list_task_runs` and `get_usage`
4. Check the result:
   - A `QA run by Claude <date>` sub-task exists under your QA task, with one findings comment.
   - The QA task moved to In progress on the QA board (and in your My Tasks, if you have an "In progress" section there).
   - `reports-core/<ID>/` holds `plan.json`, `report.txt`, `run-log.md` and screenshots.
   - Nothing was changed on the engineering ticket (the QA task's parent).

**Review is your job, every time.** Open the sub-task and read each finding in detail, with its screenshot. Reproduce anything you're unsure of yourself, and check that "Skipped" and "Not run" tests are really untested (the agent never reports those as passed, but you decide what to do about them). Only after you have confirmed a finding, post it yourself on the engineering ticket in your own words and mention the engineer. Drop false positives and fix wrong wording. The agent never posts to the ticket and never mentions anyone, so nothing reaches engineering until you send it. Make the final call on whether the task is done.

## 5. Day-to-day running

- Open a session in the repo folder and paste the prompt from section 4 whenever you want one task QA'd. One task per run, oldest first. There is no time-of-day restriction.
- A run takes up to about 80 minutes. Unfinished tests are marked NOT RUN and it still posts.
- Keep Claude desktop open and Chrome connected for the whole run.
- Blocked tasks (for example Chrome signed out of dev2) are skipped, noted in `reports-core/<ID>/ATTENTION.md`, and you get one email.
- Safety limits (see [core.md](core.md)): on production it makes no flag changes and no live publishes; on dev2 it reverts every change at cleanup; it never deletes, never @-mentions anyone, never logs in.

## 6. Scheduling it yourself (optional)

Not required, and you still review every sub-task. To run it automatically, create a scheduled task in the desktop app whose prompt is the section 4 prompt, with a working directory of your clone. Pick your own cadence and keep the desktop app open. Don't copy someone else's routine: it lives in their account.

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Run stops silently, nothing posted | No eligible task: none assigned to you, wrong QA Type, not in Upcoming/Ready, already has a `QA run by Claude` sub-task, or gated | Check the task on the board; see `reports-core/<ID>/gated-<date>.md` |
| Your task is ignored | Assigned to someone else or unassigned | Assign it to yourself |
| `ATTENTION.md` written, email received | Chrome signed out of dev2 or production | Sign in to that host in Chrome, run again |
| "Chrome not connected" / `tabs_context_mcp` fails | Claude in Chrome extension not connected | Open Chrome, reconnect the extension, retry |
| Asana tools missing or "not authorized" | Asana connector not connected | Reconnect it in Settings > Connectors |
| No blocker email arrives | Gmail connector not connected, or Asana email differs from your mailbox | Connect Gmail; check the email on your Asana profile |
| Run stalls on a permission prompt | Tool not on "Always allow" yet | Approve with Always allow (section 4) |
| Task not moved in My Tasks | You have no "In progress" section in My Tasks | Add one, or move it by hand. The board move still happens |
| Screenshot shows no link | snipboard.io upload failed twice | PNG is kept in `reports-core/<ID>/`; attach it by hand |
| Sub-task missing after run | Post failed | See `reports-core/<ID>/POST_FAILED.md` for the comment text |

## 8. Contact

Questions about the agent, its rules or the board setup: **Khizar Abbas**.
