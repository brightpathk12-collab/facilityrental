# Facilities Rental Portal — Pre-Launch QA (live app)

**Date:** September 24, 2026
**Code reviewed:** `Strive-Innovation-Labs/facilities-rental-portal` @ `74ae80c` (Sept 17, "Let renters send insurance in before the rental is approved"). This is the app behind rentfacilities.com.
**Supersedes:** `QA-REPORT.md` in this repo. That report reviewed the retired GitHub Pages prototype, not the live portal.

## How this was tested

| Check | Result |
|---|---|
| TypeScript (`tsc --noEmit`) | ✅ Clean |
| Production build (`next build`) | ✅ Clean, 20 routes |
| Test suite, against real Postgres | ✅ 268 of 268 passed, **7 runs in a row**, with no flakes |
| Browser walk-through of the running app | ✅ See the list below |
| Code review: security, plus workflow and money logic | Findings below. Each is marked **[browser]** (reproduced in the running app), **[code]** (traced through the source), or **[inferred]** |
| `npm audit` (production dependencies) | ❌ 1 critical, 4 high (see S1) |

**Browser walk-through (all passed):**
- The public catalog and a facility detail page.
- The 3-step request form, submitted for all three routes: school, School Operations and district.
- The renter's status link and the "Link not recognized" page for a bad link.
- 6 staff sign-ups, admin approval, and a login that was correctly refused while the account was pending.
- The full chain on one request: school designee approves → Kelly sets the $360 base amount → committee approves with a $200 law enforcement charge → documents verified → check recorded for **$560** → Final Approved → permit page and public `/verify` page.
- A stadium request denied by School Operations.
- A request for a school with no designee: Kelly correctly received an "Unassigned —" notice.
- Account lockout after 5 bad passwords.
- A District Admin cancelling a confirmed booking.

Every email was captured and checked for the right recipient and subject.

**Not tested:**
- **Live production:** this sandbox's network blocks rentfacilities.com.
- **Real file storage:** Cloudflare R2 is unreachable from here, so the upload showed "Something went wrong" as expected. I added the documents directly to the database to test the steps after upload.
- **Real mail delivery** through Resend.
- **Pending production data updates:** whether the two production reseeds flagged in HANDOFF.md (Sept 7 rates, Sept 14 gym capacities) have been run. Please confirm both before launch.

---

## 🔴 Fix before inviting staff

**S1. Upgrade Next.js (critical advisory).** The app runs `next 16.3.0`. `npm audit` reports GHSA-2xp9-vwfh-vxw4, remote code execution in the image optimizer, fixed in 16.3.3. The app only serves its own images, so it's probably hard to exploit here, but it's a one-line upgrade. The same audit includes a Windows-only advisory that doesn't apply on Render. Fix: `npm i next@^16.3.3`, then rerun the build and tests. [code]

**S2. Any signed-in staff account can open any request.** I signed in as the **Northside High** designee, pasted the address of a **Perry High** request, and the page showed everything: the renter's name, email and phone, the purpose, the amounts, the messages, and 10-minute download links to their insurance documents. The approve buttons were correctly hidden, so only viewing is exposed. The bootstrap admin, which has no role, can also view every request. `app/staff/requests/[id]/page.tsx` checks only that someone is signed in. Fix: add a view check so designees see only their own school. [browser]

**S3. Staff sign-up can be impersonated or duplicated.**
- `QA.Secretary@hcbe.net` registered as a **separate** pending account next to `qa.secretary@hcbe.net`. Emails aren't lowercased, and nothing checks that the person owns the mailbox.
- The approval screen shows only the email address and a date. Someone could register a lookalike address (for example `kelIy.douglas@hcbe.net`, with a capital I) and be approved by mistake.
- Also, **no one is emailed when a sign-up arrives or is approved.** After 7 sign-ups, zero emails were sent. Admins have to remember to check `/admin/approve`, and new staff aren't told they can sign in.
- Fix: lowercase emails, send a verification link before an account appears for approval, and notify admins of new sign-ups.

[browser]

## 🟠 Fix before opening the public form widely

**P1. The request form can be used to spam.**
- `/request` has no rate limit, CAPTCHA or length limits.
- Every submission sends a district-branded email from notify.rentfacilities.com to whatever address is typed. That email starts with "Thanks, {name}", and the name is whatever the submitter typed.
- Every submission also emails staff and adds to Kelly's queue.
- A script could flood Kelly and damage the sending domain's reputation.

Fix: add Cloudflare Turnstile (or similar) and per-IP limits, and cap field lengths. [code]

**P2. Invalid email addresses are accepted.** A request with the email `not-an-email@x` was accepted. The status link is sent **only** in that first email, so a typo means the renter never gets it. Fix: validate the email on the server and consider a "confirm email" field. [browser]

## 🟠 Workflow gaps that will bite at launch

These matter now because, per HANDOFF, launch starts with only three staff accounts (Kelly, Forrest and Dr. Brown) and **no school designees.**

**W1. Schools are never told about confirmed events in their building.** The "Event confirmed" email and the 3-days-before reminder go only to a school's designee. With no designee, the payment notice is silently skipped. The reminder script logs and skips too, with no fallback to Kelly. That differs from the stage notices, which correctly fall back to "Unassigned —". So until designees exist, a school can have an outside group arrive with no warning. Fix: fall back to the Facilities Secretary, or send to the principal's email from the schools list. [code]

