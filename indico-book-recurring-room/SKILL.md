---
name: indico-book-recurring-room
description: "Book a weekly/recurring Indico room series: per-date availability matrix, plan with exceptions, create series, verify occurrences."
---

# Book a recurring room series in Indico

Use for "book a room every Tuesday 14:30-15:30 until March for 10 people in 31 or 513". Builds on skill `indico-find-and-book-room` (read it for single-slot search, room-list cache, confirm/verify rules). App `indico`, connected as Alex Iribarren (login `airibarr`). Times Europe/Zurich.

Scripts: `scripts/indico-series-free` in this skill's folder, `scripts/indico-free-rooms` in `indico-find-and-book-room`'s. The folders depend on the agent program; find them once:
```
S=$(ls -d ~/.agents/skills/indico-book-recurring-room 2>/dev/null | head -1)
F=$(ls -d ~/.agents/skills/indico-find-and-book-room 2>/dev/null | head -1)
```
`F` empty → `indico-find-and-book-room` is not added to this agent: say so (single-date alternatives need it).

## 1. Parse
First date (next matching weekday, say it), last date (or "until X": last matching weekday <= X; no end given → ask), start/end time, interval (weekly=1, every other week=2), capacity, buildings, reason. Monthly/daily series: same approach, adapt the date step.

## 2. Quick check: rooms free on EVERY date (one call)
```
aurora-agent call indico --path /rooms/api/rooms/search capacity=10 start_dt=2026-12-01T14:30:00 end_dt=2027-03-30T15:30:00 repeat_frequency=WEEK repeat_interval=1 --raw
```
(add `building=513` to restrict; omit to search all CERN). start_dt = first date + start time, end_dt = LAST date + end time. `availability_days` lists every occurrence checked; `rooms` = ids free on all of them. With `repeat_frequency=NEVER` a multi-day range only checks the first day — don't do that.

## 3. Partial availability (usual case): per-date matrix
```
"$S/scripts/indico-series-free" FIRST LAST START END CAPACITY INTERVAL_WEEKS [BUILDING ...]
"$S/scripts/indico-series-free" 2026-12-01 2027-03-30 14:30 15:30 10 1 31 513 600
```
One all-CERN search per date (~0.5 s each), joined with the cached room list. Prints the dates checked, then per room (best 15): id, name, capacity, direct|NEEDS-CONFIRMATION, free N/M and the busy dates, most free dates first. Weekly steps only (at most 60 dates per run: split longer series); for a daily or monthly series call `"$F/scripts/indico-free-rooms"` per date instead.
Not used: `/rooms/api/rooms/<id>/availability` (ignores dates), `/rooms/api/booking/timeline` (blocked by gateway), `/rooms/api/calendar` over months (~2 MB/month, only for "who has it on date X").
Apply the same room sanity rules as the single-booking skill (skip "NOT A MEETING ROOM", warn on Openlab 513/R-070, prefer `direct` rooms, smallest that fits).
CERN annual closure (~20 Dec - 5 Jan) is not blocked in Indico: ask whether to skip those dates rather than book them.

## 4. Propose a plan
- A room free on all dates → propose it alone.
- Otherwise: main room = most dates free; for each busy date propose the best alternative room free that date (rerun `"$F/scripts/indico-free-rooms" DATE START END CAP BLDGS`), or skipping the date. Present as: "513/R-068 for 15 of 18 Tuesdays; 8 Dec, 15 Dec → 31/S-028; 2 Mar → skip?"
- Confirm explicitly: room(s), weekday, time, first/last date, interval, reason, exceptions.

## 5. Book (approval required in Aurora; never retry)
Series must go through the internal endpoint (book_room is single-slot only):
```
echo '{"room_id":137,"start_dt":"2026-12-01T14:30:00","end_dt":"2027-03-30T15:30:00","repeat_frequency":"WEEK","repeat_interval":1,"recurrence_weekdays":["tue"],"reason":"Linux team meeting","is_prebooking":false}' \
 | aurora-agent call indico create_booking --body -
```
- start_dt = first date+start, end_dt = last date+end (same convention as the search). Leave `user` out: books for Alex.
- Do NOT include dates where the room is busy in the hope Indico skips them: UNTESTED whether Indico rejects the whole series or skips conflicts. Split instead: book contiguous runs of free dates as separate series, and the exception dates as single bookings (`book_room ... username=airibarr` or create_booking with repeat_frequency NEVER).
- Room needing confirmation → `"is_prebooking":true`, tell the person it's pending owner approval.
- Scope refusal → say `full:everything` is needed and stop.

## 6. Verify
`aurora-agent call indico --path /rooms/api/bookings/<id> --raw` (id from the create answer, else find it in `list_my_upcoming_bookings` by room_id + start_dt). Check:
- `repetition` = [2, interval, ["tue"]] (2 = WEEK), `start_dt`/`end_dt` as requested
- `occurrences.bookings` keys = the booked dates (count them vs. plan); `occurrences.cancellations`/`rejections` empty; `is_accepted` true (or `is_pending` for pre-bookings).
Report: room, weekday/time, date range, number of occurrences, exceptions and their rooms, link https://indico.cern.ch/rooms/my-bookings.

## Cancelling one date later
`cancel_booking_occurrence` = POST /rooms/api/bookings/<id>/<YYYY-MM-DD>/cancel (approval). Whole series: POST /rooms/api/bookings/<id>/cancel. Confirm first.