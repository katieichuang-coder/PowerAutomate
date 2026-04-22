# Survey Reminder Flow

A Microsoft Power Automate flow that sends a reminder email to a user when they have not completed an assigned survey by its due date.

## What it does

1. Runs once per day (recurrence trigger).
2. Queries a SharePoint list of survey assignments.
3. For every row where `Completed = No` and `DueDate` is on or before today, sends the assignee a reminder email with a link to the survey.
4. Stamps `LastReminderSent` on the row so you can see when the last nudge went out.

## Prerequisites

- A Power Automate license (Free tier works for this flow).
- A SharePoint site you can create a list in.
- An Outlook / Office 365 mailbox the flow can send from.

## 1. Create the SharePoint list

Create a list called **`SurveyAssignments`** with these columns:

| Column name         | Type                    | Notes                                         |
| ------------------- | ----------------------- | --------------------------------------------- |
| `Title`             | Single line of text     | Name of the survey (built-in column).         |
| `AssigneeEmail`     | Single line of text     | Email address of the person to remind.        |
| `DueDate`           | Date and Time (Date only) | The due date of the survey.                 |
| `Completed`         | Yes/No                  | Default `No`. Flipped to `Yes` by the ingestion flow on submission. |
| `SurveyLink`        | Hyperlink               | URL to the survey (Forms, Qualtrics, etc.).   |
| `LastReminderSent`  | Date and Time           | Written by the reminder flow. Leave blank initially. |
| `CompletedDate`     | Date and Time           | Written by the ingestion flow on submission.  |
| `ResponderName`     | Single line of text     | Written by the ingestion flow on submission.  |
| `Answer1`           | Multiple lines of text  | Answer to the first Forms question. Rename to match the actual question if you like. |

## 2. Build the flow

You have two options:

### Option A — Import the provided definition

1. In Power Automate, go to **My flows → Import → Import Package (Legacy)** (or use the "Create from blank" approach below if import is disabled in your tenant).
2. Use `flow-definition.json` in this repo as a reference for every action and expression. Each action is named and commented inline.

### Option B — Build it by hand (recommended, 5 minutes)

Create a new **Scheduled cloud flow** and add the steps below. All expressions are exactly what you would paste into the Power Automate expression editor.

**Trigger — Recurrence**
- Interval: `1`
- Frequency: `Day`
- At these hours: `9`
- At these minutes: `0`
- Time zone: your local time zone.

**Action 1 — Initialize variable `todayDate`** (Compose)
- Inputs:
  ```
  formatDateTime(utcNow(), 'yyyy-MM-dd')
  ```

**Action 2 — Get items** (SharePoint → Get items)
- Site Address: your site.
- List Name: `SurveyAssignments`.
- Filter Query:
  ```
  Completed eq 0 and DueDate le '@{outputs('todayDate')}'
  ```
  (SharePoint stores Yes/No as `1`/`0`. Use `ne 1` if your column is a managed metadata or choice field.)
- Top Count: `5000` (or less, depending on list size).

**Action 3 — Apply to each** (loop over `value` from Get items)

Inside the loop:

**Action 3a — Send an email (V2)** (Office 365 Outlook)
- To: `@{items('Apply_to_each')?['AssigneeEmail']}`
- Subject:
  ```
  Reminder: please complete "@{items('Apply_to_each')?['Title']}"
  ```
- Body (HTML):
  ```html
  <p>Hi,</p>
  <p>Our records show that the survey
    <strong>@{items('Apply_to_each')?['Title']}</strong>
    was due on
    <strong>@{formatDateTime(items('Apply_to_each')?['DueDate'], 'MMMM d, yyyy')}</strong>
    and has not yet been submitted.
  </p>
  <p>
    Please take a moment to complete it:
    <a href="@{items('Apply_to_each')?['SurveyLink']}">Open the survey</a>
  </p>
  <p>Thanks!</p>
  ```
- Importance: `Normal` (bump to `High` if you want).

**Action 3b — Update item** (SharePoint → Update item)
- Site Address / List Name: same as Get items.
- Id: `@{items('Apply_to_each')?['ID']}`
- Title: `@{items('Apply_to_each')?['Title']}` (required field; pass through unchanged).
- LastReminderSent: `@{utcNow()}`

Save the flow. Run **Test → Manually** with a row whose `DueDate` is yesterday and `Completed = No` to verify the email arrives.

## Optional enhancements

