# CLAUDE.md — setting up the Swimlane Board

You are helping someone set up a kanban board with epic swim lanes. The board is one HTML file (`board.html`) published as an artifact. Its data lives in the person's own Airtable base, reached through the Airtable connector. Your job is to create that base, point the board at it, publish the board, and hand it over.

Read this whole file before acting. `README.md` is the human's guide; you do not need it to do the setup.

## The kit

| File | Use |
|---|---|
| `board.html` | The board. Edit only its `CONFIG` block and `<title>`. |
| `themes/*.css` | One file per theme: `scifi.css` (the default) and `agency.css`. The board loads them, so they are published with it. |
| `sample-data/schema.json` | Table and field definitions, in the shape `create_base` takes. |
| `sample-data/epics.csv`, `statuses.csv`, `tickets.csv`, `comments.csv`, `team.csv` | Sample project "Website Relaunch". Column headers are the Airtable field names. The sample team is fictional (`example.com`). |
| `preview.png` | Screenshot of the finished board. |

## Step 0. Which job is this?

- **New board.** The person wants their own board. `CONFIG.BASE_ID` in `board.html` is empty. Do steps 1 to 6.
- **Joining an existing board.** Someone shared a base with them and sent a `board.html` whose `CONFIG.BASE_ID` is already filled in. Do step 1, then skip to "Joining an existing board".
- **Upgrading an older board.** The person already runs a board from an earlier version of this kit and wants the new features (Team sign-in, due dates, watchers, the Blocked column). Do step 1, then skip to "Upgrading an older board".

If it is unclear, ask.

## Step 1. Check the Airtable connector

Look for Airtable connector tools in your tool list (`list_workspaces`, `create_base`, `list_tables_for_base`, `list_records_for_table`, `create_records_for_table`, `update_records_for_table`, `delete_records_for_table`). Load them if they are deferred.

If there are none, stop. Tell the person to add the Airtable connector in Claude and grant it access to their workspace (README, setup step 2), then come back. Do not try to set anything up without it.

## Step 2. Ask four things

Ask once, together:

1. **Project name.** Shown above the board title. Sample: "Website Relaunch".
2. **Ticket key prefix.** Two to five capital letters. Sample: `WEB`.
3. **Sample project or empty board?** The sample gives them something to click on straight away. Empty means their own epics, which you then ask for (a name each is enough; you invent a short key and pick a colour).
4. **Who is on the team?** A full name and email for each person, starting with themselves. The email must be the one they sign in to Claude with: the board recognises each viewer by it. Anyone not on the list can view and move tickets but not comment or create them.

If they say "just use the sample", use project name "Website Relaunch" and prefix `WEB`. Their own name and email are still needed, or they will be a guest on their own board.

## Step 3. Create the base

1. Call `list_workspaces`. One workspace: use it. Several: ask which.
2. Tell the person the name of the base you are about to create and where, then call `create_base` with the `tables` array from `sample-data/schema.json`. Name the base after the project, for example "Website Relaunch Board".
3. Keep the result. It holds the base ID (`app…`) and every table and field ID, which you need for inserting records.

Rules that matter:

- **Table and field names must stay exactly as in `schema.json`.** The board finds tables and fields by name each time it loads. `Tickets`, `Comments`, `Epics`, `Statuses`, `Team`; fields such as `Key`, `Epic Key`, `Show As Column`, `Blocked Reason`.
- For an empty board with the person's own epics, replace the `Epic Key` choices in `schema.json` with their epic keys before creating the base.
- If `create_base` is unavailable or refused, say so and stop. The fallback is manual: the person creates a base in Airtable and imports the five CSVs as tables named as above. The board tolerates plain-text fields.

## Step 4. Load the data

Insert with `create_records_for_table`, keyed by **field ID** (map each CSV header to its ID from the `create_base` result).

- **Statuses: always.** All six rows from `statuses.csv`. The board needs them.
- **Team: always.** The people from step 2, each with `Name`, `Email` and `Active` ticked. With the sample project, also load the three rows from `team.csv`, since the sample tickets are assigned to them.
- **Epics:** the sample rows, or the person's own. Each needs `Epic Key`, `Epic`, `Color` (hex), `Lane Order`.
- **Tickets and Comments:** sample only.

Value rules:

