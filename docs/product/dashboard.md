# Dashboard: the job search in numbers

Owner decisions of 27 Oct 2026. The Dashboard is the home page and the first menu item. The Mails page answers "what do I do now?" and the Applications page "where is this one job?". The Dashboard answers the questions in between: am I moving forward, is my effort working, where am I stuck, is it getting better, which sites work, am I keeping up, and can I trust these numbers. Everything comes from data the app already reads. The user types nothing in.

The numbers are served by `GET /api/dashboard` (see [api-contracts.md](../architecture/api-contracts.md#dashboard-27-oct-2026)). The rules below are in one backend file, `services/dashboardMetrics.ts`.

## What the page shows

Top to bottom:

1. **The period**: last 7, 30 or 90 days, or since Gmail was connected. It is kept in the address. Every number follows it, except "Right now", "Gone quiet" and "Week by week".
2. **The data line**: job mails read, mails not linked to a job, mails still being read, and fake offers caught. Each part leads to the list where it can be fixed.
3. **Tips**: up to three from fixed rules, and one from the AI (see "Tips" below).
4. **Five numbers**: applications sent, replies received, interviews reached, offers and rejections, each with the change against the same length of time just before.
5. **Right now**: active jobs by step. Each step opens its Applications tab.
6. **From applied to offer**: applications sent in the period by the furthest step they reached, with the response, interview and offer rates.
7. **Where rejected jobs ended**: before any reply, after recruiter contact, after an assessment or after an interview. It tells the user whether the CV or the interviews are the problem.
8. **Week by week**: applications sent, replies received and interviews reached in each of the last 12 weeks, whatever the period.
9. **Gone quiet**: jobs to follow up or close. Five are shown; "Show all" opens the rest in place.
10. **How fast companies answer**: the median days from applied to the first answer, the 8-in-10 value, and the three slowest companies.
11. **How you keep up**: the median time from a mail that asks for an answer by mail to the user's reply, the asks closed in the period by how they closed, and a link to the asks still open on Mails.
12. **Kept safe**: fake offers caught.
13. **What works best**: by site, by work mode, and what the automation did (sent, skipped, needs you, top skip reasons). They show for everyone; without the automation each card says the numbers come from it and links to Automation setup (owner).

States: no Gmail yet (one "Connect Gmail" start and nothing else), the AI reading the first mails (its progress instead of empty numbers), a part that did not load (`SectionError`).

## Rules for every number

- **Applied date** = `appliedAt`, else the first linked mail's `receivedAt`, else the application's `createdAt`.
- **An answer** = a linked mail of type Recruiter, Assessment, Interview, Offer or Rejection. Follow-up mails ("we got your application") do not count. Mails with a fake-offer warning never count.
- **Reached a step** = the furthest step in the status-change log before Rejected or Closed. When the user set an open status themselves, that status is the step: it is their correction of the AI.
- **Active** = not Rejected, not Closed, not archived. Archived jobs still count in the funnel and the trends.
- **The user's status wins** over the AI's, as everywhere in the app.
- **Small numbers**: a rate shows only when it rests on 5 or more applications; below that, plain counts ("1 of 3").
- **Gone quiet** (owner): 21 days at Applied with no answer, or 14 days in a later step with no new mail. Quiet counts from the last thing that happened to the job: the applied date, a mail or a status change.
- **Weeks** start on Monday in the browser's time zone.
- **Data starts** at the first job mail, application or submission. The page says "since <date>", never "all time": the first sync reads at most 30 days back.
- **Only job mail is counted.** There are no inbox totals: mail that is not about a job is deleted after 35 days.
- **Sites and work modes** come only from automation submissions. Jobs not sent by the automation are one "Not from automation" row.

## Tips

- **Rule tips**, at most three, only when true, with no AI call: jobs gone quiet; the user's reply time and mails waiting more than a day; a site that answers at least 15 points more often than another (both with 5 or more applications); it is early (the data started less than a week ago). The backend sends the facts, the page writes the words.
- **The AI tip** (owner): one tip written by the user's active AI, the free AI too, from the numbers of the last 30 days only: counts, rates and the automation's site names. No mail text, company, name or address is sent. A queued job makes it at most once a day, and calls the AI only when the numbers changed since the last tip; a failed try waits for the next day. The latest tip is kept in `dashboard_ai_tips`, so opening the page never calls the AI. The tip is hidden while the AI is off, paused or has a key problem; the rule tips stay. It is shown flat, with the AI icon and "AI tip by <model name>, from your numbers only".

## Not in it

An upcoming-interviews calendar (comes back with Agenda), industry benchmarks, a weekly goal the user types in, salary comparison, comparing users, and lists of mails or to-dos (Mails keeps those).
