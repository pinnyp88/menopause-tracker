# HerTrace — Architect's Handover

**To:** Successor model (Claude Sonnet)
**From:** Outgoing architect session
**Codebase:** One file. `index.html` (~7,400 lines). Vanilla JS + CSS, no build step, no framework, no external libraries except `supabase-js` v2 via CDN. Deployed by pushing `main` to GitHub → Vercel. Installable iPhone PWA.
**Everything below cites real line numbers as of commit `bb26038`. Line numbers drift — always re-grep the quoted code, never trust the number alone.**

Read this before touching anything: the app has a list of **protected functions** the owner has repeatedly instructed must not be modified (`saveCheckin`, `loadCheckinForDate`, `getCheckins`, `setCheckins`, `updateDashboard`, `updateCalendar`, `getSample`, `login`, `logout`, `loadUser`, `showPage`, `trendBadge`, `getTopTriggers`, `exportData`, `deleteAllData`, `saveHRT`, `loadHRTForm`, `saveProfile`, `loadProfile`) and **protected localStorage keys** that must never be renamed (`user`, `checkins`, `hrt`, `profile`, `notes`, plus `userName`, `hrtHistory`). When a fix requires touching a protected function, stop and get explicit owner sign-off first — the established pattern is: diagnose, propose, wait for "apply", show a diff, wait for "push".

---

## MODULE 1 — HRT DOSE TRACKING & ADHERENCE CALCULATION ⚠️ (KNOWN BROKEN — READ FIRST)

### 1. System mental model

There are **two separate HRT subsystems** that share only the dashboard card:

**(a) The regimen record** — *what the user is prescribed.* Stored as a single object in localStorage key `hrt` (fields: `estrogenName/Dose/Admin/Freq/Schedule/Start`, `progName/...`, `testUsed/testName/testDose/testStart`, `vagName/vagForm/vagDose/vagStart/vagNotes`, `notes`, plus legacy `doseChange*` fields kept only for backward compatibility). Edited on the HRT tab, saved by `saveHRT()`, restored by `loadHRTForm()`. Every save diffs old vs new via `hrtSummaryStr()`/`detectHRTChanges()` and appends change entries to localStorage key `hrtHistory` (array of `{hormone, date, previous, description, reason?, sideEffects?}`), deduped by `dedupHRTHistory()` on `hormone|date|description`. The user can backfill via `savePastEntry()` and delete single entries via `deleteHRTHistoryEntry()`. The earliest entry per hormone with no `previous` renders as "Started".

**(b) The adherence signal** — *whether the user took their dose each day.* One boolean per day: `hrtTaken` on each check-in record in the `checkins` array, set by the Yes/No pill on the Daily Check-In (hidden checkbox `#hrtTaken`, saved in `saveCheckin()` at ~line 4933).

The critical data-shape fact that the whole adherence calculation hinges on: **not every record in `checkins` is a real check-in.** Three writers create records:
| Writer | `loggedViaForm` | `hrtTaken` meaning |
|---|---|---|
| `saveCheckin()` (~4941) | `true` | user's actual answer |
| `quickLog()` stub (~4988) | `false` | **defaulted `false` — user was never asked** |
| Calendar day-tap stub in `updateCalendar` (~5616–5622) | `false` | **defaulted `false` — user was never asked** |
Legacy records from before the flag exist with `loggedViaForm === undefined` and are treated as form entries by the idiom `c.loggedViaForm !== false`.

