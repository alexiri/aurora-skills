---
name: indico-find-and-book-room
description: "Find a free CERN meeting room in Indico for a date/time, capacity and buildings, then book it (one search call for all buildings, cached room list, confirm, book once, verify)."
---

# Find a free Indico room and book it

Use for requests like "set up a meeting for the cloud team on Friday at 10 or 11, room for 10 people in 31, 600 or 513".
App: `indico` (indico.cern.ch, connected as Alex Iribarren). Gateway adds the token: never send ak/apikey/signature.
Times are Europe/Zurich local time.

## 0. Parse the request
- Date: resolve weekday names to the NEXT occurrence after today (if today is that weekday, ask: today or next week?). Say the resolved date explicitly.
- Slots: "at 10 or 11" = two candidate slots 10:00-11:00 and 11:00-12:00. No duration given → assume 1 h and say so.
- Capacity: the number of people (the search filter is a strict `capacity >= N`).
- Buildings: list given, or none = any building. Do not ask if buildings were given.
- Reason/title: the topic ("Cloud team: secret handling").

## 1. Search: ONE call per slot, all buildings at once
Do NOT call search_rooms once per building and get_room_details once per room. Instead:
- `GET /rooms/api/rooms/search` WITHOUT `building` returns ids of every free room (all of CERN) → filter locally.
- `GET /rooms/api/rooms/` (list_all_rooms, ~160 KB, 225 rooms, full details) → cache once per day and join.

Helper script `scripts/indico-free-rooms` in this skill's folder (read-only, executable). The folder is `~/.agents/skills/indico-find-and-book-room`:
```
S=$(ls -d ~/.agents/skills/indico-find-and-book-room 2>/dev/null | head -1)
"$S/scripts/indico-free-rooms" 2026-11-27 10:00 11:00 10 31 600 513
"$S/scripts/indico-free-rooms" 2026-11-27 11:00 12:00 10 31 600 513
```
Arguments: `DATE START END CAPACITY [BUILDING ...]`. Output TSV: id, full_name, capacity, direct|NEEDS-CONFIRMATION, comments. It caches the room list in `/tmp/indico-rooms-<today>.json`.

Gotchas (verified 2026-11-21):
- ALWAYS `repeat_frequency=NEVER`, else busy rooms come back too.
- Search answer: `{"rooms":[free ids], "rooms_without_availability_filter":[all matching ids], "total":N}`. `total` counts busy rooms too: use `rooms` only. Busy = in `rooms_without_availability_filter` but not in `rooms`.
- `--path` rejects `?` in the path: pass query params as `name=value` args. Lists: comma-separated (`room_ids=177,33`).
- `start_dt`/`end_dt` same day, format `YYYY-MM-DDTHH:MM:SS`.
- `/rooms/api/rooms/<id>/availability` ignores date params (always returns the next 5 days from today): don't use it for a future date.

## 2. Pick candidates
Drop/flag rooms whose comments say "NOT A MEETING ROOM" (e.g. 31/3-009 coffee area) or warn bookings may be cancelled (513/R-070 Openlab Space: may be bumped for visits/board meetings — mention it).
Prefer: `direct` rooms (NEEDS-CONFIRMATION ones can't be booked with book_room; they become pre-bookings needing owner approval), smallest capacity that fits, the building order the person gave, and the earlier slot if both work.
Mention building-31 / 600 notes about after-hours access only if the meeting is outside working hours.
Show at most ~5 options, grouped by slot, e.g. "10:00: 513/R-068 (19), 31/S-023 (22)…; 11:00: 513/R-068 …".

If nothing is free: show what blocks the requested buildings that day and suggest nearby times:
```
aurora-agent call indico --path /rooms/api/calendar start_date=D end_date=D --raw > /tmp/cal.json
jq -r --slurpfile r /tmp/indico-rooms-$(date +%F).json --arg d D '.[] | .room_id as $id
 | ($r[0][]|select(.id==$id)) as $room | select($room.building=="600")
 | "\($room.full_name) (\($room.capacity)): " + ([(.bookings[$d]//[])[]
 | "\(.start_dt[11:16])-\(.end_dt[11:16])"] | sort | join(", "))' /tmp/cal.json
```
(The calendar returns ALL rooms, ignoring building/room_ids filters; rooms with no bookings that day may be absent → filter in jq. Booking objects carry `reservation.booked_for_name` and `booking_reason` — don't broadcast other people's booking details in shared channels unless needed.)
Then rerun the search for alternative slots (e.g. 09:00, 14:00) before proposing them — never guess availability from the calendar alone.

## 3. Confirm (always, booking is irreversible-ish)
Show: room full name + capacity, date, start-end, reason, who it's booked for. Wait for an explicit yes.
Bookings are made as the connected account (Alex Iribarren). If someone else asked, say the booking will be under Alex's name.

## 4. Book ONCE (both operations need approval in Aurora — tell the person to approve)
Preferred, official API (needs the Indico login, not the email; /api/user/ does not return it). Alex's login is `airibarr`. For others: CERN Phonebook, login = CN in the DN (uid attribute may come back empty):
`echo '{"filter":"(mail=Alex.Iribarren@cern.ch)","attributes":["displayName","mail"]}' | aurora-agent call cern-phonebook --path /search --method POST --body - --raw`
(note: phonebook mail is alex.iribarren@, Indico account email is alejandro.iribarren@.)
```
aurora-agent call indico book_room roomid=137 from=2026-11-27T10:00 to=2026-11-27T11:00 tz=Europe/Zurich reason="Cloud team: secret handling" username=airibarr
```
Fallback / recurring / needs-confirmation rooms (books for the connected person, no username):
```
aurora-agent call indico create_booking room_id:=137 start_dt=2026-11-27T10:00:00 end_dt=2026-11-27T11:00:00 repeat_frequency=NEVER reason="Cloud team: secret handling" is_prebooking:=false
```
(for a room that NEEDS-CONFIRMATION use `is_prebooking:=true` and say it's pending owner approval). If refused for scope: say which scope (`write:legacy_api` for book_room, `full:everything` for create_booking) and stop.
NEVER retry after an error/timeout: first verify with step 5, then ask.

## 5. Verify and report
`aurora-agent call indico list_my_upcoming_bookings` → find the entry with that room_id and start_dt (reservation.id). Report room, time, map link (`map_url` from room details, or https://maps.cern.ch/?n=['513/R-068']) and https://indico.cern.ch/rooms/my-bookings.

## 6. The "set up a meeting" part
- Indico cannot create events via API (`prepare_event` only prepares one for the person to finish; approval needed). Only offer it if they want an Indico event.
- `aurora calendar_invite` can put the meeting in the REQUESTER's calendar only (with location = room name, description = topic); they can forward it to the team. Offer this after booking.
- Team membership (e.g. "cloud team" e-group) can be looked up with CERN Phonebook if they want an attendee list; I cannot invite others myself.

Tools used to report: Indico (room search, room list, booking), optionally CERN Phonebook / calendar invite.
