---
name: weekly-job-search-summary
description: Weekly TWC job search activity summary from Gmail, Google Drive, and Google Sheet (via Chrome)
---

This is an automated run of a scheduled task. The user is not present to answer questions. For implementation details, execute autonomously without asking clarifying questions — make reasonable choices and note them in your output. For "write" actions (e.g. MCP tools that send, post, create, update, or delete), only take them if the task file asks for that specific action. When in doubt, producing a report of what you found is the correct output.

## Weekly Job Search Activity Summary — TWC Compliance Log

Your task is to generate a job search activity summary for `<YOUR_NAME>` (`<YOUR_EMAIL>`) for submission to the Texas Workforce Commission (TWC).

By default, the summary covers the 7 days ending today. The user may also override this by providing a custom date range when triggering the task manually (see Step 1).

---

### STEP 1 — Determine date range

**First, check the user's prompt for a custom date range.** Look for patterns like `4/15 to 4/21`, `MM/DD to MM/DD`, `M/D - M/D`, or `YYYY-MM-DD to YYYY-MM-DD`. If only month/day is given (no year), assume the **current year**. Both dates are inclusive (start date begins at 00:00, end date ends at 23:59).

**If a custom date range IS provided** (typical for manual runs), compute the variables using Python — replace the `date(YYYY, MM, DD)` values with the parsed start and end dates:
```bash
python3 -c "
from datetime import date, timedelta
start = date(YYYY, MM, DD)  # parsed start date from user's prompt
end = date(YYYY, MM, DD)    # parsed end date from user's prompt
end_plus_one = end + timedelta(days=1)
print('SINCE:', start.strftime('%Y/%m/%d'))
print('SINCE_ISO:', start.isoformat())
print('UNTIL:', end.strftime('%Y/%m/%d'))
print('UNTIL_ISO:', end.isoformat())
print('UNTIL_PLUS_ONE:', end_plus_one.strftime('%Y/%m/%d'))
print('UNTIL_PLUS_ONE_ISO:', end_plus_one.isoformat())
print('PERIOD:', start.strftime('%B %d, %Y') + ' to ' + end.strftime('%B %d, %Y'))
print('FILENAME:', 'job-search-logs-' + start.strftime('%m-%d-%y') + '-to-' + end.strftime('%m-%d-%y') + '.txt')
"
```

**If NO custom date range is provided** (default for automatic scheduled runs), use the original 7-days-ending-today logic:
```bash
python3 -c "
from datetime import date, timedelta
today = date.today()
since = today - timedelta(days=7)
today_plus_one = today + timedelta(days=1)
print('SINCE:', since.strftime('%Y/%m/%d'))
print('SINCE_ISO:', since.isoformat())
print('UNTIL:', today.strftime('%Y/%m/%d'))
print('UNTIL_ISO:', today.isoformat())
print('UNTIL_PLUS_ONE:', today_plus_one.strftime('%Y/%m/%d'))
print('UNTIL_PLUS_ONE_ISO:', today_plus_one.isoformat())
print('PERIOD:', since.strftime('%B %d, %Y') + ' to ' + today.strftime('%B %d, %Y'))
print('FILENAME:', 'job-search-logs-' + today.strftime('%m-%d-%y') + '.txt')
"
```

Use `SINCE`, `UNTIL`, `UNTIL_PLUS_ONE`, `PERIOD`, and `FILENAME` throughout the rest of the steps.

---

### STEP 2 — Read the Weekly Job Search Logs Google Sheet (via Chrome extension)

The Weekly Job Search Logs Google Sheet is the primary source for job search activities the user has manually logged. Use the Chrome extension tools to read it:

1. Use `mcp__Claude_in_Chrome__navigate` to open:
   `https://docs.google.com/spreadsheets/d/<YOUR_SHEET_ID>/edit`

2. Wait briefly for the page to load, then use `mcp__Claude_in_Chrome__get_page_text` to extract the content.

3. Parse the sheet content to find any rows/entries dated within the date range from Step 1 (between `SINCE` and `UNTIL`, inclusive). Extract: activity type, company name (if applicable), role/position (if applicable), date, method, and any notes.

**If Chrome is not available or the page fails to load:** note this in the output and proceed with remaining steps — do not halt the entire run.

---

### STEP 3 — Gather new documents from Google Drive

Search the **`<YOUR_DRIVE_FOLDER_NAME>`** Google Drive folder (ID: `<YOUR_DRIVE_FOLDER_ID>`) for any Google Docs added or modified within the date range from Step 1:

Use the google_drive_search tool with:
`'<YOUR_DRIVE_FOLDER_ID>' in parents and modifiedTime > 'SINCE_ISOT00:00:00' and modifiedTime < 'UNTIL_PLUS_ONE_ISOT00:00:00'`
(replace `SINCE_ISO` and `UNTIL_PLUS_ONE_ISO` with the values from Step 1)

For any new or modified documents found, fetch their full content using google_drive_fetch and determine which TWC category they support (e.g. a gap analysis or reemployment plan doc belongs in Category 4 — Reemployment Services).