- **Escalate to manager after N reminders.** Add an `Integer` column `ReminderCount`, increment it in Action 3b, and branch with a **Condition** action: if `ReminderCount >= 3`, also email the user's manager (Office 365 Users → *Get manager (V2)*).
- **Stop reminding after a grace period.** Tighten the filter to `DueDate ge '@{addDays(utcNow(), -14, 'yyyy-MM-dd')}'` so the flow gives up 14 days past due.
- **Send the first reminder *before* the due date.** Change the filter to `DueDate eq '@{formatDateTime(addDays(utcNow(), 3), 'yyyy-MM-dd')}'` to nudge three days ahead.
- **Swap data source.**
  - *Dataverse:* replace Get items / Update item with **List rows** and **Update a row**. The filter becomes `cr_completed eq false and cr_duedate le @{outputs('todayDate')}`.
  - *Excel (OneDrive/SharePoint):* use **List rows present in a table** + **Update a row**. Filter rows with a **Filter array** action instead of OData, because Excel connectors do not support `le` reliably on date columns.

## 3. Build the Forms ingestion flow

The reminder flow only works if rows in `SurveyAssignments` get marked `Completed = Yes` when a user actually submits the survey. This second flow listens to a Microsoft Forms form, looks up the matching assignment row, marks it complete, and copies the answers you care about into the list.

### Prerequisite — capture the responder's email

Microsoft Forms only fills the `Responder` field automatically when the form is restricted to your organization (**Settings → Only people in my organization can respond**, with **Record name** enabled). If your form is public, add an explicit "Email address" question and reference that question in the flow instead of the built-in `responder` field.

### Find your IDs (you'll paste these into the flow)

- **Form ID:** in the form URL, the value after `?id=`.
- **Question IDs:** when you add the **Get response details** action and use the dynamic content panel, each question shows up as `outputs('Get_response_details')?['body/<questionId>']`. Hover over each one to grab the ID, or just pick from the dynamic content menu in the UI.

### Build it (Automated cloud flow)

**Trigger — When a new response is submitted** (Microsoft Forms)
- Form Id: pick your form.

**Action 1 — Get response details** (Microsoft Forms)
- Form Id: same form.
- Response Id: `@triggerOutputs()?['body/resourceData/responseId']` (offered as dynamic content "Response Id").

**Action 2 — Get user profile (V2)** (Office 365 Users) — resolve the responder's display name
- User (UPN): `@outputs('Get_response_details')?['body/responder']`

  Microsoft Forms returns the responder as an email/UPN, not a friendly name. This step turns it into "Jane Doe". Skip this action if your form has an explicit "Name" question — just use that question's answer instead.

**Action 3 — Get items** (SharePoint) — find the matching open assignment
- Site Address / List Name: same as the reminder flow.
- Filter Query:
  ```
  AssigneeEmail eq '@{outputs('Get_response_details')?['body/responder']}' and Title eq 'Q2 Engagement Survey' and Completed eq 0
  ```
  Replace `Q2 Engagement Survey` with the survey title you used when seeding the list. If you have several surveys ingested by the same flow, parameterize this string or use a different flow per form.
- Top Count: `1`.

**Action 4 — Condition** — `length(outputs('Get_items')?['body/value']) is greater than 0`

**If Yes — Update item** (SharePoint)
- Id: `@first(outputs('Get_items')?['body/value'])?['ID']`
- Title: pass through the existing Title.
- Completed: `Yes`
- CompletedDate: `@utcNow()`
- ResponderName: `@outputs('Get_user_profile_(V2)')?['body/displayName']`
- Answer1: pick the answer to question 1 from the **Get response details** dynamic content panel.

**If No — Create item** (SharePoint) — defensive branch for "responded without ever being assigned"
- Title: the survey title.
- AssigneeEmail: `@outputs('Get_response_details')?['body/responder']`
- DueDate: `@utcNow()`
- Completed: `Yes`
- CompletedDate: `@utcNow()`
- ResponderName: `@outputs('Get_user_profile_(V2)')?['body/displayName']`
- Answer1: same dynamic content as above.

Save and submit a test response to verify the row is updated and the next morning's reminder run skips it.

### One form per flow

Microsoft Forms triggers are bound to a single form, so create one ingestion flow per form. The reminder flow stays singular because it just reads the SharePoint list.

## Files in this repo

- `README.md` — this file.
- `flow-definition.json` — the reminder flow (daily recurrence → SharePoint query → email + stamp).
- `forms-to-sharepoint-flow.json` — the Forms ingestion flow (Forms response → resolve responder display name → look up assignment → write `ResponderName`, `AssigneeEmail`, `Answer1`, and `Completed = Yes`). Replace the `REPLACE_WITH_QUESTION_1_ID` placeholder with the real Forms question ID before importing, or use the file purely as a reference while building the flow in the UI.