- Leave out empty cells. Do not send empty strings.
- `Order`, `Lane Order`, `WIP Limit`, `Rank` are numbers.
- `Show As Column` and `Active` are booleans: `TRUE` becomes `true`; for `FALSE`, leave the field out.
- Single-select fields (`Epic Key`, `Type`, `Status`, `Priority`, `Category`, `Blocked Reason`) take the option name as a plain string.
- `Due Date` is a date as `YYYY-MM-DD`.
- If the person chose their own prefix with the sample data, rewrite `WEB-` to their prefix in `Key`, `Related`, `Blocked On`, and the comments' `Ticket Key`.
- Send at most 10 records per request.

Then verify: count the records in each table and compare with what you sent. Sample totals are 6 statuses, 4 epics, 14 tickets, 3 comments, and 3 team members plus the person's own.

## Step 5. Configure and publish the board

1. Copy `board.html` and the `themes/` folder to your working folder, keeping `themes/` next to `board.html`.
2. In the `CONFIG` block near the top of the `<script>`, set:
   - `BASE_ID` to the new base's ID
   - `PROJECT` to the project name
   - `KEY_PREFIX` to the prefix
   - `SERVER` only if the person's Airtable connector appears under a different name than "Airtable"
3. Set `<title>` to the project name followed by "Board", for example `<title>Website Relaunch Board</title>`.
4. Change nothing else. The file has no doctype, `<html>`, or `<body>` tags on purpose; the artifact publisher adds them.
5. Publish it as a plain HTML artifact (not a typed artifact such as Dashboard or Design). Publish every theme file listed in `CONFIG.THEMES` alongside it with `files`, at the same relative path, for example `{"themes/scifi.css": "themes/scifi.css"}`. Without them the board falls back to a plain look. Use these capabilities:

```json
{
  "mcp": {
    "servers": [
      {
        "server": "Airtable",
        "tools": [
          "list_tables_for_base",
          "list_records_for_table",
          "update_records_for_table",
          "create_records_for_table",
          "delete_records_for_table"
        ]
      }
    ]
  },
  "user": { "scopes": ["profile", "email"] }
}
```

`server` must match the connector's display name, and match `CONFIG.SERVER`. If any of the five tools is missing from the person's connector, tell them which, since that feature will not work.

The `email` scope is what lets the board match a viewer to the Team table. Where Claude does not share a viewer's email, the board falls back to their exact full Claude account name.

You cannot open the published board as the viewer, so you cannot confirm it loads. Say so plainly in the handover.

## Step 6. Hand over

Tell the person, briefly:

1. The board is published; open it and **allow Airtable** when asked. The badge should read "Base linked".
2. The round avatar at the top right should show their initials. Clicking it says who the board thinks they are. A `?` means their Claude account did not match the Team table: check the email they gave you.
3. Give them the link to the Airtable base.
4. Give them the configured `board.html` as a file, and say it is the one to send teammates together with the `themes/` folder.
5. To share: add each person to the **Team** table with the email they use for Claude, invite them to the base in Airtable as **Editor**, send them that `board.html`, the `themes/` folder and `CLAUDE.md`, and each person asks their own Claude to "join an existing board". Point them to the README section "Sharing the board with other people".
6. What you could not verify: that the board loads for them. Ask them to tell you what the badge says if it does not turn to "Base linked".

## Joining an existing board

The person has a `board.html` with `CONFIG.BASE_ID` already set.

1. Check the connector (step 1).
2. Call `list_tables_for_base` with that base ID. If it fails or returns nothing, their Airtable account or their connector has no access to the base. Tell them to accept the Airtable invitation, then re-authorize the Airtable connector and include that base. Stop until that is done.
3. Publish `board.html` unchanged, with its theme files and the capabilities in step 5. If the person did not receive the `themes/` folder, use the ones from this kit, and tell them any custom theme named in `CONFIG.THEMES` is missing.
4. Hand over as in step 6, items 1 and 2. If the avatar shows `?`, the board owner has not added them to the Team table yet; they can view and move tickets meanwhile. If their Airtable permission is Commenter or Read-only, tell them the board will open and refuse changes.

Create nothing in Airtable during a join.

## Upgrading an older board

The person's base was built from an earlier version of this kit. The new `board.html` works on it as it is: features whose table or fields are missing stay hidden. To turn them on, add what is missing. Call `list_tables_for_base` first and add only what is not already there. Tell the person what you are about to add before you add it.

