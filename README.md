# aurora-skills

Skills for Aurora agents, served as an Aurora **skill library** (connect this repository under
Manage users › Agents › Skills › Skill libraries, folder left empty).

## Layout

Every skill is a folder at the top level of the repository:

```
<skill-name>/
├── SKILL.md          # frontmatter (name, description) + instructions
└── scripts/…         # optional files; commit scripts as executable (mode 100755)
```

Aurora only looks one level deep: a `SKILL.md` inside a category folder (`indico/<skill>/SKILL.md`)
is not found. Top-level files such as this README are ignored.

## Rules Aurora enforces

- **Folder name** = skill name: lowercase letters and digits in `-`-separated parts, at most 64
  characters, not starting with `aurora-`. If `name:` is set in the frontmatter it must equal the
  folder name.
- **Frontmatter**: YAML between `---` lines with a single-line `description` (at most 500
  characters); no anchors or aliases (`&`, `*`).
- **Body** at most 24,000 characters; body plus all files at most 32,000 characters (binary files
  count as their base64 length).
- **Files**: paths of `A-Z a-z 0-9 . _ / -`, at most 4 levels deep and 200 characters, no `..`,
  no other file named `skill.md`; each file at most 64 KiB; at most 50 files per skill.
- **No symlinks or submodules.**
- **Executable bit** comes from the git mode: `git add --chmod=+x <file>`, then check with
  `git ls-files -s <skill>/` (`100755`).

## Updating a skill

Agents keep the commit a skill was added at. After a push, Aurora proposes an update on the
Skill libraries page (checked every 15 minutes, or **Check now**); someone approves the diff before
agents see it.

## Skills

| Skill | What it does |
|-------|--------------|
| `indico-find-and-book-room` | Find a free CERN meeting room for one slot (one search for all buildings, cached room list), confirm, book once, verify; script `scripts/indico-free-rooms` |
| `indico-meeting-prep` | Brief on an upcoming or past Indico meeting (agenda, speakers, slides and minutes text, summary, action items); scripts `indico-event-brief`, `doc2text` (PDF/PPTX/DOCX) |
| `indico-search-talks-and-files` | Find talks, slides, files and minutes across Indico with the search API (type, category, person, date filters) and read them (uses `indico-meeting-prep`'s scripts) |
| `indico-book-recurring-room` | Book a weekly/recurring Indico room series with a per-date availability matrix, exceptions and verification; script `scripts/indico-series-free` (builds on `indico-find-and-book-room`) |
