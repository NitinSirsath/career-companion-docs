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

1. **Never in production.** `APP_MODE=dev` with `NODE_ENV=production` stops startup. In prod mode every sandbox path answers 404 before login, the sandbox pages are "not found", and the frontend asks nothing of the sandbox API. The one exception is a test account (section 7): it uses the sandbox in prod mode, and nobody else does.
2. **Sandbox only.** Dev mode never talks to Google: no Google client is built, no Google key is needed, and real Gmail cannot be connected. The dev database never holds real mail.
3. **Separate database, same standard.** Dev mode starts only on a database whose name ends in `_dev`; prod mode refuses such a database; automated tests keep their `test` databases. The three can never share one. All use the same schema and migrations.
4. **Real AI only.** There is no fake AI, no ready-made AI result and no preloaded sample data. Everything a tester sees comes from a mail they sent. With no AI key, mail is synced but not read, and the app says "AI key is not set up". Fake AI errors, to test the error screens, are a separate command and never part of `npm run devmode` (section 6).
5. **Send, then sync.** Sending a sandbox mail writes only to the sandbox. The mail reaches the app only when a sync pulls it, with the same code as real Gmail mail.
6. **Same shape as Gmail.** A sandbox message has the structure of a Gmail message, and the app reads it with the same code as real mail.

## 3. Start dev mode

Backend, one command:

```bash
npm run devmode
```

On first run it makes `.env.dev` from `.env.dev.example` and fills in the four secrets. Every run it creates the dev database when it is missing, rebuilds the Prisma client (so a pull with schema changes needs no extra step), brings the database to the latest structure and starts the backend. It never deletes data. In dev mode the backend log also shows an AI provider's own error text on a failed call (`providerMessage`, cleaned and cut short); prod mode never logs it. If PostgreSQL is not reachable with the default user, correct `DATABASE_URL` in `.env.dev`.

Frontend: `npm run dev`, as always.

To go back to prod mode, start the backend with `npm run dev`.

## 4. Test a mail's whole life

1. Open the app. On the login page, **Create test user** (any email, a password of 10 characters or more).
2. **AI provider** page: add your own provider key. Without it, mail is synced but not read.
3. **Gmail Sync** page: **Connect sandbox mailbox**. One click, no Google screen.
4. **Mail sandbox** → **Compose**: write a mail to yourself. "From" is any address you type, or one you used before. Send a few: some about jobs, some not. To try a reply to all, put other people in the optional **Cc** line (up to 10 addresses, with commas between them).
5. The mails show **Not synced yet**. Press **Sync now** (the same sync as on the Gmail Sync page).
6. After the sync and the AI reading, each mail shows what the app did: **Job mail** with company, role, the requested action and its application, or **Not a job mail**.
7. Reply **from the app**, as you would with real Gmail: on the Mails page open the mail, write in the reply box and choose **Send reply**. The reply lands in the sandbox, in the same conversation, and the Mails page shows Replied at once. No real mail is sent. On a mail with people in Cc the box says "Reply to all"; the sandbox shows the sent reply with its To and Cc. Nothing in the sandbox itself sends a mail as you.
8. To continue the conversation as the other person, open it in the sandbox and write in the box **Next mail from <address>**, then sync. The people in Cc stay on that next mail.
9. **Reset dev data** empties the sandbox mailbox and what the app made from it. It keeps your login, AI settings and the sandbox connection.

## 5. How it works

- Every Gmail call in the backend goes through `withGmail` (`services/gmailClient.ts`). In dev mode it hands the caller a sandbox client (`services/sandbox/gmail.ts`) that answers, from the table `sandbox_messages` and in Gmail's own response shapes, the calls the app makes: `getProfile`, `messages.list`, `messages.get`, `messages.send`, `history.list`, `threads.list`, `threads.get`, `labels.list` and `attachments.get`.
- So `services/gmailSync.ts`, the sync jobs, the Gmail tab and the mail details page are unchanged: manual sync, scheduled sync, the look-back window, the history checkpoint, "new mails only" and the noting of the user's replies all run on sandbox mail.
- A test fails when the app starts to make a Gmail call the sandbox does not answer.
- Of the user's own sandbox mail the app keeps what it keeps in prod: message ID, thread ID and sent time. The text stays in the sandbox.

