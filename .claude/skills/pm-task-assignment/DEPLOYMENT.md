# Deployment — Installing PM Task Assignment for a New PM

## Quick reference

For each PM you deploy this skill to:

1. Copy the entire `PM Automation MVP 1` folder
2. Edit `config.md` — replace 2 fields with the new PM's Notion page
3. (Optional) Edit `config.md` — set the new PM's name and email for clarity
4. Send the edited folder to the PM (Slack, Orbit, email, USB drive, however)
5. The PM installs the folder in their Cowork skills directory
6. The PM runs `PM Task Assignment, run morning` — first-run setup kicks in

That's it. Per-PM configuration after that point happens through the conversational first-run setup.

---

## Step-by-step

### Step 1 — Copy the folder

Make a fresh copy of the entire `PM Automation MVP 1` folder. Don't edit the master.

### Step 2 — Edit `config.md`

Open `config.md` in any text editor. Find the section labeled `## Notion parent page`. Replace these two values:

```
DEFAULT_NOTION_PARENT_PAGE_ID = 34c3846840c8808ea886e7e6df15e2af
DEFAULT_NOTION_PARENT_URL    = https://www.notion.so/Test-Run-Ishant-PM-MVP-1-34c3846840c8808ea886e7e6df15e2af
```

Replace with the new PM's Notion parent page values:

- `DEFAULT_NOTION_PARENT_PAGE_ID` — the 32-character page ID (with or without dashes). Find it in the Notion URL: `https://www.notion.so/Page-Title-<32 char ID>?source=copy_link`.
- `DEFAULT_NOTION_PARENT_URL` — the full URL of the page.

Save the file.

### Step 3 (optional) — Edit `config.md` for PM-specific clarity

Find the `## Optional defaults (for clarity during installation)` section. Replace:

```
DEFAULT_PM_NAME  = (set per installation)
DEFAULT_PM_EMAIL = (set per installation)
```

With the actual PM's name and primary email. This is optional — the skill will infer identity from Preferences after first-run setup. But filling these in makes the config self-documenting.

### Step 4 — Send the folder to the PM

Choose any delivery method:

- Zip the folder and Slack it
- Attach to an Orbit task
- Email the zip
- Hand-deliver via USB

The folder is small (\~250KB) so any method works.

### Step 5 — PM installs

The receiving PM:

1. Unzips and copies the folder into their Cowork skills directory (path varies by Claude version — typically under `%APPDATA%\Claude\skills\` on Windows or `~/Library/Application Support/Claude/skills/` on macOS).
2. Restarts Cowork or Claude so the skill is registered.
3. Verifies installation by typing in Claude: `PM Task Assignment, run morning`.

### Step 6 — PM completes first-run setup

The PM types `PM Task Assignment, run morning`. Because their Notion parent doesn't have a `Preferences` page yet, the skill detects first-run and asks the setup questions conversationally:

1. Identity (name + Orbit user ID + email)
2. Email aliases (with WLIQ alias example)
3. Morning collection run time (options-based)
4. Execution run time (options-based)
5. Escalation backup (options-based, list of WLIQ team members)
6. Account managers (one per AM)
7. Default email preferences
8. Default Slack preferences
9. Always-include rules (free text)
10. Tone samples (skippable)

The skill writes the answers to a new `Preferences` sub-page on the parent and registers scheduled tasks. From day 2 onward, the skill runs automatically at the configured times.

---

## Pre-deployment checklist (before sending the folder)

- [ ] `config.md` has the new PM's Notion page ID and URL
- [ ] The Notion page exists and is accessible to the PM via their Notion authentication
- [ ] (Optional) `config.md` has the PM's name and email pre-filled
- [ ] Folder is zipped or otherwise prepped for delivery

## Post-deployment checklist (after the PM installs)

- [ ] PM successfully invokes `PM Task Assignment, run morning` and sees the first-run setup flow
- [ ] PM completes the 10 setup questions
- [ ] `Preferences` sub-page is created on the PM's Notion parent
- [ ] Scheduled tasks for Mode 1, Mode 2, and monthly archival are registered (visible in Cowork's scheduled-tasks dashboard if available)
- [ ] First scheduled morning run completes the next day
- [ ] PM reviews the first morning queue and provides feedback

---

## Troubleshooting

**"The skill says it can't find the Notion parent page."**
The page ID in `config.md` is wrong, or the PM's Notion authentication doesn't have access to that page. Verify both.

**"First-run setup keeps re-running every time the skill is invoked."**
The skill expects a `Preferences` sub-page on the parent. If first-run setup completed but the Preferences page wasn't created, check the Notion MCP authentication and the parent page's permissions. The skill needs write access.

**"The skill is calling Pipedrive / another connector."**
Bug. The source allowlist is supposed to prevent this. Capture the chat transcript and share with the maintainer. The fix is in `config.md` and SKILL.md — confirm both files contain the allowlist clause.

**"Scheduled runs aren't firing."**
Check the Cowork scheduled-tasks dashboard. The skill registers three scheduled tasks during first-run setup. If they're missing, the scheduled-tasks MCP may have been unavailable at setup time. Manually fire `PM Task Assignment, run morning` once — the skill re-registers schedules at the end of every Mode 1 run.

---

## Maintenance — updating the skill across all installed PMs

If the skill files are updated (e.g., bug fix, new feature):

1. Update the master folder with the new files.
2. For each PM who has the skill installed: send them the updated files (they replace their existing skill folder).
3. **Important:** preserve the per-PM `config.md` — that file is unique per installation. Don't overwrite it during updates.

For larger version changes (v1.0 → v2.0), document in a `CHANGELOG.md` (not yet created — add when the first update happens).

---

## What's NOT in deployment scope

- Setting up Cowork itself
- Authenticating MCPs (each PM does this through their own Cowork account)
- Creating Notion pages (each PM creates their own parent page in Notion before installation)
- Configuring scheduled tasks manually (the skill does this during first-run setup)
