# Agent mistakes log

Short notes for future sessions so the same issues are not repeated.

## 2026-04-30 — App shell / UI

1. **Grid `order` on main column** — Adding `order: -1` on the main column was unnecessary when DOM order is already `[main, rail]` with `grid-template-columns: 1fr var(--rail-w)`; it risked confusion. Removed; rely on DOM order.

2. **Mobile drawer + grid** — With `grid-template-columns: 1fr` and a `position: fixed` rail, the rail could still participate awkwardly in layout. Switched the shell to `display: block` below the breakpoint so only the main column flows in normal document layout.

3. **Drawer vs scrim z-index** — If the scrim (`z-index` 30) stacks above the rail (`z-index` 20), the drawer is not clickable. Raised the rail above the scrim on small screens only (`z-index: 40` on rail, scrim stays at 30).

4. **`withFallback` + `User.email`** — TypeScript treats `email?: string` as incompatible with an object that always includes `email: string | undefined` from an API map. Fix: build `user` with conditional spread or annotate `AuthSession` on the callback return.

5. **Left rail** — Rail is the first grid column (`var(--shell-rail-w) minmax(0, 1fr)`), DOM order `aside` then `main`, `border-right` on rail. Mobile drawer uses `left: 0` and `translateX(-100%)` when closed.

## 2026-04-30 — Demo backend/dashboard

1. **Mock fallback helper drift** — While adding real dashboard DTO fields, I referenced `buildRecentAttendance`, but the existing mock helper is named `buildAttendanceRecords`. Fixed the mock fallback to reuse the existing helper and reran frontend typecheck.

## 2026-04-30 — Dashboard restructure

1. **Frontend ⇄ backend endpoint drift** — Several services were calling endpoints that did not exist (`/dashboards/parent`, `/dashboards/proctor`, `/marks?studentId=...`, `/timetable?role=...`). Fixed by aligning service paths with what `backend/src/routes/*` actually mounts and by mapping response envelopes (`{ slots }`, `{ fees }`, `{ attendance }`, `{ marks }`) explicitly inside the service layer instead of letting raw shapes leak into UI types. New parent/proctor endpoints were added under `/api/dashboard/{parent,proctor}` to match the singular path.

2. **Per-period attendance** — The earlier seed wrote one attendance row per student per day (period 0). Replaced with per-timetable-slot rows so the student weekly grid can render P/A/L/E for each subject independently. Used `prisma.attendance.createMany` in 1 000-row chunks (~18 k total) instead of per-row inserts; the original approach was slow at this scale.

3. **Tabs component is uncontrolled** — `Tabs` accepts `defaultTabId` only and renders `content` from each tab item. Using a controlled `activeId` prop fails compile. Workaround: for the timetable view switcher I used a small button group with `aria-selected`, since the tab content depends on day-state that lives outside the Tabs.

4. **vite/plugin-react lockfile mismatch** — The committed `package-lock.json` had `frontend/node_modules/@vitejs/plugin-react@6.0.1` (which peers vite ^8) even though `frontend/package.json` says `^4.3.3`. This forced an install of vite 8 with rolldown native binding that fails on Windows. Resolved by deleting `package-lock.json` and reinstalling — npm then resolved correctly to vite 5.4.21 + plugin-react 4.7.0. Watch for this when reviewing lockfile diffs.

5. **TimetableSlot day union missed `SUN`** — Code that converts `Date.getDay()` (0–6) to a TimetableSlot["day"] needs `SUN`. Type was `MON..SAT` only. Extended to include `SUN` so today-detection on Sundays does not break.