**Note:** The Google Drive search tool only returns Google Docs and Folders — Google Sheets will not appear here. The Weekly Job Search Logs Sheet is handled separately in Step 2.

---

### STEP 4 — Gather data from Gmail

Use the Gmail search_threads tool to find job-search-related emails within the date range from Step 1. Use Gmail's `after:` and `before:` operators (Gmail's `before:` is exclusive, so `UNTIL_PLUS_ONE` captures activity through the END date inclusively). Substitute `SINCE` and `UNTIL_PLUS_ONE` from Step 1:

- Job applications submitted: `subject:(application OR applied OR "your application") after:SINCE before:UNTIL_PLUS_ONE`
- Interview invitations: `subject:(interview OR "phone screen" OR "schedule a call") after:SINCE before:UNTIL_PLUS_ONE`
- Networking / job fair confirmations: `subject:(networking OR "job fair" OR "career fair" OR "informational") after:SINCE before:UNTIL_PLUS_ONE`
- Recruiter outreach or responses: `subject:(recruiter OR opportunity OR "job opportunity") after:SINCE before:UNTIL_PLUS_ONE`

For each relevant thread, use get_thread to read the full content if needed to extract: company name, role/position, type of activity, and date.

Distinguish between job ALERT emails (LinkedIn/Indeed notifications about open roles — these confirm active platform use for Category 1) and actual application CONFIRMATIONS (emails confirming the user submitted an application — these belong in Category 2).

---

### STEP 5 — Look up company phone numbers

For **every confirmed job application** identified in Steps 2–4, use WebSearch to find a **public phone number** for that company (e.g., search "[Company Name] main phone number" or "[Company Name] HR contact"). Use the first reliable result (company's own website preferred). Note it as "Not found" if unavailable.

---

### STEP 6 — Synthesize into TWC categories

Organize all activities from the date range into the following 4 TWC-defined categories. Only include categories that have actual activity; mark empty ones as "No activities this week."

**Category 1: Online & Database Registration**
Activities such as: registering/searching on WorkInTexas.com, using Virtual Recruiter tool, registering with employment agencies or job-matching platforms (LinkedIn, Indeed, Handshake, etc.). Receiving LinkedIn job alert emails or Indeed match emails confirms active platform registration.

**Category 2: Direct Employer Contact**
Activities such as: submitting job applications (online, email, in-person), interviews, follow-ups on job leads.
- For each confirmed job application, include:
  - Company name
  - Position/role applied to
  - Date of application
  - Method (online, email, etc.)
  - Company's public phone number

**Category 3: Networking & Professional Development**
Activities such as: job fairs, networking events, job clubs, resume/interview workshops, skills assessments, informational interviews, courses or bootcamps enrolled in.

**Category 4: Reemployment Services**
Activities such as: updating the reemployment plan, creating supporting documents (gap analyses, skill roadmaps), accessing labor market information, using Workforce Solutions resources or online career tools. New documents added to the `<YOUR_DRIVE_FOLDER_NAME>` Google Drive folder typically belong here.

---

### STEP 7 — Write the summary file

Format the output as a clean, human-readable plain text file using the structure below. Then save it.

```
====================================================
WEEKLY JOB SEARCH ACTIVITY LOG
Period covered: [PERIOD from Step 1, e.g., April 15, 2026 to April 21, 2026]
Prepared for: Texas Workforce Commission (TWC) Compliance
Name: <YOUR_NAME> | Email: <YOUR_EMAIL>
====================================================

SUMMARY: [X] job search activities documented across [N] categories.

----------------------------------------------------
1. ONLINE & DATABASE REGISTRATION
----------------------------------------------------
[bullet list of activities, or "No activities this week."]

----------------------------------------------------
2. DIRECT EMPLOYER CONTACT
----------------------------------------------------
[For each confirmed application:]
  - Company: [Name]
    Position: [Role]
    Date Applied: [Date]
    Method: [e.g., Online via company website]
    Company Phone: [Phone number or "Not found"]

[Or "No activities this week."]

----------------------------------------------------
3. NETWORKING & PROFESSIONAL DEVELOPMENT
----------------------------------------------------
[bullet list or "No activities this week."]

----------------------------------------------------
4. REEMPLOYMENT SERVICES
----------------------------------------------------
[bullet list or "No activities this week."]

====================================================
Log generated on: [today's date and time]
Sources: Gmail (<YOUR_EMAIL>), Google Drive (<YOUR_DRIVE_FOLDER_NAME>), Weekly Job Search Logs Sheet
====================================================
```

Then use the Write tool to save the file at your output directory, using the `FILENAME` value computed in Step 1:
- Default 7-day runs: `job-search-logs-MM-DD-YY.txt` (where MM-DD-YY is today)
- Custom range runs: `job-search-logs-MM-DD-YY-to-MM-DD-YY.txt` (start date to end date)

---

### STEP 8 — Verify

After saving, confirm:
1. The file exists at the expected path
2. The file is non-empty and well-formatted
3. Report back: date range covered, total activities logged, categories covered, whether the Google Sheet was successfully read via Chrome, and any companies where a phone number could not be found.