**W2. Cancelling or rescheduling a confirmed booking tells only the renter.** When Dr. Brown cancelled a paid, Final Approved booking, the renter was emailed. The school that had received "Event confirmed" was **not**, and neither was Kelly, who holds the check and now owes a refund. Rescheduling works the same way. WORKFLOW.md promises "+ staff at the stage it was cancelled from", but the Final Approved stage has no staff recipients. [browser for cancel; code for reschedule]

**W3. Conflict detection misses shared spaces.** The "another request wants this facility" warning compares only the exact same facility. A stadium and its track (Herb St. John, McConnell-Talbert, and all 8 middle school stadium/track pairs) can both be approved for the same day with no warning. It also doesn't count the next day when an event runs past midnight. [code]

**W4. The private-event filter is unreliable.**
- **Blocked (false positives):** "America's 250th **birthday** community celebration" [browser], plus "Wake Forest alumni game" and "Awakening youth revival" [code].
- **Accepted (misses):** "**Baby shower** and gender reveal" [browser], plus quinceañera, bridal shower, anniversary party, celebration of life and "Smith reunion" [code].
- The block is a hard stop with no override, and only the purpose is checked, not the group name.
- Suggestion: turn it into a staff-review flag instead of a block. The attestation checkbox already covers the policy.

**W5. Insurance sent in early (the Sept 17 change) has rough edges.**
- If the renter uploads the wrong file early, they can't replace it: the upload form hides, and Kelly can't reject until Insurance Needed.
- The committee-approval email still says "upload your proof of insurance" when both documents are already on file.
- Nothing stops duplicate uploads, and the page then shows one of them at random.
- The form's attestation text still says insurance is required "after committee approval".

[code]

**W6. A $0 base amount can be finalized.** Kelly's amount field accepts 0, which the app also uses to mean "not yet finalized". Separately, before Kelly sets an amount, the renter's page shows **"Total due $0.00"** directly under an estimate of $360 [browser]. That could confuse renters. Fix: require an amount above $0, and hide "Total due" until it is set. [code + browser]

**W7. Price increases after payment can't be billed.** This gap is known. A paid booking rescheduled to more days has no way to collect the difference. It's worth a decision before launch. [code]

## 🟡 Lower priority

- **Facility page on phones:** the photo and the "About this space" card are 25px wider than the screen and get cut off on the right. The home, request, login and status pages fit. [browser]
- **Public `/verify` page** shows the facility and dates of requests a renter **withdrew** before approval, against its own "approved bookings only" rule. Reference codes are 6 random digits with no rate limit, so someone could step through them all. [code]
- **Abuse of sign-in and reset:**
  - Anyone can lock out a named staff member by entering 5 wrong passwords every 15 minutes (the lockout itself works [browser]).
  - Sign-up and lockout messages reveal which staff emails exist.
  - `/forgot-password` has no throttle.

  [code]
- **Security headers:** none are set (clickjacking protection, HSTS, referrer policy). [code]
- **Races:**
  - Two verifies at the same moment can strand a request at Insurance Needed with both documents verified.
  - Two refunds at the same moment can over-refund.
  - A designee's deny racing a stage change can land at the wrong stage.

  All need two people or two tabs acting at the same moment. [inferred]
- **Unbounded input:** attendance has no upper bound (a huge number produces "Something went wrong"), and text fields have no length limits. [code]
- **Formatting:** amounts show as "$1400.00" with no comma. [browser]
- **Contradictory copy:** step 2 of the form says costs are "not shown here" right after step 1 showed an estimate. [code]

## ✅ Solid, and verified

- **Every staff action re-checks the signed-in user and role on the server**, and designee decisions are scoped to their school. Renter actions use only the status link, never a request ID sent from the browser.
- **Status links and reset links** use 256-bit random tokens stored as hashes. A tampered link shows a friendly "Link not recognized" page. Reset links are single-use and expire in 30 minutes.
- **No script injection:** a `<script>` tag entered as the purpose displayed as plain text on the status page, the permit and the staff screens. HTML emails escape everything.
- **Passwords and access:** bcrypt, a 5-attempt lockout, `@hcbe.net`-only sign-up (a gmail address was refused), and no login while pending.
- **Pricing and money:** pricing, lead-time and weekend-gap rules match the fee schedule and procedures, including across daylight saving time. The amount owed equals the base plus committee charges.
- **Workflow:** each stage accepts only the right action. "Unassigned —" fallback notices reach Kelly.
- **No secrets are committed**, and `.env` is gitignored.

## Suggested order

1. **Today:** upgrade Next.js (S1). Confirm the two production reseeds ran.
2. **Before the staff sign-up email goes out:**
   - Scope the request detail page (S2).
   - Lowercase and verify sign-up emails, and add sign-up notices (S3).
   - Add fallbacks so schools and Kelly hear about confirmed and cancelled events (W1, W2).
3. **Before promoting the public form:** add the spam protection and email validation (P1, P2).
4. **Soon after launch:**
   - Shared-space conflicts (W3).
   - The private-event filter as a flag (W4).
   - Early-upload fixes (W5).
   - The $0 guard (W6).
   - The phone layout.
   - Security headers.
