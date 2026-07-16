# Attendance Page — Bug Fixes

**Scope:** Code review + fixes for commit `49ba728` ("feat(founder): status-aware attendance + forgotten-checkout correction") in `D:\leadpilot-founder`.

**Repos touched:** Frontend only (`D:\leadpilot-founder`). The backend (`D:\leadpilot-backend`, commit `61c2484`) was **read** to confirm the API contract (status values, validation rules, response shape) but **no backend files were changed** — all 11 issues were frontend logic/UX bugs, not API contract mismatches.

**Files changed:**
- `apps/founder/src/app/dashboard/telecallers/attendance/page.tsx`
- `apps/founder/src/components/ui/Badge.tsx`
- `apps/founder/src/lib/utils.ts`

**Verification caveat:** `node_modules` isn't installed in this environment (no `pnpm`/`tsc` available), so changes were verified by careful manual re-reading against the project's `tsconfig.json` (`strict: true`) rather than an actual build. Before merging, run:
```
pnpm install
pnpm --filter founder build   # or: pnpm --filter founder exec tsc --noEmit
```

---

## 1. Modal race condition could discard or misattribute a founder's edits — **fixed**

**What it caused in the app:** A founder opens "Set check-out" for telecaller A, clicks Save, then — while that request is still in flight — closes the modal (X or Cancel, neither was disabled during save) and opens "Set check-out" for telecaller B. When A's request finally resolves: if it succeeded, it force-closed B's modal and silently discarded whatever the founder had just typed for B; if it failed, A's error message ("Check-out must be after check-in") appeared inside B's modal, wrongly blaming B's input.

**Fix:** The modal's close (X) and Cancel button are now disabled/no-op while `saving` is true, so a founder can no longer switch to a different record mid-save. The race is closed at the source instead of patched after the fact.

## 2. "Set check-out" on an active shift always failed — **fixed**

**What it caused in the app:** The per-row "Set check-out" link appeared for telecallers who were still actively clocked in (`on_shift`), not just for missed check-outs. Clicking it pre-filled the time picker with the telecaller's own check-in time (there's no auto-capped time for an active shift). If the founder clicked Save without changing anything — a very natural thing to do, since the field already showed a real timestamp — the backend rejected it every time with "Check-out must be after check-in," with no explanation of why the untouched default could never work.

