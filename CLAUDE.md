# capstone — Goals & Activities system

This repo holds Ravi's personal Goals + Activities tracking system: a `Goals` agent that
manages his goals, and an `Activities` dataset that logs what he actually does, synced
hourly from his Granola voice notes with Jarvis. The two are linked so goal progress is
grounded in real logged activity, not just self-report.

## Data files

- `data/goals.json` — Ravi's goals. Schema and edit rules are owned by the `Goals` agent
  (`.claude/agents/goals.md`) — read that file for the authoritative schema.
- `data/activities.json` — activities synced from Granola. `id` is the Granola meeting
  UUID, `last_synced_at` / `last_synced_meeting_id` mark the sync watermark. Schema notes
  are in the file's `$schema_notes` field.

Both files cross-reference each other via `linked_goal_ids` / `linked_activity_ids`. Keep
both directions in sync when editing either.

## Instructions for Jarvis (this assistant, in this repo)

- **Whenever Ravi asks to add, update, close, or check a goal** — delegate to the `Goals`
  agent (Agent tool, `subagent_type: Goals`) rather than editing `data/goals.json`
  directly. It owns the schema and linking rules.
- **Never delete a goal or activity** to "clean up" — goals are additive; change `status`
  instead. Activities are an append-only log.
- **Ground progress claims in `data/activities.json`.** If Ravi asks "how am I doing on
  X", answer from linked activities (dates, what was actually logged), not vibes.

## Hourly Granola sync

An hourly trigger fetches new Granola meetings and appends them to `data/activities.json`
as activities, then asks the `Goals` agent to relink. The sync follows the automation
rules Ravi specified himself in the Jul 3, 2026 "Workflow consolidation... Jarvis
automation setup" Granola note (see `cac407d7-...` in `data/activities.json`):

1. **Dedupe first** — read `last_synced_meeting_id` / scan existing activity `id`s in
   `data/activities.json`; only fetch and append meetings not already present. Never
   re-append an existing id.
2. **New activity per new meeting** — `id` = meeting UUID, `title`/`date` from Granola,
   a short `category` + `tags` + `summary` derived from the meeting content, and initial
   `linked_goal_ids: []`.
3. **Relink** — after appending, run the `Goals` agent's linking pass so new activities
   get matched to relevant goals (and vice versa).
4. **Update the watermark** — set `last_synced_at` / `last_synced_meeting_id` to the
   newest synced meeting.
5. **Commit and push** the updated `data/activities.json` (and `data/goals.json` if
   relinked) to this branch, with a short commit message naming the date range synced.

This trigger is managed outside this repo (Claude Code scheduled trigger, hourly cron).
To change cadence or pause it, ask Jarvis to update or disable the trigger — don't try to
recreate the schedule by hand in this repo.
