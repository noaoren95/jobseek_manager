# Jobseek Manager — Daily Gmail Scan

You maintain a job-application tracker CSV stored in the user's Google Drive, by reading the user's Gmail.
Tools: Gmail MCP, Google Drive MCP, Bash (for base64/CSV handling). Timezone for all dates: Asia/Jerusalem.

## 1. Load the CSV from Google Drive
- Drive folder ID: `1KHO50rlIgmkTkwZ-RFJoxqQVs700QdPd` (My Drive/builder_projects/jobseek_manager).
- Find the file: `search_files` with query `title = 'applications.csv' and parentId = '1KHO50rlIgmkTkwZ-RFJoxqQVs700QdPd' and trashed = false`. If several exist, use the one with the latest `modifiedTime` (and trash the others at the end).
- Get its raw content with `download_file_content` (returns base64 or a saved file) and decode with Bash (`base64 -d`) if needed. Do NOT use `read_file_content` (it markdown-escapes underscores).
- Header: `applicationID,application_date,company_name,job_title,first_phone_call,interview_1,interview_2,interview_3,interview_4,rejection_date,offer_date,is_referral,referral_id`

## 2. Scan window
- If the CSV has **no data rows** (header only): BACKFILL — scan emails `after:2026/10/19` (inclusive).
- Otherwise: scan emails from the **last 2 days** — `newer_than:2d`. The overlap with the previous run is intentional (catches emails that arrived right after the previous scan, or a skipped run); the dedupe rules in step 4 prevent double-counting.

Many emails are in **Hebrew** and many come from **staffing agencies** (Manpower/Experis, Insite-HR, AdamTotal, etc.) or recruiters' personal addresses, so keyword searches alone are NOT enough. Run ALL of these searches, paginate until done (pageSize 50), and union the results:
1. **Catch-all (required):** `<window> -category:promotions -in:sent -from:me` with `view: THREAD_VIEW_SUMMARY`. Look at EVERY subject/sender/snippet in it, in any language, and decide whether it is about one of the user's applications.
2. `<window> (application OR applying OR applied OR candidacy OR candidate OR interview OR "phone call" OR "phone screen" OR "intro call" OR "next steps" OR "other candidates" OR "moving forward" OR unfortunately OR position OR role OR opportunity OR offer)`
3. `<window> (מועמדות OR מועמדותך OR משרה OR למשרת OR "קורות החיים" OR קו"ח OR ראיון OR לצערנו OR התאמה OR "הצעת עבודה" OR "שיחת היכרות" OR גיוס)`
4. `<window> from:linkedin.com (subject:"application was sent" OR subject:"your application to")`
5. `<window> (from:comeet-notifications.com OR from:greenhouse-mail.io OR from:hire.lever.co OR from:ashbyhq.com OR from:workablemail.com OR from:bamboohr.com OR from:myworkday.com OR from:smartrecruiters.com OR from:indeed.com OR from:manpowergroup.co.il OR from:insite-hr.co.il OR from:adamtotal.co.il)`

Search results don't include dates. Get a message's received date with `get_message` (MINIMAL format, `date` field), or decode it from the hex message ID: `python3 -c "import datetime;print(datetime.datetime.fromtimestamp((int('<id>',16)>>20)/1000, datetime.timezone(datetime.timedelta(hours=2))).date())"` (use +3 during Israeli summer time). Read the full body (`PLAIN_TEXT`) whenever the subject isn't enough to classify or to get the company/job title — this is usually the case for Hebrew and agency emails (e.g. subject "מועמדותך למשרת X" can be a rejection). LinkedIn "application sent" emails put the job title on the line right after "Your application was sent to <Company>".

## 3. Classify every email
**Everything not about the user's own job applications is IGNORED**: job alerts, "this job is a match", recommended jobs, newsletters, LinkedIn "application viewed", "people also applied", recruiter marketing, government employment-service (שירות התעסוקה / gov.il) notices, GitHub/other notifications.

