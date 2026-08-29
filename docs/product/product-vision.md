# Product Vision — Career Companion

> **Status:** Finalized
> **Linear Issue:** [COM-2 — Define Product Vision](https://linear.app/welcome-nitin/issue/COM-2/define-product-vision)
> **Last Updated:** 2026-08-30

---

## Document Status

This document was developed incrementally through the Product Discovery workshop for COM-2.

All Product Vision sections are now finalized.

---

## ✅ Problem Statement

Job seekers lose visibility into their job search because application information becomes fragmented across Gmail.

As the number of applications increases, it becomes difficult to understand:

- Which companies have responded
- Which applications are still active
- Which emails require action
- Which assessments are pending
- Which interviews are scheduled
- Which applications have been rejected
- Overall progress of the job search

Managing this manually consumes significant time and mental effort, especially for professionals who are already working full-time.

Career Companion aims to eliminate this manual tracking by automatically understanding recruitment emails and presenting the complete state of the user's job search from a single dashboard.

---

## ✅ Primary Problems Identified

- No centralized view of the job search.
- Application status becomes difficult to track.
- Important emails become buried.
- Required actions are easy to miss.
- Information is scattered across Gmail.
- Reading hundreds of emails is time-consuming.
- Professionals do not have enough time to manually organize everything.

---

## ✅ Desired Outcome

The user should be able to open Career Companion and immediately understand:

- Where they have applied
- Current status of every application
- What requires action
- Upcoming interviews
- Pending assessments
- Recent recruiter communication
- Overall progress of the job search

...without manually reading Gmail.

---

## ✅ Product Direction

Career Companion is **NOT** simply a Gmail dashboard.

Career Companion is **NOT** an AI email reader.

Career Companion is a **job-search companion** that gives users complete visibility into their job search.

---

## ✅ Application Status Tracking

Application status tracking is a **core MVP capability**.

Career Companion should automatically detect and update application states:

| Status               | Included in MVP |
|----------------------|-----------------|
| Applied              | ✅              |
| Recruiter Contacted  | ✅              |
| Assessment           | ✅              |
| Interview            | ✅              |
| Offer                | ✅              |
| Rejected             | ✅              |
| Closed               | ✅              |

**Automatic rejection detection belongs in the MVP** — it is part of application tracking.

Advanced analytics and insights based on historical application data are **NOT part of the MVP** — those belong to a future phase.

---

## ✅ Target User

**MVP primary user**

Experienced professionals actively switching jobs.

**Future vision**

Expand Career Companion to support any job seeker.

---

## ✅ Out of Scope

Career Companion is NOT being built for:

- Recruiters
- HR Teams
- Hiring Managers
- Recruitment Agencies
- Enterprise Recruiting

---

## ✅ Assumptions (MVP)

- User uses Gmail.
- User is actively searching for jobs.
- User applies to multiple companies.
- Recruitment communication primarily happens through email.
- Most emails are in English.

---

## ✅ Vision Statement

Career Companion is an AI-powered job search companion that automatically understands recruitment emails, tracks every application, surfaces required actions, and gives users a clear, real-time view of their entire job search from one dashboard.

---

## ✅ Value Proposition

Career Companion turns scattered recruitment emails into an organized, actionable view of a job search, saving users time and helping them stay on top of applications, responses, interviews, assessments, follow-ups, and other recruitment requirements. Many job seekers, especially working professionals, do not have enough time to manually track every application and follow-up. Career Companion provides detailed monitoring of what has happened, what is pending, and what requires action — including assessments, document requests, resume or portfolio requirements, technology-stack requirements, and other job-specific requests.

**Core value:** The product should answer not only "What happened in my job search?" but also "What do I need to do next?"

---

## ✅ Product Goals

1. **Provide a clear view of the entire job search** — Users can quickly understand where their applications stand.
2. **Reduce manual tracking effort** — Automatically turn relevant Gmail communication into structured job-search information.
3. **Surface required actions** — Clearly identify assessments, document requests, interviews, follow-ups, and other things the user needs to act on.
4. **Keep application status up to date** — Detect meaningful recruitment events such as recruiter responses, interviews, offers, and rejections.
5. **Help users make better decisions about their job search** — Give users enough reliable information about their application activity and outcomes to understand how their search is progressing.

Advanced analytics are a future capability. The MVP should first establish reliable structured job-search data; analytics can be built on top of that data later.

---

## ✅ Non-Goals

Career Companion is not intended to become:

- A resume builder or resume optimization tool
- A job board or job discovery platform
- A recruiter, HR, or hiring-management platform
- A full email client
- A full task/project-management application
- An interview preparation or coaching platform
- An automatic job-application platform
- An advanced job-search analytics platform in the MVP
- A multi-platform communication hub (such as Outlook, LinkedIn, or other channels) in the MVP

**Core boundary:** Career Companion manages and understands the state of the user's job search rather than becoming an all-in-one job-search platform.

---

## ✅ Success Metrics

1. **Application visibility** — Users can see their tracked applications and current status from one place.
2. **Action visibility** — Relevant recruitment actions are surfaced clearly, so users can identify what needs attention without manually searching their inbox.
3. **Status accuracy** — Application statuses and major recruitment events are correctly identified from relevant emails.
4. **Time saved** — Users spend significantly less time manually reviewing and organizing recruitment emails.
5. **Job-search awareness** — Users can quickly understand their overall search progress, including applications, responses, interviews, assessments, offers, and rejections.
6. **Reliability** — Career Companion processes relevant job-related emails consistently without requiring users to manually maintain their application tracker.

No arbitrary numerical targets are defined at the Product Vision stage. Concrete technical and product targets will be established later when requirements and testing are defined.

---

## Document Versioning

| Version | Date       | Notes |
|---------|------------|-------|
| 0.1     | 2026-08-29 | Initial document. Finalized: Problem Statement, Primary Problems, Desired Outcome, Product Direction, Application Status Tracking, Target User, Out of Scope, Assumptions. |
| 0.2     | 2026-08-30 | Synchronized Vision Statement and Value Proposition from Linear COM-2 finalized decisions. |
| 1.0     | 2026-08-30 | Finalized Product Goals, Non-Goals, and Success Metrics. COM-2 completed. |