## 6. Test the AI error screens (fake AI errors)

Owner decision, 2026-10-14: fake AI errors are their own command, next to the smoke script. Dev mode itself stays real AI only (rule 4).

Backend: `npm run fake-ai-errors -- <fault>`. It starts dev mode as `npm run devmode` does, but every AI call gets the chosen error instead of an answer. Nothing is sent to any AI provider, so no real key is needed. Each fake is what the AI library itself gives the app for that problem (its own error classes and the provider's own error body), so the app's real error handling runs. The app's code is not changed.

- **Never in prod.** It starts only when `.env.dev` says `APP_MODE=dev` on a `_dev` database, it refuses `NODE_ENV=production`, and the server checks the app mode again before it loads the fakes.
- **One fault per run:** `busy`, `no-answer` (the call stops after 5 seconds), `deadline` (504), `rate-minute`, `rate-day`, `key`, `billing`, `model-gone`, `unusable` (cut off at `MAX_TOKENS`), `no-connection` (`ENOTFOUND`), `line-dropped` (`ECONNRESET`) or `server-error` (500). Without a fault it lists them all. The codes each provider gets are in the backend README, "Testing AI error screens".

Steps:

1. `npm run fake-ai-errors -- busy`, and start the frontend as usual.
2. If the test user has no AI key yet, choose Google Gemini on the **AI provider** page and enter any 8 or more characters without spaces as the key, during a fault other than `key`, `billing` or `model-gone` (those refuse a new key). A user who already has a real key keeps it.
3. In the **Mail sandbox** send a mail, then **Sync now**. Compare the Gmail Sync and Mails pages, the row's (i) and the AI providers page with the approved mock. Search the server log for the mail's ID.
4. Stop with Ctrl+C and start again with the next fault.

Fake errors leave their states in the dev database: pauses, a rejected key, failed mails. Use a separate test user, or afterwards, in `npm run devmode`, save the key again and choose **Reset dev data**. The app's own pause lengths are not shortened. To try again sooner, "Read waiting mails now" sends one waiting mail as a test at once.

## 7. Test accounts on a shared deploy

Owner decision, 2026-10-20: one deploy (Render's free plan has one server and one database) serves real users and testers. Everyone signs in with Google and uses real Gmail. A few fixed **test accounts** sign in on their own page and use the mail sandbox instead of Gmail, with a real AI provider. This is prod mode, not dev mode: nothing else changes.

- **Setting:** `TEST_ACCOUNTS=dev1@test.com:<password>,dev2@test.com:<password>` in the backend's settings. At most 5 accounts. Each password is 16 to 200 characters. Empty: no test accounts, and the sandbox answers 404 before login as before.
- **Passwords:** long and random, one per account, kept only in the host's secret settings (on Render: Environment, as a secret). Never in code, GitHub or chat. Hand each tester theirs privately.
- **Created at start:** every server start creates a missing test account and sets its password to the setting. A changed password ends the account's sessions. An account already linked to Google is left alone. There is no other way to create one.
- **Sign-in:** `/test-login`, not linked from `/login`; share the address with the tester. It takes test accounts only (`POST /api/auth/test-login`, 403 `NOT_TEST_ACCOUNT` for any other email), with the same guessing limits as the normal sign-in. The normal sign-in and Google sign-in refuse a test account. A test account cannot change its password in the app.
- **In the app:** a "Test account" banner, the **Mail sandbox** menu entry and **Connect sandbox mailbox** on the Gmail Sync page. From there it works as in section 4, steps 2 to 9. Real users never see any of it.
- **How it works:** one check, "does this user use the sandbox?", is true in dev mode for everyone and in prod mode for test accounts (`services/testAccounts.ts`). `withGmail`, Gmail connect and disconnect, replies and the sandbox routes ask it instead of the app mode. `GET /api/auth/me` says `testAccount: true`. Each test account has its own sandbox mailbox.
- **Shared database:** test data sits in the same tables as real data and belongs to the test account's user, like any user's data. Leave the test accounts out when counting across all users.

## 8. Not built yet

Later work, tracked as `[Later]` issues of the dev mode epic: choosing a mail's arrival time, attachments, search, and an AI writer for test mail. A received sandbox mail always arrives in Gmail's "Primary" category, and a new mail always goes to the user's own inbox.
