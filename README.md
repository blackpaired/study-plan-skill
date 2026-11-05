# study-plan (Claude Code skill)

Turns lecture notes, Moodle videos and Anki decks into a day-by-day study plan and writes it
into the Anki add-on **Smart Study Plan**. Includes a daily routine (fixed times, sport) as day notes.

## Requirements
- Anki with the add-on "Smart Study Plan"
- Python 3 (Anki's `anki` library, ships with Anki 23.10+)
- Node.js (for the Moodle link reader), `markitdown` (for `to_md.sh`), `pdftoppm` for scanned PDFs

## Install
Copy this folder to `~/.claude/skills/study-plan/` (or any folder your Claude Code setup reads skills from).

Then ask Claude Code e.g. "make a study plan until my exam on 2026-12-14, 3 h a day, gym 17:00–18:00".

## Files
- `SKILL.md`: the instructions Claude Code follows
- `study_plan.py`: reads a copy of your Anki collection, schedules with the add-on's own code
- `moodle_links.mjs`: collects video links from a Moodle course (you log in yourself; only links, no downloads)
- `to_md.sh`: converts PDFs/Office files to Markdown to find chapters and page numbers

## Notes
- Nothing is downloaded from Moodle, and no password is ever typed by the tool.
- The tool never deletes cards or files and never resets Anki progress.
