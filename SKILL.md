---
name: study-plan
description: Turns lecture notes, Moodle videos and Anki decks into a day-by-day study plan until the exam and writes it into the Anki add-on "Smart Study Plan". Includes a daily routine (fixed times, sport) as day notes. Use for "study plan", "exam plan", "exam prep", "plan this until my exam", "put this into my Anki", "add the videos to my plan".
---

# Study plan – from your materials to a finished plan in Anki

The add-on **Smart Study Plan** (Anki: Tools → Study Plan…) shows a plan per day: reviews, script pages, videos (as clickable links), new cards and free-text notes. This skill builds that plan from your files, Moodle course and Anki decks.

The user gives: **what** (files, Moodle course, decks), **until when** (exam date) and **how much time per day**. You do the rest.

Language: answer in the language the user writes in. The plan file and the skill itself are in English.

**Running in a chat app** (no local Anki, no shell): this skill needs local access to Anki's add-on folder and collection. Say so briefly and offer the plan as a day-by-day Markdown table instead (same rules for topics and videos), which can be entered in the add-on by hand.

## Helper scripts (this folder)

Always go through these, never touch the Anki collection directly:

```bash
python3 -I <skill-folder>/study_plan.py --addon <addon-folder> addon                 # existing plans
python3 -I <skill-folder>/study_plan.py --addon <addon-folder> decks [word]          # decks with card counts (total / new)
python3 -I <skill-folder>/study_plan.py --addon <addon-folder> build <plan.json> --dry-run   # schedule and show, save nothing
python3 -I <skill-folder>/study_plan.py --addon <addon-folder> build <plan.json>     # schedule and save into the add-on
python3 -I <skill-folder>/study_plan.py --addon <addon-folder> show <plan-id>        # saved plan day by day
python3 -I <skill-folder>/study_plan.py --addon <addon-folder> replan <plan-id>      # spread again from today on
node <skill-folder>/moodle_links.mjs login "<course-url>"                            # login window, the user signs in
node <skill-folder>/moodle_links.mjs videos "<course-url>" --deep --json <file>
bash <skill-folder>/to_md.sh "<folder-or-file>"                                      # PDFs/Office files to Markdown
```

`<addon-folder>` is the folder that contains `planner.py`, `store.py` and `user_files/` (e.g. `~/.local/share/Anki2/addons21/<id>/plan` on Linux). Find it with `find ~ -name planner.py -path "*addons21*"`.

`study_plan.py` reads Anki from a **copy** of the collection (Anki may stay open) and schedules with the add-on's own code, so the result matches "Create plan" in the add-on. An existing plan is backed up first; finished items are kept.

## Ground rules

1. **Link videos, never download them.** The plan gets the Moodle link. Never save files from Moodle.
2. **Invent nothing.** Video titles, lengths, chapters and page numbers only from Moodle, the materials or the decks. If a length is missing, use `defaultMin` and say so.
3. **The user logs in.** Never type a password or fill in a login page. If a course needs a login, open the login window and let the user sign in.
4. **Dry run first, then save.** Always `--dry-run` first, show a summary, fix overload, then save.
5. **Respect existing plans.** If a plan for the same exam exists, extend it (same `id`, keep sources, topics and finished items).
6. **Never reset Anki progress.** Do not "reset" or "forget" decks to start over. Use a later `start` date or `replan` instead.
7. **Never delete files or cards without asking.** Copies of files are fine; deleting originals or notes is a separate, explicit decision.

## Workflow

### 1. Take stock
- `addon`: which plans exist? If one fits the exam, take it as the base (extend it).
- Convert materials to Markdown with `to_md.sh`; the table of contents gives topics and page ranges. For scans, render pages as images (`pdftoppm -r 100 -png -f 1 -l 6 file.pdf <temp>/toc`) and read them yourself. Never modify the originals.
- **Page numbers:** printed page numbers often differ from PDF page numbers. Compare one footer with the PDF page to get `pageOffset` (PDF page = printed page + offset). Use printed numbers in topics.
- `decks <word>`: matching Anki decks and subdecks with the number of new cards. Add up the **new** cards per topic to see the real workload (`new cards × minPerUnit`).

### 2. Ask briefly (only what is missing)
- **Exam:** date (and time if known), interim dates (tests, practicals)
- **Study time per day** and **days off**
- **Anki decks:** which decks belong to the exam; which are only reviewed (`reviewOnly`); which overlap and should be left out
- **Priorities:** topics the user wants first (they become the first topics in the list)
- **Start date:** today, or a later day (e.g. a pause). The start is a date, not a reset.
If the user says "just do it": start tomorrow, 4 h/day, no days off, and say so.

### 3. Get the videos from Moodle (links only)
```bash
node <skill-folder>/moodle_links.mjs videos "<course-url>" --deep --json <temp>/course.json
```
- Exit code 2 ("Not logged in"): run `login`, ask the user to sign in, run `videos` again.
- Use only entries with `video: true`. Skip files and interactive tasks unless the user wants them (then as `tasks`).
- Plan time per video: real length × 1.3, rounded up (pausing, rewinding). Mention this in the summary.
- If neither the scripts nor the browser work: ask the user to save the course page (Ctrl+S) and read the HTML.

