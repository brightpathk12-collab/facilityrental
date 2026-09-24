# Facilities Rental Portal — QA Report

**Date:** September 24, 2026
**Build reviewed:** `index.html` "Prototype v7" plus `photos/` (commit `4bd90ef`)
**Method:** Full read of the code, automated checks of the facility, school, and photo data, and browser tests of the real user flows in headless Chromium at desktop (1366px) and phone (375px) widths. Every finding below was reproduced unless it is marked *(code review)*.

---

## Bottom line

The portal is a **working prototype** of the workflow, not a production system, and **it is not ready for real users yet**. The catalog, fee rules, photos, and five-step approval flow are well designed and work as a demo. But the portal has no server, no database, no login, and no email:

- Every request lives only in the visitor's browser tab. Refreshing the page erases it.
- Anyone can pick "Facilities administrator" from the role menu.
- The "Check request status" page shows another requester's request and email address.
- The 10-business-day rule is calculated from a frozen date of **July 13, 2026**.

**Recommendation:** Treat v7 as the specification. Before any users are added, move it onto a real platform with a database, district sign-in for staff, private status links for requesters, and Outlook email. The public **browse-only catalog** could go live sooner, after the content cleanup in Section C.

| Severity | Count | Meaning |
|---|---|---|
| 🔴 Blocker | 6 | Must be resolved before anyone uses the portal for real requests |
| 🟠 High | 8 | Wrong numbers, lost information, or broken workflow steps |
| 🟡 Medium | 7 | Data and content readiness |
| ⚪ Low | 5 | Polish, accessibility, mobile |

---

## A. 🔴 Blockers

**A1. Nothing is saved; nothing is sent.**
All requests are held in the browser's memory (`let requests = [...]`, index.html ~line 3872). In testing, a new request disappeared on page refresh (7 requests before, 6 after). The Outlook notifications and status-link emails appear only as "would send" messages; no email code exists. A real submission would reach no one.

**A2. No sign-in and no access control.**
"Staff sign in" opens the staff dashboard with no password. The "Preview signed-in role" menu lets any visitor become School approver, School Operations, Facilities Department, Committee, or Administrator, and approve, deny, cancel, or record payments. Also verified:
- A **school approver sees every school's requests**, not just their own. There is no link between a user and a school.
- If a user switches from Administrator to School approver while on *Setup data*, the admin-only screen stays open.
- All staff data (principal names and phones, fee setup, the request queue) ships in the public page source. Hiding a screen does not protect it.

**A3. The requester status page is not private.**
Clicking **Check request status** as a member of the public shows the currently selected request, *Houston County Debate Association, riley.chen@example.org*, with no link or code required. Production needs a unique, hard-to-guess link per request.

**A4. Unsafe handling of user-entered text (script injection).**
Organization name, contact, email, and purpose are inserted into the page as raw HTML in the dashboard, detail panel, status page, and monthly report. A test submission with the organization name `<img src=x onerror=...>` **ran code** when the staff dashboard loaded. Once requests are stored on a server, any member of the public could run code in a principal's or administrator's browser. Fix: escape every user-supplied value before display. An `escapeAttr()` helper already exists but isn't used there.

**A5. "Today" is frozen at July 13, 2026.**
`todayDate()` returns `2026-07-13` (line ~4045), so:
- The form suggests a first rental date of **July 28, 2026**, which has already passed.
- **August 3, 2026** (past) shows *"Eligible: this request is 15 business days away"* and submits successfully.
- Every recorded check payment is dated **2026-07-13** (line ~4286).

The 10-business-day rule is also checked only in the browser. A server must enforce it.

**A6. Demo content in the public form.**
- Every field comes pre-filled with the sample requester (*Houston County Robotics Club / Jordan Lee / jordan.lee@example.org / 1200 Main Street*). A rushed user could submit it unchanged.
- The **indemnification / hold-harmless agreement checkbox is pre-checked**. Legal consent should be an affirmative action. The checkbox also has no `name`, so the agreement is never recorded with the request.
- A **"Try blocked date"** demo button appears on the public form.
- Six sample requests (FR-2026-014 to 019) are loaded into every queue.

---

## B. 🟠 High: calculation and workflow bugs

**B1. The second tournament weekend is dropped when saved.** With a second weekend entered, the form showed **$2,460.00+**, but the saved request recorded **$820** and did not keep the second-weekend dates. The submit handler calls `calculateEstimate` without the extra days (line ~4567).

**B2. Committee "Approve with conditions" always adds exactly $292.** The amount is hard-coded (`payment.amount += 292`) and the conditions text is fixed. There is no way for the committee to enter the actual personnel, conditions, or amount.

**B3. A denial by the Facilities Department is shown as still pending.** After a Department denial, the request status is "Denied", but `secretaryDecision` stays "Pending". The requester's timeline still shows *"Facilities Department review — Status: Pending."*

**B4. No limit on the length of a date range.** A **120-day** range was accepted with a **$99,220** estimate, which conflicts with the "no standing or seasonal reservations" policy.

**B5. No double-booking or calendar check** *(code review)*. Two organizations can request the same facility at the same time, and nothing flags it. School Operations' stadium calendar is referenced only in text.