| Category | Examples | Action |
|---|---|---|
| CONFIRMATION | "Thanks for applying", "We got it", "Application received", "תודה שהגשת מועמדות", "קיבלנו את קורות החיים שלך", ATS confirmations. **From LinkedIn, ONLY "Your application was sent to …" emails count as confirmations.** | New row |
| FIRST PHONE CALL | Invitation to the initial phone call / phone screen / intro call (שיחת היכרות / שיחה טלפונית), usually with a recruiter or HR, when `first_phone_call` is still empty and no interview is recorded yet | `first_phone_call` |
| INTERVIEW | Invitation to an interview (video, on-site, or any call after the first phone call) / home assignment / next stage | Next empty `interview_N` |
| REJECTION | "decided to move forward with other candidates", "not moving forward", "position filled", "the opportunity is no longer open", "לצערנו", "לא נמצאה התאמה", "החלטנו להמשיך עם מועמדים אחרים", "Update on your application" with a negative answer (including rejections relayed via LinkedIn "Your application to X at Y") | `rejection_date` |
| OFFER | Job offer / הצעת עבודה | `offer_date` |

Read "Update on your application"-style emails to determine whether they're a rejection, a phone call or interview invite, or nothing.
Staffing-agency emails: use the **hiring company** as `company_name` when the email names it (e.g. Manpower email about a job "ב שומרה חב' לביטוח" → `שומרה`); otherwise use the agency name.
Generic agency/recruiter intake emails that don't refer to a specific job application ("your details were received", "fill in this questionnaire") are not applications — list them under "ambiguous" in the report instead.

## 4. Update rows
- Dates are `YYYY-MM-DD`, the date the email was received.
- **CONFIRMATION** → append a row; `applicationID` = max existing ID + 1 (first is 1). Extract `company_name` (clean: "AppCard, Inc." → "AppCard", "Acme Ltd." → "Acme") and `job_title` from the email. If the title isn't stated anywhere, write `Unknown`.
  - Dedupe: if a row for the same company + same job title already exists (e.g., LinkedIn + ATS confirmation, two confirmation emails, or the same email seen again in the overlapping window), don't add another; keep the earliest `application_date`. Different job titles at the same company are separate rows.
- **FIRST PHONE CALL / INTERVIEW / REJECTION / OFFER** → match to an existing row by company (fuzzy: ignore case, "Inc/Ltd", Hebrew vs. English spelling, etc.); if the company has several rows, match on job title; if still ambiguous, use the most recent row that has no rejection. First phone call: write the date into `first_phone_call`. If the process starts directly with an interview, leave `first_phone_call` empty. Interview: write the date into the first empty `interview_1..4` — but don't add a new interview if that same date is already in `first_phone_call` or one of the interview columns (reschedules/time confirmations of the same round are not new interviews). If no matching row exists, create one with empty `application_date`.
- If the target field already holds a value (e.g. the email was already processed by the previous run), leave it alone.
- Process emails in chronological order (oldest first).
- **Never** modify `is_referral` or `referral_id` (the user fills them by hand), never delete rows, never change existing non-empty values except to set an earlier `application_date` when deduping.
- Write proper CSV (quote fields containing commas or quotes; UTF-8; Hebrew is fine). Keep rows sorted by `applicationID`.

## 5. Save back to Drive (only if something changed)
The Drive connector cannot overwrite file content, so replace the file:
1. `create_file` with `title: "applications.csv"`, `parentId: "1KHO50rlIgmkTkwZ-RFJoxqQVs700QdPd"`, `contentMimeType: "text/csv"`, `disableConversionToGoogleType: true`, `textContent: <full new CSV>`.
2. Verify the new file: `download_file_content` on its ID, decode, and check that the row count matches what you wrote.
3. Only after verification: `trash_file` the old file ID.
If nothing changed, don't touch Drive.

## 6. Report
End with a short summary: window scanned, number of rows in the CSV (count data rows with a CSV parser, not by eye), new applications (company — title), first phone calls, interviews, rejections, offers, and any ambiguous emails the user should check manually.
