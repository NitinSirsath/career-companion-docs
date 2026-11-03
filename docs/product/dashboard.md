# Dashboard: the job search in numbers

Owner decisions of 27 Oct 2026, reworked by the owner on 3 Nov 2026. The Dashboard is the home page and the first menu item. It first shows what needs the user now, then how their search is going, then their mailbox. The Mails page answers "what do I do now, mail by mail?" and the Applications page "where is this one job?". Everything comes from data the app already reads. The user types nothing in.

The numbers are served by `GET /api/dashboard` (see [api-contracts.md](../architecture/api-contracts.md#dashboard-27-oct-2026-reworked-3-nov-2026)). The rules below are in one backend file, `services/dashboardMetrics.ts`.

## Decided by the owner (3 Nov 2026)

- The tips box at the top is removed, with its AI tip. The app no longer makes AI tips.
- **Needs you now** comes first and does not follow the period.
- **Your progress** and **Your mailbox** follow the period. The period picker sits in the "Your progress" header.
- Removed: the data line (its parts moved to Your mailbox), Right now, Where rejected jobs ended, Week by week (replaced by the trend lines and the running total), How fast companies answer (its median is now "Usual reply time"), How you keep up, and the Kept safe card (its number is in Your mailbox).
- **What works best** shows only to users who sent at least one application through the automation. This replaces the 27 Oct 2026 decision to show it to everyone.
- The **Calm look**, picked from three in a local mock (`DASHBOARD-V2-mock/` on the owner's computer): a greeting, soft grey panels without outlines, a blue dot on a tile that holds something, more space.
- On tablet widths the side menu shows icons only (see `DESIGN_SYSTEM.md`).

## What the page shows

Top to bottom:

1. **Greeting** (the page heading): "Good morning", "Good afternoon" or "Good evening" by the browser's clock, with the user's first name, then today's date and "your job search, from your mails since <date>". The menu still says Dashboard.
2. **Needs you now**, three tiles, whatever the period:
   - **New job mails**: conversations not opened yet, and how many of them got their newest mail today. Opens Mails with "New only".
   - **Needs action**: open requests, and how many answers by mail wait over a day. Opens Mails on "Needs action".
   - **Interviews coming up**: how many, and the next one with its day, time and company. Opens that conversation.
   - With nothing in them: "You have opened every job mail", "Nothing is waiting on you", "No interviews booked yet".
3. **Your progress**, with the period (last 7, 30 or 90 days, or since Gmail was connected; kept in the address):
   - **Five numbers**: applications sent, replies received, interviews reached, offers and rejections, each with the change against the same length of time just before, and a small trend line of the last 12 weeks (the weeks in the period in blue).
   - **Running total this period**: applications sent, replies received and interviews reached added up day by day, three lines on one axis.
   - **From applied to offer**: applications sent in the period by the furthest step they reached (columns, or bars when the card is narrow), with the response, interview and offer rates and the usual reply time.
   - **Gone quiet**: jobs to follow up or close. Three are shown; "Show all" opens the rest in place.
4. **Your mailbox**: mails scanned and job mails, all time with their count in the period, mails not linked to a job (leads to Mails), fake offers caught; then **Job mails per day** by kind (replies from companies, interviews and tests, rejections, other job mails) as stacked columns.
5. **What works best**, for users of the automation only: by site, by work mode, and what the automation did.

States: no Gmail yet (one "Connect Gmail" start and nothing else), the AI reading the first mails (its progress instead of empty numbers), a part that did not load (`SectionError`).

The layout follows the page's own width, so it fits with the side menu open or closed: phone under 600px, tablet from 600px, laptop from 900px.

## Rules for every number

- **Applied date** = `appliedAt`, else the first linked mail's `receivedAt`, else the application's `createdAt`.
- **An answer** = a linked mail of type Recruiter, Assessment, Interview, Offer or Rejection. Follow-up mails ("we got your application") do not count. Mails with a fake-offer warning never count.
- **Reached a step** = the furthest step in the status-change log before Rejected or Closed. When the user set an open status themselves, that status is the step: it is their correction of the AI.
- **The user's status wins** over the AI's, as everywhere in the app.
- **Small numbers**: a rate shows only when it rests on 5 or more applications; below that, plain counts ("1 of 3").
- **New job mails and Needs action** use the same queries as the Mails page, so the counts always match it.
- **Interviews coming up** come from the interview date and time the AI reads off each job mail (free text such as "Thursday, November 5, 2026" and "10:00 AM IST"). Each conversation counts its newest date; a range or a choice of days ("November 2-6") is not a booked interview; a job that is archived, rejected or closed does not count; the same interview found in two conversations counts once. Today counts all day, because the AI's time has no reliable time zone. The agenda's structured dates are not used: their extraction (`AGENDA_EXTRACTION_V3_ENABLED`) is off in production. The dates are the AI's, not confirmed by the user.
- **The series** covers the period day by day in the browser's time zone, week by week when it is longer than 120 days. Applications, replies (each job once, on its first answer in the period) and interviews add up to the five totals.
- **Kinds of job mail**: Recruiter and Offer are replies from companies; Interview and Assessment are interviews and tests; Rejection; everything else, and every mail with a fake-offer warning, is other.
- **Usual reply time** = the median days from applied to the first answer, for jobs whose first answer came in the period.
- **Mails scanned** come from `mail_scan_days`: every new inbox mail the sync takes in is counted on its UTC day, because mail that is not about a job is deleted after 35 days. The counts started on 3 Nov 2026 from what was still stored (all job mails, and other mail of the last 35 days), so totals from before then are partial. "In the period" sums whole UTC days.
- **Gone quiet** (owner, 27 Oct 2026): 21 days at Applied with no answer, or 14 days in a later step with no new mail. Quiet counts from the last thing that happened to the job: the applied date, a mail or a status change.
- **Weeks** start on Monday in the browser's time zone.
- **Data starts** at the first job mail, application or submission. The page says "since <date>", never "all time": the first sync reads at most 30 days back.
- **Sites and work modes** come only from automation submissions. Jobs not sent by the automation are one "Not from automation" row.

## Not in it

The Agenda page and confirming interview dates, industry benchmarks, a weekly goal the user types in, salary comparison, comparing users, and lists of mails or to-dos (Mails keeps those).