1. **Ticket fields**, with `create_field` on `Tickets`, as defined in `schema.json`: `Due Date` (date), `Watchers` (text), `Blocked Reason` (single select: "Blocked by Ticket", "Waiting on Team Member"), `Blocked On` (text).
2. **The Blocked status.** Add "Blocked" to the choices of the `Status` field, then a row in `Statuses`: Status "Blocked", Category "In Progress", Show As Column ticked, and an `Order` that puts it after In Progress. Renumber the rows after it so the orders stay whole numbers.
3. **The Team table**, with `create_table`: `Name` (text), `Email` (email), `Active` (checkbox). Ask who is on the team (step 2, item 4) and add them. Use the names already on tickets where they match, so existing assignments line up.
4. **The board.** Take the new `board.html`, copy `CONFIG` values and `<title>` from their old one (`BASE_ID`, `PROJECT`, `KEY_PREFIX`, `SERVER`, `POLL_SECONDS`), and republish it to their existing artifact with the theme files and the step 5 capabilities. Note the `user` scopes now include `email`. Without that scope the board matches people by account name only.
5. Hand over as in step 6. The **I am** menu is gone; the avatar replaces it.

## After setup: working with the board

**Republishing.** When the person asks for a change to the board, edit the same file and publish to the same artifact so the link stays the same, with the theme files. Give them the new `board.html` (and any changed theme files) for teammates.

**Themes.** The look lives in `themes/<name>.css`; `board.html` holds only layout. A theme sets the variables in the `:root` block of `themes/scifi.css`, which documents each one, and may add its own rules after it.

- **Changing the look** ("make the accent amber"): edit the theme file, not `board.html`.
- **A new theme:** copy `themes/scifi.css` to `themes/<name>.css` (lowercase letters, digits, hyphens), set every variable, add `<name>` to `CONFIG.THEMES`, and publish the new file with the board. A **Theme** menu appears in the toolbar once `CONFIG.THEMES` has two or more names. Each viewer's choice is remembered in their browser. A theme that differs a lot from scifi, as `agency.css` does, can restyle any part of the board in rules after `:root`, because theme files load after the board's own rules.
- **The default theme** is `CONFIG.THEME`. It must be in `CONFIG.THEMES`. When you change it, also change the `href` of the `themeCss` link near the top of `board.html` to match.
- Fonts load with `@import` from Google Fonts at the top of the theme file. The artifact allows stylesheets from no other host.

**Working on tickets in chat.** You can read and change the base directly. Keep to the board's conventions so both stay consistent:

- **Key:** prefix, hyphen, number. A new ticket takes the highest existing number plus one.
- **Epic:** set both `Epic Key` and `Epic` (the name) on the ticket. A new epic also needs a row in `Epics`.
- **Status:** must be a `Status` value from the `Statuses` table.
- **Blocked:** a ticket in Blocked needs both `Blocked Reason` ("Blocked by Ticket" or "Waiting on Team Member") and `Blocked On` (a ticket key, or a team member's name). When a ticket leaves Blocked, clear both.
- **People:** `Assignee`, `Watchers` and `Blocked On` use names from the `Team` table. A first name that only one team member has also counts as that person.
- **Watchers:** names, semicolon-separated.
- **Due Date:** `YYYY-MM-DD`. Done and Won't Do tickets never show as overdue.
- **Team:** one row per person, with the email they use for Claude. Untick `Active` rather than deleting someone, so their name stays on old tickets.
- **Rank:** a number; lower sorts higher within a cell. A new ticket takes the highest rank plus one.
- **Labels:** one text field, semicolon-separated.
- **Related:** ticket keys, semicolon-separated. Store a link on one of the two tickets; the board shows it on both.
- **Comments:** one row in `Comments` with `Comment ID` (any unique string), `Ticket Key`, `Author`, `Created At (UTC)` as an ISO 8601 timestamp, and `Body`.
- **Deleting a ticket:** also delete its rows in `Comments` and remove its key from other tickets' `Related`.

**Renames break the board.** If the person asks to rename a table or one of the fields above, explain that the board looks them up by name, and offer to update `TABLE_NAMES` or the field references in `board.html` in the same change.

## Guardrails

- Ask before creating the base, and before deleting or bulk-changing anything in it.
- Team emails are personal data. Use them only in the person's Airtable base, never in `board.html` or anything published.
- Never put ticket data into `board.html`. The page reads everything from Airtable.
- Create one base per setup. If a step fails partway, tell the person what exists and what does not, and continue from there instead of starting a second base.
- If something in these instructions does not match what the tools actually do, trust the tools, tell the person what differed, and carry on with their goal.