**B6. Request numbers are generated from a count** *(code review)*. New IDs come from `requests.length + 18` with the year fixed to 2026. With a database, deletions or two people submitting at once will cause duplicate numbers.

**B7. The insurance certificate file is thrown away** *(code review)*. The upload button marks insurance "received" without storing the file. There's no size check, and staff verify a file that was never saved.

**B8. The monthly report isn't monthly.** The heading is hard-coded to "July 2026 board report". The table and the "Reportable this month" tile include *every* request regardless of date.

---

## C. 🟡 Medium: data and content readiness

**C1. Facility data is mostly unverified.** 136 of 147 facilities are marked *"Assumed - verify"*. Only the 3 stadiums, 2 stadium tracks, and 6 district facilities are confirmed. The internal note *"Facility type is inferred from planning assumptions and must be confirmed by the school"* **appears to the public on 131 facility cards**. Every card, including confirmed ones, shows an **"Inventory to verify"** badge.

**C2. Required routing data is missing.**
- **0 of 147** facilities have a capacity ("Capacity to be confirmed").
- **All 38 schools** have their approver email set to *"To be collected"*, so school-level routing can't work.

**C3. Pricing inconsistencies to confirm with Facilities:**
- The high school tracks at Houston County, Northside, and Veterans are priced as *asphalt* ($50/hr). The published fee schedule has no high school asphalt rate, only "High School Track no lights $100/hr".
- 8 middle school tracks are labeled **"Surface unknown"** but priced as asphalt ($50/hr). If they are rubber, the rate is $100/hr.
- The fee schedule says "High School Soccer Field **no lights**", but the 5 high school soccer fields offer the $300 lights add-on.
- All 25 elementary gyms have no rate ("Set by committee"), and the district facilities are still "addendum pending".

**C4. The private-event keyword filter is unreliable.** It blocked *"America's 250th **birthday** community celebration"* (false positive) and allowed *"Marriage celebration for the Smith family"* (missed). Suggestion: flag these for staff review instead of hard-blocking on keywords. The attestation checkbox already covers the policy.

**C5. Typos and labels:**
- Photo captions: "**Cafeteriateria**" (Feagin Mill Middle) and "Freedom **FI**eld 2".
- Central Office facilities are labeled "**District school**", and stadium tracks are labeled "High school".
- The page briefly shows "163 facilities at 38 schools" before correcting to the actual **147 at 40**.

**C6. Photos are missing for 60 facilities:** all 50 elementary facilities, the 6 district facilities, and 4 high school facilities (Warner Robins High cafeteria; Northside, Veterans, and Warner Robins High soccer fields).

**C7. Staff names on the public site.** Public cards name the Facilities Secretary (*Kelly Douglas*) and the committee members, and principal names and phone numbers are in the page source. Confirm this is intended.

---

## D. ⚪ Low: polish, accessibility, mobile

- **D1. Money and time formatting:** amounts appear as "$770" in some places and "$770.00" in others, with no thousands separator ("$99220"). New requests show 24-hour times ("17:00 - 19:20"), while sample requests use "5:00 PM".
- **D2. Staff dashboard on phones:** the request table is 494px wide on a 375px screen, so users must scroll sideways. Public pages fit correctly.
- **D3. Keyboard access:** facility photos can only be opened with a mouse (not focusable). Opening the photo viewer or a message box doesn't move keyboard focus into it.
- **D4. Form default:** the facility picker opens on *District Stadiums → McConnell-Talbert Stadium*. A "Choose a school…" placeholder would prevent accidental stadium requests.
- **D5.** The "Next owner" text for school-route requests reads "*Perry High first approver*" and should use a real name or role once the approver data exists.

---

## What passed ✅

- All **216 photo files** load: no broken images, no missing files, no unused files, and every photo maps to a real facility.
- No JavaScript errors on load. Facility IDs are unique and no HTML element IDs are duplicated.
- Pricing is internally consistent: every 4-hour block equals 4 × the hourly rate.
- **The full approval path works end to end:** School Operations approval → Facilities Department → Committee approval with conditions → insurance → check payment → *Final Approved*, and the request appears correctly in the report.
- Form checks work for: end time before start time, last date before first date, second weekend order, and the 7-day gap between weekends.
- Public pages display correctly on a phone with no sideways scrolling.

---

## Suggested path to launch

1. **Decide the production platform.** It needs: a database, district Microsoft sign-in for staff with each approver tied to their school, private per-request status links for requesters, Outlook email, secure certificate storage, and an audit trail of who approved what.
2. **Fix the code issues** in A4–A6 and B1–B8 as the workflow moves to that platform. Enforce all rules, including the 10-day rule, eligibility, and date limits, on the server.
3. **Collect and verify the data:** approver emails for 38 schools, capacities, confirmation of facility types and track surfaces, and resolution of the pricing questions in C3.
4. **Optional quick win:** publish the catalog as browse-only, with the request form linked to the current paper or email process, after fixing C1, C5, and C6.
5. **Pilot** with one high school, one middle school, and School Operations before the district-wide rollout.
