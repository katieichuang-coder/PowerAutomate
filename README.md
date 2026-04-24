# Survey Reminder Flows

Three Microsoft Power Automate flows that work together to run a multi-survey research campaign end-to-end:

1. **Seeding flow** — reads an Excel distribution list and creates one `SurveyAssignments` row per person per survey.
2. **Reminder flow** — runs daily, and on each survey's reminder date sends every assignee who still has outstanding surveys a single consolidated email listing all of them.
3. **Ingestion flow** — listens to Microsoft Forms responses and marks the matching assignment row `Completed = Yes` so reminders stop for that person.

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

## 2. Build the seeding flow (Excel → SharePoint)

You'll run this flow **four times** — once per survey — feeding it that survey's release date and reminder date.

### Prepare the Excel file

1. Save the workbook in a SharePoint document library (e.g. `Documents/SurveyOps/Distribution.xlsx`).
2. The data must be a real Excel **table**, not a plain range. Select the data → **Insert → Table** → check "My table has headers" → name the table `Distribution` under **Table Design**.
3. Required column: `Email`. Anything else is ignored.

### Build it (Instant cloud flow)

**Trigger — Manually trigger a flow** with five inputs:
- `surveyTitle` (Text) — e.g. `Survey 1`. Must match the Title used by the ingestion flow for this survey.
- `surveyLink` (Text) — the URL users click to take the survey.
- `releaseDate` (Date) — `yyyy-MM-dd`.
- `reminderDate` (Date) — `yyyy-MM-dd`.
- `researchEndDate` (Text) — ISO 8601, e.g. `2026-05-10T23:45:00`.

**Action 1 — Get items** (SharePoint) — load existing assignments to dedupe
- Site Address / List Name: your list.
- Filter Query: `Title eq '@{triggerBody()?['surveyTitle']}'`
- Select Query: `AssigneeEmail`
- Top Count: `5000`.

**Action 2 — List rows present in a table** (Excel Online (Business))
- Location / Document Library / File: point at `Distribution.xlsx`.
- Table: `Distribution`.

**Action 3 — Apply to each** (loop over `value` from Action 2):

- **Action 3a — Filter array** — is this email already assigned for this survey?
  - From: `@outputs('Get_items')?['body/value']`
  - Condition (advanced mode):
    ```
    @equals(toLower(item()?['AssigneeEmail']), toLower(items('Apply_to_each')?['Email']))
    ```

- **Action 3b — Condition** — `length(body('Filter_array'))` is equal to `0`
  - **If yes — Create item** (SharePoint):
    - Title: `@triggerBody()?['surveyTitle']`
    - AssigneeEmail: `@items('Apply_to_each')?['Email']`
    - ReleaseDate: `@triggerBody()?['releaseDate']`
    - ReminderDate: `@triggerBody()?['reminderDate']`
    - ResearchEndDate: `@triggerBody()?['researchEndDate']`
    - SurveyLink: `@triggerBody()?['surveyLink']`
    - Completed: `No`
    - (Leave the other columns blank — the other flows fill them.)
  - **If no** — skip; the person is already assigned.

### Running it for the current campaign

Run it four times with these inputs (dates as `yyyy-MM-dd`, end date as ISO 8601):

| Run | surveyTitle | releaseDate | reminderDate | researchEndDate         |
| --- | ----------- | ----------- | ------------ | ----------------------- |
| 1   | Survey 1    | 2026-04-28  | 2026-04-30   | 2026-05-10T23:45:00     |
| 2   | Survey 2    | 2026-05-01  | 2026-05-03   | 2026-05-10T23:45:00     |
| 3   | Survey 3    | 2026-05-04  | 2026-05-06   | 2026-05-10T23:45:00     |
| 4   | Survey 4    | 2026-05-07  | 2026-05-10   | 2026-05-10T23:45:00     |

Adjust the year for your actual campaign. The `researchEndDate` should be written in your tenant's local time — SharePoint stores it, and the reminder flow compares it with `utcNow()` so it will behave correctly as long as the stored value reflects an absolute point in time.

Re-running a survey's seeding run is safe: existing rows are skipped.

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
      <li><strong>@{items('Apply_to_each_3')?['Title']}</strong> &mdash; <a href="@{items('Apply_to_each_3')?['SurveyLink']}">Open survey</a></li>
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
- `flow-definition.json` — the reminder flow (daily recurrence → outstanding query → filter today's trigger rows → group by assignee → build per-person HTML list via a string variable → fetch display name via Office 365 Users → send personalised consolidated email → stamp trigger rows). `[SUPPORT EMAIL]`, `[name]`, and the logo block in the email body are placeholders — replace before going live.
- `forms-to-sharepoint-flow.json` — the Forms ingestion flow (Forms response → resolve display name → look up assignment → write `ResponderName`, `Answer1`, `Completed = Yes`). Duplicate this one per form and edit the `surveyTitle` parameter and the `REPLACE_WITH_QUESTION_1_ID` placeholder each time.
- `excel-to-assignments-flow.json` — the seeding flow (manual trigger → read distribution list → dedupe → create rows with per-survey dates). Replace the `REPLACE_WITH_*` Excel placeholders with your document library and file IDs. Run once per survey.
