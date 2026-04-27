# Survey Reminder Flows

Microsoft Power Automate flows that run a multi-survey research campaign end-to-end:

1. **Auto-seed flow** — watches an Excel distribution list in SharePoint; on every save it creates `SurveyAssignments` rows for any newly-added emails (one per survey) and emails brand-new participants a welcome message with the first survey link and task instructions.
2. **Reminder flow** — runs daily; on each survey's reminder date it sends every assignee who still has outstanding surveys a single consolidated email listing all of them with their task instructions.
3. **Ingestion flow** — listens to Microsoft Forms responses and marks the matching assignment row `Completed = Yes` so reminders stop for that person.

A manual seeding flow (`excel-to-assignments-flow.json`) is also kept in the repo as a backup / fallback — useful if you'd rather seed assignments by clicking Run once per survey instead of letting file saves trigger automatically.

## Campaign model

Each survey has its own **release date** (when it's sent) and its own **reminder date** (when a nudge goes out to stragglers). All surveys share one **research end date** — the global deadline. Example from the current campaign:

| Survey | Release date | Reminder date |
| ------ | ------------ | ------------- |
| Survey 1 | Tue 28 Apr | Thu 30 Apr |
| Survey 2 | Fri 1 May  | Sun 3 May  |
| Survey 3 | Mon 4 May  | Wed 6 May  |
| Survey 4 | Thu 7 May  | Sun 10 May |

**Research end date:** Sun 10 May, 11:45 PM.

**How the reminder works:** every day at 9 AM the reminder flow looks for rows where `Completed = No`, `ReminderDate` has arrived, `LastReminderSent` is null, and the research period is still open. For each unique assignee among those rows, it fetches *all* of their currently-outstanding surveys (released but not completed) and sends one email listing them. Each trigger row gets `LastReminderSent` stamped, so the flow won't re-fire on subsequent days — but later reminder dates (and more outstanding surveys) will still trigger fresh emails.

Example: Jane has all 4 surveys assigned and completes none.

- **30 Apr** — Survey 1 reminder fires. Email lists [Survey 1].
- **3 May** — Survey 2 reminder fires. Email lists [Survey 1, Survey 2] (both released, still outstanding).
- **6 May** — Survey 3 reminder fires. Email lists [Survey 1, Survey 2, Survey 3].
- **10 May** — Survey 4 reminder fires. Email lists all four.
- **After 10 May 11:45 PM** — no more reminders, regardless of completion state.

## Prerequisites

- A Power Automate licence (Free tier works).
- A SharePoint site you can create a list and document library in.
- An Outlook / Office 365 mailbox the flow can send from.
- A Microsoft Form per survey (4 in this campaign).

## 1. Create the SharePoint list

Create a list called **`SurveyAssignments`** with these columns:

| Column name         | Type                      | Notes                                                            |
| ------------------- | ------------------------- | ---------------------------------------------------------------- |
| `Title`             | Single line of text       | Survey name, e.g. `Survey 1` (built-in column).                  |
| `AssigneeEmail`     | Single line of text       | Email address of the person to remind.                           |
| `ReleaseDate`       | Date and Time (Date only) | Day the survey is sent / becomes available.                      |
| `ReminderDate`      | Date and Time (Date only) | Day the reminder should fire for this survey.                    |
| `ResearchEndDate`   | Date and Time             | Global campaign deadline. No reminders fire after this.          |
| `Completed`         | Yes/No                    | Default `No`. Flipped to `Yes` by the ingestion flow.            |
| `SurveyLink`        | Hyperlink                 | URL to the survey (Forms, Qualtrics, etc.).                      |
| `LastReminderSent`  | Date and Time             | Written by the reminder flow. Leave blank initially.             |
| `CompletedDate`     | Date and Time             | Written by the ingestion flow on submission.                     |
| `ResponderName`     | Single line of text       | Written by the ingestion flow on submission.                     |
| `Answer1`           | Multiple lines of text    | Answer to question 1. Rename to match the question if you like.  |
| `TaskInstructions`  | Multiple lines of text    | Task instructions accompanying this survey. Written by the seeding flow; surfaced in welcome and reminder emails. |

## 2. Build the auto-seed flow (Distribution.xlsx → SharePoint)

This flow watches `Distribution.xlsx` and, on every save, creates assignment rows for any newly-added emails and emails them a welcome message. Existing participants are skipped via dedup. All 4 surveys are assigned to every participant — even if they join after some surveys' release dates, so they catch up via the reminder flow.

### Prepare the Excel file

1. Save the workbook in a SharePoint document library (e.g. `Documents/SurveyOps/Distribution.xlsx`).
2. The data must be a real Excel **table**, not a plain range. Select the data → **Insert → Table** → check "My table has headers" → name the table `Distribution` under **Table Design**.
3. Required column: `Email`. Anything else is ignored.

### Build it (Automated cloud flow)

**Trigger — SharePoint → When a file is created or modified (properties only)**
- Site Address: your SharePoint site.
- Library Name: the document library that holds `Distribution.xlsx` (usually `Documents`).
- Folder (advanced options): point at the folder containing the file, e.g. `/SurveyOps`.

**Action 1 — Condition `Is_distribution_file`** — only proceed if the saved file is `Distribution.xlsx`
- Left: Expression `triggerOutputs()?['body/{FilenameWithExtension}']`
- Operator: `is equal to`
- Right: `Distribution.xlsx`
- **If no** branch: add a **Terminate** action with status `Succeeded`. Silently skips other files in the folder.
- **If yes** branch: holds the rest of the actions below.

**Action 2 — Compose `SurveyConfigs`** — embed the 4 surveys' configuration
- Inputs: paste the JSON from `distribution-watcher-flow.json` (the array under `SurveyConfigs.inputs`). One object per survey with `title`, `link`, `releaseDate`, `reminderDate`, `researchEndDate`, and `taskInstructions`. Replace the form URLs and instruction text with your real values.

**Action 3 — List rows present in a table** (Excel Online (Business))
- Location / Document Library / File: point at `Distribution.xlsx`.
- Table: `Distribution`.

**Action 4 — Apply to each `For_each_distribution_row`** (loop over Excel rows):

- **Get items `Get_existing_rows_for_email`** (SharePoint) — count this person's existing rows
  - Filter Query: `AssigneeEmail eq '@{items('For_each_distribution_row')?['Email']}'`
  - Top Count: `10`.

- **Apply to each `For_each_survey`** (nested loop, `outputs('SurveyConfigs')`):
  - **Filter array `Already_assigned`**
    - From: `body('Get_existing_rows_for_email')?['value']`
    - Condition (advanced mode): `@equals(item()?['Title'], items('For_each_survey')?['title'])`
  - **Condition** — `length(body('Already_assigned'))` is equal to `0`
    - **If yes — Create item** (SharePoint):
      - Title: `@items('For_each_survey')?['title']`
      - AssigneeEmail: `@items('For_each_distribution_row')?['Email']`
      - ReleaseDate: `@items('For_each_survey')?['releaseDate']`
      - ReminderDate: `@items('For_each_survey')?['reminderDate']`
      - ResearchEndDate: `@items('For_each_survey')?['researchEndDate']`
      - SurveyLink: `@items('For_each_survey')?['link']`
      - TaskInstructions: `@items('For_each_survey')?['taskInstructions']`
      - Completed: `No`.

- **Condition `Is_new_participant`** — was this email's existing row count `0` *before* we created any?
  - Left: Expression `length(body('Get_existing_rows_for_email')?['value'])`
  - Operator: `is equal to`
  - Right: `0`
  - **If yes** branch:
    - **Get user profile (V2)** (Office 365 Users), renamed `Get_new_participant_profile`. User (UPN): `@items('For_each_distribution_row')?['Email']`
    - **Send an email (V2)** (Office 365 Outlook), renamed `Send_welcome_email`:
      - To: `@items('For_each_distribution_row')?['Email']`
      - Subject: `Welcome to the Linkt research study — Survey 1 ready for you`
      - Body (HTML, paste with `</>` Code View on):
        ```html
        <p>Hi @{coalesce(outputs('Get_new_participant_profile')?['body/givenName'], 'there')},</p>
        <p>Welcome to the Linkt research study! Over the next two weeks you'll be invited to complete <strong>4 short surveys</strong>, each tied to a small task in the Linkt app.</p>
        <p>Your first survey is ready now:</p>
        <p><strong>Survey 1</strong> — <a href="@{first(filter(outputs('SurveyConfigs'), equals(item()?['title'], 'Survey 1')))?['link']}">Open survey</a></p>
        <p><strong>Task instructions:</strong> @{first(filter(outputs('SurveyConfigs'), equals(item()?['title'], 'Survey 1')))?['taskInstructions']}</p>
        <p>The remaining 3 surveys will be released over the next two weeks. We'll send reminders if you haven't completed any by their reminder dates. All 4 must be completed by <strong>Sun 10 May 2026, 11:45 PM</strong>.</p>
        <p><b>Still need the app?</b></p>
        <ol>
          <li>Open TestFlight → find Linkt → tap Install. Work through your tasks when you're ready.</li>
        </ol>
        <p>Email <a href="mailto:[SUPPORT EMAIL]">[SUPPORT EMAIL]</a> if you need a hand — we're here during business hours.</p>
        <p>Thanks,<br>
        [name]<br>
        Linkt Research Team</p>
        <p>[TU email signature logo]</p>
        <p>[support email]</p>
        ```
        Replace the placeholders before going live.
  - **If no** branch: leave empty (existing participants don't get re-welcomed).

### Operating notes

- **Toggle the flow Off while you edit the spreadsheet pre-launch.** Turn it back On when ready to start admitting participants. Past saves don't replay; only saves while the flow is on are picked up.
- **Re-saves with no new emails** are safe: dedup catches everyone, no rows or emails are created.
- **Removing an email from the spreadsheet does not delete their assignment rows.** Delete those manually in SharePoint if you want to stop reminding someone.
- **Bulk paste risk.** A 5,000-row paste & save will assign all 5,000. Decide who has write access to `Distribution.xlsx`.

### Backup: manual seeding flow

If you'd rather seed by clicking Run, the original manual flow is still in the repo as `excel-to-assignments-flow.json` — same logic, but with a *Manually trigger a flow* trigger and per-run inputs. Walk-through is in the file's parameters section. Run it four times (once per survey) using the values:

| Run | surveyTitle | releaseDate | reminderDate | researchEndDate         |
| --- | ----------- | ----------- | ------------ | ----------------------- |
| 1   | Survey 1    | 2026-04-28  | 2026-04-30   | 2026-05-10T23:45:00     |
| 2   | Survey 2    | 2026-05-01  | 2026-05-03   | 2026-05-10T23:45:00     |
| 3   | Survey 3    | 2026-05-04  | 2026-05-06   | 2026-05-10T23:45:00     |
| 4   | Survey 4    | 2026-05-07  | 2026-05-10   | 2026-05-10T23:45:00     |

The manual flow doesn't write `TaskInstructions` and doesn't send the welcome email — use the auto-seed flow as your default; this one is just a fallback.

## 3. Build the reminder flow

Create a **Scheduled cloud flow** — runs daily, composes one consolidated email per assignee.

**Trigger — Recurrence**
- Frequency: `Day`; Interval: `1`; At these hours: `9`; At these minutes: `0`.
- Time zone: your local time zone.

**Action 1 — Compose `todayDate`**
- Inputs: `@formatDateTime(utcNow(), 'yyyy-MM-dd')`

**Action 2 — Get items** (SharePoint) — everybody's outstanding surveys that are still in play
- Filter Query:
  ```
  Completed eq 0 and ReleaseDate le '@{outputs('todayDate')}' and ResearchEndDate ge '@{utcNow()}'
  ```
- Top Count: `5000`.

**Action 3 — Initialize variable `outstandingHtml`** (at the top of the flow, before Get items) — holds the per-assignee HTML list
- Name: `outstandingHtml`
- Type: `String`
- Value: leave blank.

**Action 4 — Filter array `Filter_trigger_rows`** — today's "fire the reminder" rows
- From: `@outputs('Get_items')?['body/value']`
- Condition (advanced mode — click the ⇄ icon):
  ```
  @and(lessOrEquals(formatDateTime(item()?['ReminderDate'], 'yyyy-MM-dd'), outputs('todayDate')), empty(item()?['LastReminderSent']))
  ```

  **Why the `formatDateTime` wrapper:** SharePoint returns `ReminderDate` as a full ISO timestamp (`2026-04-30T00:00:00Z`), which sorts *after* `2026-04-30` in a string comparison — so a row due today would be excluded. `formatDateTime(..., 'yyyy-MM-dd')` strips the time so the comparison works. `empty()` (rather than `equals(..., null)`) handles SharePoint's mix of null / empty-string / missing-field semantics reliably.

**Action 5 — Select `Select_trigger_emails`** — pull out the email addresses
- From: `@body('Filter_trigger_rows')`
- Map (text mode — click the **T** icon to switch from key/value to single-value): `@toLower(item()?['AssigneeEmail'])`

**Action 6 — Compose `Unique_assignee_emails`** — dedupe
- Inputs: `@union(body('Select_trigger_emails'), body('Select_trigger_emails'))`

**Action 7 — Apply to each unique assignee** (loop over `@outputs('Unique_assignee_emails')`):

- **Filter array `Filter_this_assignees_outstanding`** — pick out *this person's* outstanding surveys:
  - From: `@outputs('Get_items')?['body/value']`
  - Condition (advanced mode):
    ```
    @equals(toLower(item()?['AssigneeEmail']), items('Apply_to_each'))
    ```

- **Set variable `Reset_outstandingHtml`** — clears the HTML accumulator for this assignee:
  - Name: `outstandingHtml`
  - Value (via the **Expression** tab, because the designer won't accept an empty literal): `concat('')`

- **Apply to each outstanding item** (nested loop over `@body('Filter_this_assignees_outstanding')`):
  - **Append to string variable**:
    - Name: `outstandingHtml`
    - Value:
      ```html
      <li><strong>@{items('Apply_to_each_3')?['Title']}</strong> &mdash; <a href="@{items('Apply_to_each_3')?['SurveyLink']}">Open survey</a><br><em>Task:</em> @{items('Apply_to_each_3')?['TaskInstructions']}</li>
      ```
      (Use whatever the nested loop's actual internal name is — check its title bar. Spaces become underscores.)

- **Compose `Research_end_pretty`** — format the deadline nicely:
  - Inputs: `@formatDateTime(first(body('Filter_this_assignees_outstanding'))?['ResearchEndDate'], 'dddd d MMMM yyyy, h:mm tt')`

- **Get user profile (V2)** (Office 365 Users) — rename it `Get_assignee_profile`. Resolves the recipient's first name for personalisation.
  - User (UPN): `@items('Apply_to_each')`

- **Send an email (V2)** (Office 365 Outlook):
  - To: `@items('Apply_to_each')`
  - Subject: `Reminder: outstanding survey(s) to complete`
  - Body (HTML — click the `</>` icon on the Body field to enable code view before pasting):
    ```html
    <p>Hi @{coalesce(outputs('Get_assignee_profile')?['body/givenName'], 'there')},</p>
    <p>You still have <strong>@{length(body('Filter_this_assignees_outstanding'))}</strong> outstanding survey(s) to complete before the research period closes on <strong>@{outputs('Research_end_pretty')}</strong>:</p>
    <ul>@{variables('outstandingHtml')}</ul>
    <p><b>Still need the app?</b></p>
    <ol>
      <li>Open TestFlight → find Linkt → tap Install. Work through your tasks when you're ready.</li>
    </ol>
    <p>Email <a href="mailto:[SUPPORT EMAIL]">[SUPPORT EMAIL]</a> if you need a hand — we're here during business hours.</p>
    <p>Thanks,<br>
    [name]<br>
    Linkt Research Team</p>
    <p>[TU email signature logo]</p>
    <p>[support email]</p>
    ```
    Replace `[SUPPORT EMAIL]`, `[name]`, `[support email]`, and `[TU email signature logo]` with your real values. For the logo, swap the placeholder for `<img src="https://your-logo-url.png" alt="TU logo" width="200">` or attach the image inline via Send email's attachment settings and reference it with `cid:logo.png`.

**Action 8 — Apply to each trigger row** (loop over `@body('Filter_trigger_rows')`) — stamps LastReminderSent so today's trigger rows don't re-fire tomorrow:

- **Update item** (SharePoint):
  - Id: `@items('Apply_to_each_2')?['ID']`
  - Title: `@items('Apply_to_each_2')?['Title']` (required pass-through)
  - LastReminderSent: `@utcNow()`

Save and test. The cleanest dry-run: manually insert a single row with `ReleaseDate` = yesterday, `ReminderDate` = today, `Completed = No`, `ResearchEndDate` in the future, then trigger the flow. You should receive one personalised email and see `LastReminderSent` populate on the row.

### Common pitfalls we hit

- **"Greyed out" Apply to each actions in run history** = 0 iterations. Work backwards: check the output count on each action (Get items → Filter_trigger_rows → Unique_assignee_emails) until you find the one that returned 0.
- **`concat('')` showing as literal text in the email** = you pasted it into the Value field as text instead of via the Expression tab. Re-enter through the Expression tab so it becomes a coloured chip.
- **Set variable "value is required" error** = the designer won't accept an empty literal; use `concat('')` via the Expression tab instead.
- **`"Item/Title" is no longer present in the operation schema"` error on Update item** = the SharePoint schema cache is stale. Re-select the list name in the action's dropdown to force a refresh, or delete and recreate the action.
- **`Get_assignee_profile` reference errors** = the action's display name doesn't match the expression name. Rename the action to exactly `Get_assignee_profile` (no spaces, no parens) via **⋯ → Rename** so the expression resolves.

## 4. Build the Forms ingestion flow

One of these per Microsoft Form (4 total in this campaign). The reminder flow only stops pinging a person once `Completed = Yes` — that's this flow's job.

### Prerequisite — capture the responder's email

Microsoft Forms fills the `Responder` field automatically when the form is restricted to your organisation (**Settings → Only people in my organization can respond** with **Record name** enabled). For a public form, add an explicit "Email address" question and use that answer in place of the built-in `responder` field.

### Find your IDs

- **Form ID:** the value after `?id=` in the form URL.
- **Question IDs:** when you add *Get response details* and open the dynamic content panel, each question appears as `outputs('Get_response_details')?['body/<questionId>']`. Grab the one for question 1.

### Build it (Automated cloud flow)

**Trigger — When a new response is submitted** — pick your form.

**Action 1 — Get response details** (Microsoft Forms)
- Form Id: same form.
- Response Id: `@triggerOutputs()?['body/resourceData/responseId']`.

**Action 2 — Get user profile (V2)** (Office 365 Users) — resolve the display name
- User (UPN): `@outputs('Get_response_details')?['body/responder']`

  Skip this step if your form already has an explicit "Name" question; use that answer directly instead.

**Action 3 — Get items** (SharePoint) — find the matching open assignment
- Filter Query:
  ```
  AssigneeEmail eq '@{outputs('Get_response_details')?['body/responder']}' and Title eq 'Survey 1' and Completed eq 0
  ```
  Replace `Survey 1` with the survey title for *this form*. (One ingestion flow per form, so hard-coding the title is fine.)
- Top Count: `1`.

**Action 4 — Condition** — `length(outputs('Get_items')?['body/value']) is greater than 0`

- **If yes — Update item** (SharePoint):
  - Id: `@first(outputs('Get_items')?['body/value'])?['ID']`
  - Title: pass through existing.
  - Completed: `Yes`
  - CompletedDate: `@utcNow()`
  - ResponderName: `@outputs('Get_user_profile_(V2)')?['body/displayName']`
  - Answer1: pick the answer to question 1 from the dynamic content panel.

- **If no — Create item** (SharePoint) — defensive fallback for "responded without ever being assigned":
  - Title: the survey title for this form.
  - AssigneeEmail: `@outputs('Get_response_details')?['body/responder']`
  - ReleaseDate / ReminderDate / ResearchEndDate: `@utcNow()` (placeholders — the row is being created already `Completed = Yes`, so it will never generate reminders).
  - Completed: `Yes`
  - CompletedDate: `@utcNow()`
  - ResponderName: `@outputs('Get_user_profile_(V2)')?['body/displayName']`
  - Answer1: same dynamic content as above.

### One form per flow

Microsoft Forms triggers are bound to a single form, so make 4 ingestion flows — one per survey — each with the corresponding `Title` value in its filter.

## Files in this repo

- `README.md` — this file.
- `distribution-watcher-flow.json` — the auto-seed flow (file-modified trigger on `Distribution.xlsx` → embedded SurveyConfigs → per-participant dedup + 4-row creation → welcome email for brand-new participants). Replace the form URLs, `taskInstructions`, and email-body placeholders (`[SUPPORT EMAIL]`, `[name]`, logo) before going live.
- `flow-definition.json` — the reminder flow (daily recurrence → outstanding query → filter today's trigger rows → group by assignee → build per-person HTML list via a string variable → fetch display name via Office 365 Users → send personalised consolidated email with task instructions → stamp trigger rows). Same email-body placeholders to replace.
- `forms-to-sharepoint-flow.json` — the Forms ingestion flow (Forms response → resolve display name → look up assignment → write `ResponderName`, `Answer1`, `Completed = Yes`). Duplicate this one per form and edit the `surveyTitle` parameter and the `REPLACE_WITH_QUESTION_1_ID` placeholder each time.
- `excel-to-assignments-flow.json` — fallback manual seeding flow (manual trigger → read distribution list → dedupe → create rows with per-survey dates). Run once per survey. Doesn't write `TaskInstructions` or send a welcome email — use the auto-seed flow as your default.
