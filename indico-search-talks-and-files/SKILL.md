---
name: indico-search-talks-and-files
description: "Find talks, slides, files and minutes across Indico with the search API (type, category, person, date filters) and read the files."
---

# Find talks, slides and minutes in Indico

Use for "find the slides from that AlmaLinux talk at ITUM", "who presented about OpenStack upgrades last year?", "minutes mentioning X". App `indico`; Indico's search (internal endpoint, verified 2026-11-21).

Reading files uses the scripts of skill `indico-meeting-prep` (`doc2text`, `indico-event-brief`). Find its folder once:
```
M=$(ls -d ~/.agents/skills/indico-meeting-prep 2>/dev/null | head -1)
```
Empty → that skill is not added to this agent: searching still works, but say that reading files needs `indico-meeting-prep`.

## 1. Search
```
aurora-agent call indico --path /search/api/search q=AlmaLinux type=attachment category_id=2958 sort=mostrecent --raw > /tmp/s.json
```
- `type` is REQUIRED (else HTTP 422): `event`, `contribution` (talks), `attachment` (files), `event_note` (minutes), `category`.
- Narrowing (all verified): `category_id=<id>` (e.g. 2958 ITUM, 3407 ITTF, 5419 ITLT, 82 Computing Seminar, 1805 HEPiX, 15233 CLI section; see skill `indico-meeting-prep` for the list), `person_name=Alex Iribarren` (speaker/author/uploader), `start_range=[2026-01-01 TO 2026-11-21]` (also `[* TO 2025-01-01]`), `sort=mostrecent` (default = relevance), `page=2` (10 results/page; `pages` and `total` in answer).
- The answer's `aggregations` (category, person_name, start_range, venue, ...) give bucket counts and the exact `filter` value — use them to refine a broad query instead of paging through hundreds of hits.
- Start specific: keyword + type + category/person. If total > 30, refine; if 0, drop filters one at a time, try synonyms (Alma/EL9/RHEL), or switch type (event ↔ contribution ↔ attachment).
- Results are public + what Alex can see. Don't dump unrelated hits (e.g. other experiments' minutes) into chat.

## 2. Read the hits
```
jq -r '.total, (.results[] | [.type, (.title // .filename), (.start_dt // .modified_dt // "")[0:10], ((.event_path // [])|map(.title)|join(" > ")), .url] | @tsv)' /tmp/s.json
```
Field shapes:
- event: `event_id, title, start_dt (UTC!), location.room_name, persons[].name, url`
- contribution: `contribution_id, event_id, title, persons, start_dt, url`
- attachment: `event_id, contribution_id, folder_id, attachment_id, filename, attachment_type (file|link), modified_dt, user.name, event_path[], url`
- event_note: `event_id, contribution_id, content (plain text), url`
Times: `start_dt` is UTC; convert to Europe/Zurich when reporting. URLs are relative: prefix https://indico.cern.ch.

## 3. Open the file (if asked what it says)
`aurora-agent call indico --path /export/event/<event_id>/material/<folder_id>/<attachment_id>.bin --out /tmp/f.bin`, then `"$M/scripts/doc2text" /tmp/f.bin 20000` (PDF, PPTX with speaker notes, DOCX; the output says when it was cut). The `url` field itself (/event/.../attachments/...) is not reachable through the gateway. `link` attachments can't be fetched: give the URL.
Whole event context (agenda, other slides, download paths): `"$M/scripts/indico-event-brief" <event_id>`.

## 4. Report
Top 3-5 matches: title, speaker, event, date (local), and direct link https://indico.cern.ch<url>. If summarising content, say which file/pages you read. If ambiguous (several talks with that name), list them and ask rather than guess.

Other lookups: events by title only → `--path /export/event/search/<term>.json` (official API); category by name → `--path /category/search q=<name>`.