### 4. Build topics (the core)
A **topic** bundles what belongs together: script pages + videos + Anki subdeck. The add-on schedules each topic as **read → video → new cards**, so the video is watched on the day its cards are learned.
- Order topics by the user's priorities first, then by course structure (`"order": "listed"`).
- For each topic, the matching subdeck as a search: `"anki": {"<source-id>": "deck:\"Deck::Subdeck\""}`. Several subdecks: join with ` OR `.
- Assign each video to exactly one topic. Videos without a topic are scheduled at the end.
- Interim dates: `milestones`; topics linked via `"milestone": "<id>"`; practical sessions with `"prep": true`.
- **Written exam questions:** if the written part is the bottleneck, do not leave the question banks to the last days. Put them into the plan from the middle of the period on (e.g. a fixed block every second day).

### 5. Daily routine (day notes)
The add-on shows a free-text note per day. Use it for a fixed daily routine that the planner cannot schedule, e.g. sport, breaks, or a fixed time table:

```json
"dayNotes": {
  "YYYY-MM-DD": "Routine: 08:00 Anki reviews (due cards first) · 09:00 video of the topic · 09:30 script pages · 10:30 new cards · 13:00 practice questions · 17:00–18:00 sport (1 h)"
}
```
- Write the same routine for every day (generate the dict in a short script) and adjust the days where it changes.
- Fixed activities are notes, not scheduled items. Set `minutesPerDay` to the study time *without* the fixed activities, so the planner does not fill the sport slot.

### 6. Write the plan file
Structure (fields as in the add-on):
```json
{
 "id": "my-exam", "title": "My exam", "subtitle": "short summary",
 "exam": "YYYY-MM-DD", "examTime": "09:00", "start": "YYYY-MM-DD",
 "minutesPerDay": 240, "offWeekdays": [6], "dayMinutes": {"YYYY-MM-DD": 120},
 "dayNotes": {"YYYY-MM-DD": "Routine: ..."},
 "milestones": [{"id": "m1", "name": "Test 1", "date": "YYYY-MM-DD", "time": "", "prep": false}],
 "sources": [
  {"id": "script", "type": "pages", "name": "Script", "file": "/path/script.pdf", "pageOffset": 4, "minPerUnit": 5},
  {"id": "vid", "type": "videos", "name": "Moodle videos", "defaultMin": 10, "minPerUnit": null,
   "videos": [{"id": "v1", "name": "Topic A video", "min": 18, "url": "https://moodle.example.edu/mod/..."}]},
  {"id": "cards", "type": "anki", "name": "Anki deck", "decks": ["Deck::Topics"], "minPerUnit": 1.5},
  {"id": "table", "type": "anki", "name": "Reference table (review only)", "decks": ["Deck::Table"], "minPerUnit": 0.25, "reviewOnly": true}
 ],
 "topics": [
  {"id": "k1", "name": "Topic A", "pages": {"script": [1, 10]}, "videos": {"vid": ["v1"]},
   "anki": {"cards": "deck:\"Deck::Topics::Topic A\""}, "milestone": "m1"}
 ]
}
```
- `offWeekdays`: 0 = Monday … 6 = Sunday. `dayMinutes`: exceptions for single days.
- `minPerUnit`: minutes per page / per new card. For videos, the video's `min` counts.
- IDs: letters, digits, `-`, `_` only. For an existing plan, keep its source and topic IDs.
- Keep personal paths and Moodle links out of anything you share publicly.

### 7. Dry run, fix, save
1. `build <file> --dry-run`. Read the notes at the end ("too full", "reviews alone take more time") and adjust time, `maxNew` or scope with the user.
2. Check that videos fall on the same day as the cards of their topic.
3. Show the user a summary: number of days, study hours per day, first days in detail, the rest summarised.
4. `build <file>` (saves, creates a backup).
5. Tell the user: close and reopen the Study Plan window in Anki. Videos open with "open ↗"; Anki items are ticked automatically, pages and videos by hand.

**Other devices:** recent add-on versions keep the plan in the Anki collection, so a normal AnkiWeb sync moves it to every device.

### 8. Later
- **Extend** (new videos, new deck, different date): take the plan JSON from the add-on, change it, `build` again.
- **Fell behind**: `replan <id>` spreads everything open from today on.
- **Archive an old plan:** set `"status": "archived"` in its JSON file in `user_files/plans/` (keep a backup copy first).
- **Duplicate cards:** check on a copy. In Anki, use Browser → Notes → Find Duplicates and delete there; then `replan`.

## What to avoid
- Opening or changing the real Anki collection – only through the scripts (on a copy).
- Resetting Anki decks or scheduling data.
- Downloading videos or saving Moodle content.
- Typing a password or filling in login pages.
- Saving without `--dry-run`.
- Creating a new plan when one already exists for the same exam.
- Deleting files or cards without the user's explicit confirmation.
