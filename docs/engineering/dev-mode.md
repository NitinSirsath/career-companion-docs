# Dev Mode and the Mail Sandbox

Dev mode is a way to **run the app** for manual testing: the same code as prod, switched by one setting, with the user's Gmail account replaced by a mail sandbox. It lets one person test a mail's whole life (arrive, sync, AI reading, application, reply) from one place, with no second Gmail account and no live data.

It has nothing to do with the working modes in [DEVELOPMENT-MODES.md](../../DEVELOPMENT-MODES.md), which say who may push to GitHub from which machine.

Introduced 2026-10-12. It replaces the earlier "manual test environment" (test inbox, `TEST_TOOLS_ENABLED`, `VITE_TEST_TOOLS`), which is gone.

## 1. The two modes

| | Prod mode | Dev mode |
|---|---|---|
| Setting | `APP_MODE=prod` (also when empty) | `APP_MODE=dev` |
| Backend env file | `.env` | `.env.dev` |
| Start the backend | `npm run dev` | `npm run devmode` |
| Sign-in | Google, or email and password | Email and password only |
| Gmail | The user's real Gmail account | The user's sandbox mailbox |
| Database | Any name that does not end in `_dev` | Name must end in `_dev` |
| AI | The user's own provider key | The same: real AI, the user's own key |

The frontend has no mode setting. At start it asks the backend (`GET /api/app-mode`) and follows it. In dev mode a "DEV MODE" banner is always visible.

## 2. Hard rules

1. **Never in production.** `APP_MODE=dev` with `NODE_ENV=production` stops startup. In prod mode every sandbox path answers 404 before login, the sandbox pages are "not found", and the frontend asks nothing of the sandbox API.
2. **Sandbox only.** Dev mode never talks to Google: no Google client is built, no Google key is needed, and real Gmail cannot be connected. The dev database never holds real mail.
3. **Separate database, same standard.** Dev mode starts only on a database whose name ends in `_dev`; prod mode refuses such a database; automated tests keep their `test` databases. The three can never share one. All use the same schema and migrations.
4. **Real AI only.** There is no fake AI, no ready-made AI result and no preloaded sample data. Everything a tester sees comes from a mail they sent. With no AI key, mail is synced but not read, and the app says "AI key is not set up".
5. **Send, then sync.** Sending a sandbox mail writes only to the sandbox. The mail reaches the app only when a sync pulls it, with the same code as real Gmail mail.
6. **Same shape as Gmail.** A sandbox message has the structure of a Gmail message, and the app reads it with the same code as real mail.

## 3. Start dev mode

Backend, one command:

```bash
npm run devmode
```

On first run it makes `.env.dev` from `.env.dev.example` and fills in the four secrets. Every run it creates the dev database when it is missing, brings it to the latest structure and starts the backend. It never deletes data. If PostgreSQL is not reachable with the default user, correct `DATABASE_URL` in `.env.dev`.

Frontend: `npm run dev`, as always.

To go back to prod mode, start the backend with `npm run dev`.

## 4. Test a mail's whole life

1. Open the app. On the login page, **Create test user** (any email, a password of 10 characters or more).
2. **AI provider** page: add your own provider key. Without it, mail is synced but not read.
3. **Gmail Sync** page: **Connect sandbox mailbox**. One click, no Google screen.
4. **Mail sandbox** → **Compose**: write a mail to yourself. "From" is any address you type, or one you used before. Send a few: some about jobs, some not. To try a reply to all, put other people in the optional **Cc** line (up to 10 addresses, with commas between them).
5. The mails show **Not synced yet**. Press **Sync now** (the same sync as on the Gmail Sync page).
6. After the sync and the AI reading, each mail shows what the app did: **Job mail** with company, role, the requested action and its application, or **Not a job mail**.
7. Reply in one of two ways:
   - **From the app:** on the Mails page open the mail, write in the reply box and choose **Send reply**. The reply lands in the sandbox, in the same conversation, and the Mails page shows Replied at once. No real mail is sent. On a mail with people in Cc the box says "Reply to all"; the sandbox shows the sent reply with its To and Cc.
   - **In the sandbox**, as if written in Gmail itself: open the conversation and **Reply**. A reply is always yours. Sync again: the reply shows **Recorded as your reply**, and the Mails page shows the conversation as Replied.
8. To continue the conversation as the other person, use **Write next mail from <address>** below the reply box, and sync. The people in Cc stay on that next mail.
9. **Reset dev data** empties the sandbox mailbox and what the app made from it. It keeps your login, AI settings and the sandbox connection.

## 5. How it works

- Every Gmail call in the backend goes through `withGmail` (`services/gmailClient.ts`). In dev mode it hands the caller a sandbox client (`services/sandbox/gmail.ts`) that answers, from the table `sandbox_messages` and in Gmail's own response shapes, the calls the app makes: `getProfile`, `messages.list`, `messages.get`, `messages.send`, `history.list`, `threads.list`, `threads.get`, `labels.list` and `attachments.get`.
- So `services/gmailSync.ts`, the sync jobs, the Gmail tab and the mail details page are unchanged: manual sync, scheduled sync, the look-back window, the history checkpoint, "new mails only" and the noting of the user's replies all run on sandbox mail.
- A test fails when the app starts to make a Gmail call the sandbox does not answer.
- Of the user's own sandbox mail the app keeps what it keeps in prod: message ID, thread ID and sent time. The text stays in the sandbox.

## 6. Not built yet

Later work, tracked as `[Later]` issues of the dev mode epic: choosing a mail's arrival time, attachments, search, and an AI writer for test mail. A received sandbox mail always arrives in Gmail's "Primary" category, and a new mail always goes to the user's own inbox.