So the correct adherence definition, used everywhere, is: *of the days the user actually answered the question (form entries), on what % was `hrtTaken` true.* Days with no record at all are deliberately ignored (it's "% of logged days", stated as such in the GP report scale column).

### 2. Strategic decisions, security & compliance

- Adherence is **presented to a doctor** on the GP report ("HRT Adherence … % of logged days HRT taken"). Any defect here is a clinical-communication defect, not a cosmetic one. Treat this module's correctness like you'd treat a dosage display.
- Health data flow: `checkins`/`hrt`/`hrtHistory` live in localStorage (primary) and are mirrored as one JSON blob per user to the Supabase `user_data` table via the storage-layer sync (see Data Storage module, next batch). There is **no field-level encryption** — protection at rest relies on device security + Supabase RLS (`auth.uid() = user_id`).
- **APP compliance flag (carried in full in the Storage module):** the on-screen disclaimer still says "All data is securely isolated on your local device" — this became **untrue** when cloud sync shipped. Under APP 1/APP 5 (open and transparent management, notification of collection) this is a live risk. Also `deleteAllData()` (~5793) calls `localStorage.clear()` but **never deletes the user's Supabase `user_data` row** — "Delete ALL your data? This cannot be undone" is misleading, an APP 11/12-adjacent risk. Do not "fix" silently; surface to the owner.

### 3. Tactical handoff — the bug, precisely

**What it should do:** if there are zero form-logged days in the window, adherence is *unknown* → display `—` / `N/A` and never trigger adherence-based messaging.

**What it actually does:** every calculation site correctly filters the numerator to form entries, but each site **fails differently when `formEntries.length === 0`**, and one mixes populations. Empirically verified (seed a week containing only quick-log stubs): dashboard shows **0%**, insight shows **"HRT adherence dipped this week."**, GP report prints **0%** for the doctor, Trends shows **"0% — 0 of 0 logged days"** with an empty bar.

The five sites (grep the code, don't trust line numbers):

| # | Site | Code | Defect when no form entries |
|---|---|---|---|
| 1 | Dashboard card, in `updateDashboard` (~5426–5429) | `var hdTotal = formEntries.length \|\| s.length;` | **Worst one — mixed populations.** Numerator counts `hrtTaken` among *form* entries (0), denominator falls back to *all* entries incl. stubs → false hard 0%. Should render `—`. |
| 2 | Dashboard insight, in `updateDashboard` (~5521–5526) | `var haPct = haForm.length > 0 ? … : 0;` then `else if (haPct < 70 && s.length >= 3) insight = 'HRT adherence dipped this week.'` | Defaulting the unknown to `0` makes the `< 70` branch fire → false alarm shown on the Home screen. Guard must be `haForm.length > 0 && haPct < 70 …`. |
| 3 | Printable GP report, in `exportData` (~5684–5685) | `Math.max(1, s7.filter(form).length)` | The `Math.max(1, …)` hack turns 0/0 into 0/1 → prints "0%" to the GP. Should print `N/A` (the `s7.length>0 ? … : 'N/A'` outer guard only covers a fully empty week). Same on the 30-day line. |
| 4 | Trends tab, in `renderTrends` (~6429–6433) | `var hrtPct = totalLogged > 0 ? … : 0;` | Renders "0% — 0 of 0 logged days" + empty bar. Should say "No HRT answers logged in this period." |
| 5 | Weekly Review, in `renderReflection` (~6862–6865 quote, ~7009–7012 GP list) | `qPct/hrtPct = … : 0;` | Quote path is safe (checks `qForm.length > 0` before praising 100%), but the GP-items path fires `'HRT was missed on some days — 0% taken.'` on a stub-only week. Same guard needed. |

**What already works (don't re-break):** numerator filtering is correct everywhere; quick-log and calendar stubs both carry `loggedViaForm:false`; the in-app Report page (`renderReport`, ~6701–6703) is the **one site done right** — `a7 = hrtLogged7 > 0 ? … : null` and downstream renders `—` for null. Use it as the reference implementation.

**Edge cases to watch while fixing:**
- Legacy entries (`loggedViaForm === undefined`) must keep counting as form entries — preserve the `!== false` idiom exactly; `=== true` would silently drop old users' history.
- **Timezone (latent, separate):** windowing uses `now - new Date(c.date) <= N*86400000`; date-only strings parse as **UTC midnight**, while records are written with **local** date strings (`localDate()`, streak logic uses local components). In Australia (UTC+10/+11) the 7/30-day windows are offset by 10–11 h — a day can drop out of "last 7 days" early. Do not fix casually: `getSample` is protected and its memoised `State._sample`/`_sampleValid` cache invalidation runs through `setCheckins`. Flag, propose, get sign-off.
- Adherence intentionally ignores never-logged days. If the owner ever asks for "of the last 7 calendar days", that's a product change, not a bug fix.

### 4. Concrete execution checklist

All five fixes are in **protected functions** (`updateDashboard`, `exportData`) or render functions — get explicit owner approval for the exact lines first, then:

1. **Site 1** (`updateDashboard`): delete the `|| s.length` fallback → `var hdTotal = formEntries.length;` (the existing ternary then renders `—`). One token removal; verify the card shows `—` on a stub-only week and the correct % on a mixed week.
2. **Site 2** (`updateDashboard`): change the branch to `else if (haForm.length > 0 && haPct < 70 && s.length >= 3) …`.
3. **Site 3** (`exportData`): compute `var f7 = s7.filter(form).length;` then `a7 = f7 > 0 ? Math.round(taken/f7*100)+'%' : 'N/A';` — remove `Math.max(1, …)`. Mirror for `a30`. Both the 7-day and 30-day table rows use these variables directly, nothing else to touch.
4. **Site 4** (`renderTrends`): when `totalLogged === 0`, emit a "No HRT answers logged in this period." line instead of the bar+percent block.
5. **Site 5** (`renderReflection`): wrap the `hrtPct < 80` GP-item push in `hrtFormWeek.length > 0 && …`.
6. **Verify in the preview** (established pattern: drive `preview_eval`, seed `checkins` directly, remember to null `State.checkins/_sample/_sampleValid` after writing localStorage): (a) stub-only week → `—`/`N/A`/no-alarm everywhere; (b) 3 form days 2 taken → 67% everywhere; (c) mixed stubs+form → stubs excluded from both numerator and denominator; (d) GP report prints `N/A` not `0%`.
7. **Pitfalls:** don't touch `hrtSummaryStr`/`detectHRTChanges` (regimen subsystem — unrelated); don't rename `loggedViaForm` or backfill it onto old records (cloud-synced peers may hold unstamped copies — the `scaleV:2` migration shows the accepted per-record-stamp pattern if a migration is ever wanted); test after `updateDashboard` runs, since `showPage('dashboard')` triggers it and overwrites manual DOM pokes.

---

## MODULE 2 — USER AUTHENTICATION

### 1. System mental model

Auth is **Supabase email OTP layered over a legacy local-name gate**, and the layering — not the OTP itself — is where all the historical bugs lived.

- **Legacy gate (still the real switch):** `loadUser()` (~4700, protected) shows the app iff `localStorage.userName` exists; `login()`/`logout()` (protected) manage it. The greeting treats the sentinel `'there'` as "no name chosen" and renders a generic greeting.
- **Supabase OTP flow:** login card → email input → `sendMagicLink()` calls `supabaseClient.auth.signInWithOtp({email, options:{shouldCreateUser:true}})` → user enters 6-digit code → `verifyOtpCode()` calls `auth.verifyOtp({email, token, type:'email'})`. Client initialised at the bottom of the file with the project URL + publishable key (`https://uenqxahaavdmcirfymay.supabase.co`); Supabase persists its session in its own localStorage keys.
- **The bridge — `supabaseSignIn(session)` (~7200, NOT protected, your main workbench):** writes `user = {name: emailPrefix, email}` (protected key, exact shape relied on by export/import and the GP report's contact-email line), sets `userName = 'there'` only if unset (**never the email prefix** — that caused the "Morning, pinnyp88." bug, fixed with a one-time migration that maps prefix→profile first name→sentinel), then calls `loadUser()` **only if the login page is still active**. That guard is load-bearing: `getSession()`/`onAuthStateChange('SIGNED_IN')` fire asynchronously and re-fire on token refresh; unguarded, they re-activated `dashboardPage` on top of whatever tab the user was on (the page-overlap bug). `syncSettingsEmail(email)` then restores the real email into Settings (because `loadUser()` writes `userName` into `#settingsEmail`).
- **Resilience layer:** a **separate `<script>` tag at the end of `<body>`** holds (a) the greeting-name migration and (b) the returning-user guard — if `userName` exists but the login page is still showing (i.e. an exception earlier killed startup), it calls `loadUser()` anyway. Being a separate script tag is the point: an error in the main script cannot prevent it. Keep it last; add nothing above it that can throw.
- Auth is also the trigger for cloud sync: the same two Supabase events call `startCloudSync(session)` which sets `_cloudUserId` and pulls/merges the `user_data` row (Storage module, next batch).

### 2. Strategic decisions, security & compliance

- **OTP-only, no passwords, no guest.** The "Continue without account" button was added once and deliberately removed — do not reintroduce it. Users who entered as `'Guest'` before removal still have `userName` set and bypass login until they log out; accepted by the owner.
- **`loadUser()` gates on `userName`, not on a Supabase session.** Consequence: a user who clears the Supabase session but keeps `userName` stays "logged in" locally with sync inert. Local-first by design — do not "tighten" this to require a session without owner sign-off; it would lock existing users out of their local data.
- **⚠️ APP compliance flags for this module:**
  1. **`logout()` does not call `supabaseClient.auth.signOut()`.** The message says "Your data will remain on this device", but the *auth session* also remains — reopening the app auto-signs back in via `getSession()`, re-populating `user` and re-arming cloud sync. On a shared device this is an APP 11 (security of personal information) risk. `logout()` is protected: the fix pattern is a wrapper at the caller (the More-sheet Logout button) that also signs out of Supabase — propose it, don't just do it.
  2. **`deleteAllData()` clears localStorage but not the Supabase session or the cloud `user_data` row** — after "delete everything", a reopen can resurrect the account and re-pull the cloud copy of the health data. This actively contradicts the deletion promise; highest-priority compliance item in the app. (Full treatment in the Storage module.)
  3. OTP emails come from Supabase's built-in sender (rate-limited); redirect URLs must be whitelisted in the Supabase dashboard (`Authentication → URL Configuration`) — a deploy-domain change silently breaks login.

### 3. Tactical handoff

- **What failed before (do not regress):** (1) unguarded `loadUser()` in auth callbacks → two `.page.active` at once (now impossible to *overlap* since pages are static blocks in the flex shell, but the guard also prevents yanking the user back to Home mid-task); (2) `userName` seeded from the email prefix → email leaked into the greeting; (3) users stranded on the OTP screen when a mid-file script error stopped startup → hence the end-of-body guard script.
- **Known edge cases:** `SIGNED_IN` fires on token refresh, not just fresh sign-ins — everything inside `supabaseSignIn` must stay idempotent and must not steal UI focus. `verifyOtpCode` reads `otpEmail` (module-level var set by `sendMagicLink`) — a page reload between "send" and "verify" loses it; user must resend. Error copy is fixed by owner spec: "That code didn't work — check your email and try again."
- **Testing recipe (established):** you cannot complete a real OTP in the preview. Simulate with `supabaseSignIn({user:{email:'x@example.com'}})` after stubbing `window.confirm`/`alert`; assert on `localStorage.user`, `userName`, `.page.active` count, and `#settingsEmail`. Supabase rejects `example.com` addresses on real sends — useful for testing the error path.

### 4. Concrete execution checklist

1. **(Owner decision) Sign out of Supabase on logout:** change the Logout button's `onclick` (More sheet, ~3720s) to a small new wrapper: `function fullLogout(){ if(supabaseClient) supabaseClient.auth.signOut().catch(function(){}); logout(); }` — leaves protected `logout()` untouched. Also set `_cloudUserId = null` (the `SIGNED_OUT` listener in the sync block already does this — verify it fires).
2. **(Owner decision, compliance-critical) Extend deletion to the cloud:** in `deleteAllData()`'s caller path, delete the `user_data` row (`supabaseClient.from('user_data').delete().eq('user_id', _cloudUserId)`) *before* `localStorage.clear()`, then `auth.signOut()`. `deleteAllData` is protected — same wrapper pattern.
3. **Never** write anything but `'there'` or a user-chosen first name into `userName`; never remove the login-page guard inside `supabaseSignIn`; never move the end-of-body guard script.
4. When testing any auth change, run the four-scenario suite from this session: fresh sign-in from login page enters app; auth event after navigating away does **not** change the active page; settings email correct; GP report shows the login email when profile has none.

---

## MODULE 3 — SYMPTOM LOGGING (Daily Check-In + Notes)

### 1. System mental model

One record per calendar day in the `checkins` array, keyed by `date` (local `YYYY-MM-DD` string). The Daily Check-In page (`#checkinPage`) is a single form whose fields map 1:1 onto record fields; `saveCheckin()` (protected, ~4920) reads the DOM and upserts by date; `loadCheckinForDate()` (protected, ~4790) restores a record into the form or — critically — **resets every field to its neutral default when no record exists** (this was a real bug once: fields not in the reset list leaked values across dates; if you ever add a field, you MUST add it to save, restore, *and* the blank-date reset, then call `syncCheckinVisuals()` still works).

Field inventory and their widgets:
- **Custom sliders** (`initSlider(fieldId, min, max, default, isPeach, reversed)` ~4580): `brainFog`, `patience`, `workImpact`, `relImpact`, `energyImpact`, `mentalClarity` (1–10, default 5), `sleepQuality` (1–5, default 3). Hidden `<input>` holds the value; track/fill/thumb divs render it; `sliderSetters.<field>(v)` is the programmatic setter. **Unified clinical severity scale (`scaleV:2`): 1 = mild/good LEFT, high = severe/bad RIGHT for every slider, badge = raw stored value.** The `reversed` option exists but is currently unused. The touch handler deliberately does NOT commit on touchstart — only on deliberate horizontal drag or stationary tap (8px intent threshold) so scroll-brushes can't change values. Do not "simplify" that handler.
- **Pill selectors** backed by hidden inputs: `hotFlush` (0–3), `nightSweat` (0–3), `urinary` (none/mild-urgency/frequent/urgency-incontinence/nocturia/multiple), `bleeding` (none/spotting/light/medium/heavy), `hrtTaken` (checkbox + Yes/No pills via `pickHrt`). `pickPill()` writes value + visual; `syncCheckinVisuals()` (~4885) re-derives all pill visuals from the hidden inputs — it is called after every `loadCheckinForDate()` (date-change handler, `showPage`, post-save). Never restore values without also calling it.
- **Mood** emoji row (1–5) → hidden `#mood`; **triggers** as `.trig-chip` multi-select (`TRIGGERS`/`TLABELS` arrays); **somatic symptoms** as checkboxes `som-<key>` driven by the `SOMATIC` array + `SLABELS` map (13 items incl. `hairloss`, `digestive`, `itchyskin`; last three are the "Intimate Health" subsection). Everything downstream (dashboard card, GP report counts) iterates `SOMATIC` generically — adding a symptom = add to both arrays + one chip div, nothing else.
- Numbers: `sleep` (hours, default 7), `hotFlushCount`, `nightWakings` (default 0). Free text: `notes` (per-day, stored on the check-in — distinct from the Notes tab).
- Save UX: button morphs to "✓ Saved" + `.save-success-message.show` for 2.5 s (`saveDelightMsg`). The message has explicit `line-height:1.6; padding:4px 0` because iOS Fraunces-italic metrics clipped it — keep those.

**Notes tab** (separate feature, localStorage key `notes`, JSON array): `{id: Date.now(), date: ISO, category, text, priority, askAtAppointment, answered}`. `saveNote/editNote/deleteNote/toggleNoteAnswered/renderNotes(filter)` (~6500–6650). Text is entity-escaped on render; category/priority are NOT — see pitfall below.

### 2. Strategic decisions, security & compliance

- Intimate Health symptoms carry an explicit on-screen privacy promise: "This stays completely private on your device." **That promise is now qualified by cloud sync** (the record syncs to Supabase like everything else) — same APP transparency flag as the disclaimer; surface to owner, don't silently reword.
- Date semantics are LOCAL day strings everywhere in this module. The UTC-parse windowing mismatch lives downstream (Module 1 §3) — do not "fix" it here by changing how dates are written; you'd corrupt the join key for upserts and cloud merges.
- `quickLog()` (~4985) is the low-friction path (hot flush / night sweat / mood dip from the Home modal): increments/sets minimal fields on today's record, stamps `loggedViaForm:false`. It must never set fields the user didn't express — that's what keeps the adherence semantics (Module 1) sound.

### 3. Tactical handoff

- **Live crash found during this session (unfixed):** `renderNotes` throws `TypeError: … 'charAt' of undefined` at the `n.priority.charAt(0)` badge line when a note lacks `priority` (and would similarly break on missing `category` classes). Reachable via backup **import** or **cloud merge** of a partial note object — the import validator only checks that `notes` exists, not per-note shape. Because `showPage('notes')` calls `renderNotes`, one malformed note bricks the whole Notes tab. Fix: default in render (`(n.priority||'low')`, `(n.category||'other')`) or sanitise on import/merge.
- The per-date reset list in `loadCheckinForDate` is the regression hotspot — history: sleep/hotFlush/bleeding/urinary/hrtTaken/notes were once missing (data bled between dates). It is complete today; keep it complete.
- Editing a quick-logged day via the form upgrades the record to `loggedViaForm:true` on save — intended.

### 4. Concrete execution checklist

1. **Fix the renderNotes crash** (non-protected function): add fallbacks for `priority`/`category`, and/or extend `importMyData`'s note merge to skip/normalise notes missing `id/text/date/category/priority`. Test by importing a backup with `{id:'x', text:'t', date:'2026-01-01'}` and opening the Notes tab.
2. When adding any check-in field, touch all four: HTML widget, `saveCheckin` (add-only line), `loadCheckinForDate` restore + blank-reset lines, and (if reportable) the GP report rows. Owner precedent: additive lines in protected functions are acceptable with sign-off; rewrites are not.
3. Never bypass `sliderSetters` to move a slider (writing the hidden input alone desyncs the visual thumb).

---

## MODULE 4 — CYCLE / PERIOD TRACKING (Bleeding Calendar)

### 1. System mental model

There is no separate cycle store — bleeding lives on the same per-day check-in records (`bleeding` field). The Calendar tab renders a month grid from `checkins`:

- `updateCalendar()` (protected, ~5560): reads `#calendarMonth`/`#calendarYear` (two **hidden** legacy `<select>`s — kept as state holders; the visible UI is the `‹ Month Year ›` arrow header, `calNavMonth(dir)` + `syncCalNavLabel()`, which rolls years and appends `<option>`s dynamically). Builds the grid off-DOM in a DocumentFragment; each day cell gets class `calendar-day <bleeding>` which drives the colour legend (none/spotting=#fecaca/light/medium/heavy).
- **Tap-to-cycle:** each cell's onclick advances bleeding through `['none','spotting','light','medium','heavy']`. If the day has no record it **pushes a stub** (`mood:0, brainFog:0 … loggedViaForm:false`, ~5616–5622) — these stubs are why every average/adherence consumer must filter (Modules 1, 6). `updateCalendarCell()` patches the single cell; full re-renders are skipped via the `State._calendarKey` month cache — **any code that mutates checkins outside `setCheckins` must null `State._calendarKey`** or the calendar shows stale data (the nav arrows already do this).
- Profile-level cycle context (`lastPeriod`, `menstrualStatus` incl. "Postmenopausal…") lives on the `profile` key and only surfaces on the GP report header.

### 2. Strategic decisions, security & compliance

- Calendar stubs are the accepted trade-off for one-tap logging; the invariant is that stubs are always `loggedViaForm:false` and slider fields `0` (never the form defaults of 5/7) so they're distinguishable. **Do not change stub shape** — three modules' filters depend on it.
- Bleeding data appears on the GP report (bleed-day counts, per-level day counts in Trends). Same sensitivity class as everything else; no special handling.

### 3. Tactical handoff / 4. Checklist

- Cache pitfall above is the only recurring trap. Nav rollover (Dec↔Jan) and logged-day highlight persistence across navigation are tested-good.
- If asked for cycle *prediction* or period-length analytics: that's new derived logic — build it read-only over `checkins`, filter `bleeding !== 'none'`, and remember stubs are legitimate bleeding data (that's their whole purpose) even though they're excluded from symptom averages.

---

## MODULE 5 — DATA STORAGE & PRIVACY LAYER

### 1. System mental model

Three tiers, strictly ordered:

1. **In-memory `State`** (~4504): memoised parses of localStorage (`checkins/hrt/profile/user/notes`) + caches (`_sample/_sampleValid`, `_calendarKey`). All reads go through `getCheckins()/getHRT()/getProfile()/getUser()` etc.; `setCheckins()/setHRT()` write both cache and localStorage and invalidate `_sample`. **When you write localStorage directly in tests or new code, you must null the relevant `State` fields** — half the "it didn't update" bugs in this app's history were stale `State`.
2. **localStorage — the primary, always-authoritative-locally layer.** Keys: `checkins` (array), `hrt` (object), `hrtHistory` (array), `profile`, `user`, `notes`, `userName`, plus flags (`iosDismissed`). Per-record schema version: `scaleV:2` stamps migrated check-ins (Module 3 scale). Migrations follow the **per-record stamp pattern** (`migrateScaleV2`, runs at startup and after every cloud merge) — idempotent, cloud-safe, never a global flag.
3. **Supabase cloud mirror** — one row per user in `user_data` (`user_id uuid PK = auth.uid()`, `data jsonb`, `updated_at`; RLS `auth.uid() = user_id` for all ops; table created manually via SQL in the dashboard — the client cannot create it). Sync engine (~7230–7400):
   - **Push:** `localStorage.setItem` itself is wrapped; writes to any `CLOUD_SYNC_KEYS` (`checkins,hrt,hrtHistory,profile,notes`) schedule a 2 s-debounced `pushToCloud()` (full-snapshot upsert). `visibilitychange→hidden` flushes pending pushes. Inert when `_cloudUserId` is null. `_cloudApplying` guard prevents merge-writes from re-triggering push scheduling mid-merge (writes during merge use the saved `_origSetItem`).
   - **Pull/merge:** on sign-in, `pullFromCloud()` → `applyCloudSnapshot()`. Merge rules (identical philosophy to backup import): check-ins merge **by date, local wins conflicts**; `hrtHistory` concat + dedup on `hormone|date|description`; `hrt` object and `profile` fill **only when local is empty** (profile per-field); notes dedup by `id` (fallback `text|date`). After merge: run `migrateScaleV2`, refresh views, push merged state back.
4. **Manual backup:** Settings → `exportMyData()` downloads `hertrace-backup-YYYY-MM-DD.json` (same snapshot shape + `app:'hertrace'`); `importMyData()` validates shape (`checkins` array, `hrt`/`profile` objects, `notes` present), confirms with a count, merges with the same rules, refreshes views. Error copy is owner-specified verbatim.

### 2. Strategic decisions, security & compliance ⚠️ (the app's main APP exposure lives here)

- **No encryption at rest, either tier.** localStorage is plaintext on device; `user_data.data` is plaintext jsonb guarded only by RLS + the publishable (anon) key model. For health data under APP 11 this is thin; at minimum it must not be *misrepresented*, which leads to:
- **APP 1/5 — misleading statements (LIVE):** clinical footer: *"All data is securely isolated on your local device"*; login card: *"Your data stays private on this device."*; Intimate Health: *"This stays completely private on your device."* All three predate cloud sync and are now **false for signed-in users**. Highest-priority copy fix, owner sign-off on wording.
- **APP 11/12 deletion gap (LIVE):** `deleteAllData()` clears only localStorage; the cloud row and auth session survive (details + fix pattern in Module 2 §4). "This cannot be undone" is doubly wrong — it *is* undone by the next sign-in.
- Data residency: Supabase project region wasn't chosen in-code; owner should confirm region for Australian-hosted preference (APP 8 cross-border disclosure).
- The `user` key shape `{name, email}` is written by auth and read by export/GP report — treat as API.

### 3. Tactical handoff

- **What failed before:** interception initially pushed during merges (fixed with `_cloudApplying`); profile import once required local-empty-whole-object and never refreshed the Settings form (now per-field merge + `loadProfile()`); notes import crash vector still open (Module 3 §3).
- **Merge is last-write-wins at the *snapshot* level** (upsert replaces the whole jsonb). Two devices editing offline → the later push wins wholesale for `hrt`/`profile`; only `checkins/hrtHistory/notes` get true merging, and only at *pull* time. Known limitation, accepted; don't promise otherwise.
- Console strings are the observability layer: "Cloud sync: data saved to Supabase", "Cloud sync push failed: …". Keep them — the owner debugs by console.

### 4. Concrete execution checklist

1. Fix the two LIVE compliance items (copy + cloud deletion) after owner decisions — exact mechanics in Module 2 §4.
2. Sanitise notes on import/merge (Module 3 §4.1) — it's a storage-layer validator gap.
3. Any new persisted feature: add its key to `CLOUD_SYNC_KEYS` **and** to `cloudSnapshot()` **and** to export/import **and** to the merge rules — four places, or data silently doesn't sync/back up.
4. Never rename protected keys; never write through `_origSetItem` outside the sync engine (bypasses cloud push).

---

## MODULE 6 — DASHBOARD & REPORTING

### 1. System mental model

Four read-only consumers over `checkins` + `hrt`/`hrtHistory` + `profile`, all rebuilt on navigation via `showPage(n)` hooks:

- **Dashboard** — `updateDashboard()` (protected, ~5380–5555): greeting/time-of-day (uses `userName`, sentinel `'there'` → generic greeting; a MutationObserver-driven streak leaf fills by streak tier 0.2/0.55/1.0 — it watches `#dashStreakText`, don't rename that id), This-Week snapshot (avg sleep/fog, bleed days), one insight line (priority chain — severity thresholds documented in Module 1 site 2), Life-Impact summary (`v > 6` = "felt the strain"; impact fields were always high=bad), detail cards with `trendBadge(s, field, up)` arrows (protected; `up=true` means rising is good — after the severity unification only `sleep` is `true`), top-6 somatic frequencies, HRT regimen card (em-dash separators — owner explicitly declined changing them).
- **Trends** — `renderTrends()` (~6280): range selector 7/14/30 (`#trendsRange`), "Based on N logged days", metric cards with canvas sparklines + `trendDir(field, upGood)` (only `sleep` is `upGood:true`), HRT-change markers from `hrtHistory`, trigger bars, bleeding pills, adherence bar (broken empty-state — Module 1 site 4).
- **Weekly Review** — `renderReflection()` (~6840): current-week window, quote fragments (severity-direction-corrected: fog ≥7 = "harder", patience ≥7 = "short supply"), week-over-week `compareNote`, GP-items list (Module 1 site 5), staggered card animation.
- **Reports, two-layer:** the **Report tab** (`renderReport`, in-app cards, the one with *correct* adherence null-handling) and the **printable GP report** (`exportData`, protected): builds HTML into `#gpReportPage` (an in-app page — never `window.open`), Back button + `window.print()`; `@media print` (~3308) hides chrome, resets the flex shell (`html/body/.container` → block/visible) and restores `table-layout:auto` (screen uses `fixed` + word-wrap so the 7-column HRT table fits phones — keep both halves of that pair). Scale-text strings in the tables are load-bearing clinical documentation — they were re-worded for the severity unification; any future scale change must update them in BOTH 7- and 30-day tables (they're duplicated strings, not shared).

### 2. Strategic decisions, security & compliance

- The GP report is the app's clinical output; the owner's standing acceptance test is **"no undefined, NaN, or blank values anywhere in the report"**. `avg()` in `exportData` treats missing fields as 0 (drags averages down when stubs/legacy records lack a field) — accepted historically, but it's the same class of honesty problem as the adherence bug; if the owner ever asks why averages look low, this is why.
- Patient identity on the report comes from `profile` (name/dob/lastPeriod/menstrualStatus) with the login email as contact — sensitive output; never add fields without owner request.

### 3. Tactical handoff

- Direction semantics are now uniform (high = bad for every slider metric; only `sleep` is high = good). The historical bugs here were exactly mismatched `up`/`upGood` flags — if you add a metric, set the flag consciously in BOTH `updateDashboard`'s `trendBadge` call and `renderTrends`' metrics array.
- All four consumers assume `State` caches are fresh; they're safe when navigation goes through `showPage` (which triggers the renders). Direct DOM pokes get overwritten on next render — test *after* triggering the render, a repeated gotcha.
- Rendering is innerHTML-rebuild for Trends/Review/Notes (was implicated in an iOS compositing drift, resolved by removing `-webkit-overflow-scrolling: touch`); keep heavy DOM work out of scroll handlers.

### 4. Concrete execution checklist

1. Apply Module 1 fixes (sites 1–5) — they land in this module's functions.
2. Owner-decision queue for reporting honesty: `avg()`-counts-missing-as-0, and the `Math.max(1,…)` idiom anywhere else it appears.
3. Full regression recipe after any reporting change (the established preview drill): seed known data → assert exact averages on Dashboard/Trends/Review/GP report → assert zero `NaN`/`undefined` in `#gpReportContent` → print-preview sanity for the table layout.

---

*End of handover. The two cross-cutting items I'd fix first as your predecessor: the adherence bug family (Module 1 — verified, checklist ready) and the deletion/disclaimer compliance pair (Modules 2 & 5 — needs owner decisions). Everything else is stable if you respect the protected lists, the stub invariants, and the State caches.*
