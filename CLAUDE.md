# Jobseek Manager

Tracks job applications by scanning Gmail (via Gmail MCP) once a day.

- `applications.csv` — the tracker (one row per application).
- `SCAN_INSTRUCTIONS.md` — copy of the prompt used by the daily cloud routine "Jobseek Manager — daily Gmail scan" (runs 07:00 Asia/Jerusalem). If you change it, also update the routine at https://claude.ai/code/routines.
- The routine updates the CSV through the Google Drive connector by uploading a new applications.csv and trashing the old one, so the Drive file ID changes after each run that has updates.

`is_referral` and `referral_id` are filled manually by the user — never overwrite them.