**Fix:** The action (both the table's inline link and the underlying handler reachability) is now restricted to `auto_closed` rows only — the one status this feature was actually built to correct. Active shifts no longer show a correction action at all.

## 3. A successful correction could be reported to the founder as a failure — **fixed**

**What it caused in the app:** After successfully saving a correction, the app used to reload the entire attendance list from the server. If that follow-up request happened to fail (slow network, brief server hiccup), the founder saw a red "Failed to load attendance" banner immediately after saving — even though their correction had already been saved. The row would also still show its old (pre-fix) status, since the failed reload never updated it, making it look like the fix hadn't taken and inviting the founder to redo it.

**Fix:** The save handler now updates the corrected row directly from the data the save API call already returns, instead of triggering a second network request. There's no longer a second request that can fail and contradict a successful save.

## 4. Same fix also removed a wasted network round-trip — **fixed**

**What it caused in the app:** Every single correction re-fetched and re-rendered the entire telecaller roster (blanking the table back to "Loading attendance…") just to reflect the one row that changed — an unnecessary full-page flash and an extra server request every time.

**Fix:** Same change as #3 — the corrected record from the save response is patched directly into the on-screen list.

## 5. The "currently on shift" count and the table could silently disagree — **fixed**

**What it caused in the app:** Two different parts of the page derived a telecaller's status from the same data in two different ways. If a record ever arrived without a `status` field (e.g. mid-deploy, while an older backend instance was still briefly serving traffic), the table would show that row as "On shift" while the summary banners' counts would completely ignore it — so the "N telecallers currently on shift" banner and the missed-checkout alert could both undercount relative to what the table displayed, and a genuinely missed checkout could disappear from the alert entirely instead of prompting a correction.

**Fix:** Both places now read the same `status` value the same way — no more separate ad-hoc guesswork in the table.

## 6. An unrecognized status value could crash the whole page — **fixed (hardened)**

**What it caused in the app:** Nothing today (the backend currently only ever sends 3 known status values) — but if a future backend change ever introduced a new status before this page was updated to match, or any bad data slipped through, the page had no fallback: it would throw and crash the entire attendance table, with no error boundary anywhere in the app to contain the damage to just this page.

**Fix:** The status→color lookup now falls back to a safe default instead of crashing if it ever sees a value it doesn't recognize.

## 7. The "missed checkout" banner could show a stale row while the table said "loading" — **fixed**

**What it caused in the app:** Right after a correction, while the page was mid-refresh, the "on shift" banner correctly hid itself, but the amber "missed checkout" banner did not — so it could keep showing the just-fixed telecaller (with a live "Set check-out" button still pointing at the old record) while the table underneath displayed "Loading attendance…", making a successful fix look like it hadn't worked.

**Fix:** The missed-checkout banner now hides during a reload, consistent with the "on shift" banner right above it.

## 8. Saving an untouched auto-filled time could silently lose up to a minute of precision — **fixed**

**What it caused in the app:** The time picker only showed hours and minutes, not seconds. So if a founder opened "Set check-out" on an auto-closed shift and saved the pre-filled time without adjusting it, the value actually saved could be up to 59 seconds earlier than the real auto-closed timestamp — a small but silent loss of precision in exactly the data this feature exists to let founders enter accurately.

**Fix:** The time picker and the underlying conversion now preserve seconds, so an untouched save reproduces the exact original timestamp.

## 9. Duplicated color logic between the status dots and the existing Badge component — **fixed**

**What it caused in the app:** Nothing user-visible yet, but a maintenance trap: this page hand-rolled its own copy of the emerald/blue/amber color mapping that the shared `Badge` component already defines. If a future design change updated Badge's colors, this page's dots would silently fall out of sync since there was no way to know they needed the same update. Separately, a `label` field ("Completed"/"On shift"/"Auto-closed") was defined for every status but never actually used anywhere — dead code that looked like it controlled visible text but didn't.

**Fix:** The shared color mapping now lives in one place (`Badge.tsx`, exported and reused by both the Badge component itself and this page); the unused `label` field was removed.

## 10. `toLocalInput` was a third hand-rolled copy of the same local-date logic — **fixed**

**What it caused in the app:** Nothing broken yet, but real risk: this was the third independent copy in the codebase of "extract local date/time parts and zero-pad them," alongside `DateRangePicker.tsx` and the main dashboard page — and `DateRangePicker.tsx` carries an explicit comment noting that the naive alternative (`toISOString()`) already caused a real IST/UTC date-rollback bug once before. A third un-unified copy meant any future timezone fix had to be remembered and applied in three separate places.

**Fix:** Extracted a single shared `toLocalDateTimeInput` helper into `@/lib/utils.ts` and switched this page to use it. (`DateRangePicker.tsx` and the dashboard page were left as-is — consolidating those too would be a separate, broader change outside this diff's scope.)

## 11. The correction modal's time field had no label — **fixed**

**What it caused in the app:** A screen-reader user opening "Set check-out time" heard no field name announced when focusing the date/time input — unlike every other input in every other modal in this app, which all have visible labels.

**Fix:** Added a visible "Check-out time" label tied to the input, matching the pattern used elsewhere in the app (e.g. the team invite modal).

---

## Not changed (by design)

- **Backend (`d:\leadpilot-backend`)** — untouched. The API contract (status enum, validation, auth) was already correct; every issue found was on the frontend's side of that contract.
- **`DateRangePicker.tsx` / dashboard `iso()` helper** — left as separate hand-rolled implementations; only this page was switched to the new shared helper, to keep this fix focused.
