---
name: indico-meeting-prep
description: "Brief on an upcoming or past Indico meeting: find it in Alex's categories, agenda, speakers, slides/minutes text, summary and action items."
---

# Meeting prep briefing from Indico

Use for "what's on the agenda for the next section meeting?", "are the slides up for Friday's ITTF?", "summarise last week's Linux meeting / action items". App `indico` (export API = official, reliable).

Scripts are in this skill's folder (read-only, executable): `scripts/indico-event-brief` and `scripts/doc2text`. The folder depends on the agent program; find it once:
```
S=$(ls -d ~/.agents/skills/indico-meeting-prep 2>/dev/null | head -1)
```

## 1. Find the event
Alex's favourite categories (verified 2026-11-21; refresh with `aurora-agent call indico --path /user/api/favorites/categories --raw | jq -r '.[]|"\(.id)\t\(.title)"'`):
- 15233 CD-CLI Section Meetings ("section meeting", "CLI meeting")
- 11474 Linux Team Meetings (Tue 14:30, 31/2-028)
- 15350 Cloud Infrastructure Meetings (Cloud Sprint Review etc.)
- 15152 CD Group and Technical meetings (IT-CD Group meeting)
- 17199 SLM; 11233 Ironic; 16529 Cloud Network; 15676 Public Cloud
- 2958 ITUM; 3407 ITTF; 5419 ITLT; 21502 OTF; 21218 ITTC; 7443 ITSC; 9147 IT Department Meetings; 691 CNIC; 82 CERN Computing Seminar; 13390 Linux Future Committee

Next/last event in a category (several: `15233-11474`):
```
aurora-agent call indico --path /export/categ/15233.json from=2026-11-21 to=2026-12-21 --raw | jq -r '.results|sort_by(.startDate.date+.startDate.time)[]|"\(.id) \(.startDate.date) \(.startDate.time[0:5]) \(.title)"'
```
"Next" → first with date >= today; "last" → use from=today-30d to=today, take the latest. All followed events in a week: `--path /export/categ/favorites.json from=.. to=..` (also includes some non-favourite categories, e.g. favourite events).
Unknown category name → `aurora-agent call indico --path /category/search q=<name> --raw`. Unknown event title → `--path /export/event/search/<term>.json` or the search skill `indico-search-talks-and-files`.

## 2. Read it
```
"$S/scripts/indico-event-brief" EVENT_ID
```
Prints header (time, room, chairs, URL), description (HTML stripped), links found in the description, agenda (time, title, duration, speakers, abstract, per-talk attachments and minutes), "material on N" of the agenda items, the event minutes (Indico notes) and the event's own attachments. Each file attachment ends with `-> /export/event/<event>/material/<folder>/<attachment>.bin`, the path to download it. Raw source, if you need a field the brief leaves out: `--path /export/event/<id>.json detail=contributions` → `.results[0]` with `contributions[]`, `folders[].attachments[]` (event and per contribution), `note.html`.

## 3. Read slides/minutes when asked to summarise
- Download with the path the brief printed: `aurora-agent call indico --path /export/event/<event>/material/<folder_id>/<attachment_id>.bin --out /tmp/f.bin`. The `/event/.../attachments/...` URL itself is NOT reachable through the gateway.
- Text: `"$S/scripts/doc2text" /tmp/f.bin 20000` (PDF, PPTX with speaker notes, DOCX; PPTX/DOCX need nothing installed, PDF uses pypdf if present, else `pdftotext`; if neither: `pip install --user pypdf`). Old `.ppt`/`.doc` are not supported: use a PDF version if there is one. Prefer the PDF when both PDF and PPTX exist. The output says when it was cut. Big decks (100+ pages): read the first pages + summary slides, say so.
- `link` attachments (videos.cern.ch recordings, Google/CodiMD docs) can't be fetched: list them.
- Many CLI/Linux meetings keep agenda+minutes in CodiMD (codimd.web.cern.ch, linked in the description): I can't read it (SSO). Say so and give the link.

## 4. Report
Briefing: title, date/time, room (+ Zoom/link if in description), chairs; agenda as a short list with speaker and slot; which talks have slides yet ("slides: 2/5 uploaded"); for summaries: 3-6 bullets per talk, plus action items/decisions if minutes exist (attribute them to owners as written). Always include the event URL. Mention if Alex himself has an agenda item (e.g. "News & SLM Input — you, 50 min").
For "before the meeting" prep, optionally offer a calendar reminder (aurora calendar_invite, only to the requester).
