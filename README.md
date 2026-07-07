# jsonata-syntax

A teaching skill for **JSONata** — the JSON query-and-transform language — built to teach the *mental model*, not just list functions.

Drop `SKILL.md` into any agent that loads Markdown skills (Claude Code and compatible harnesses read `SKILL.md`). The repo root **is** the skill. In `actions.json.dev` it is vendored as a submodule under `skills/jsonata-syntax`, next to `write-actions-json` (which it complements — JSONata is the expression language inside actions.json map workflow/projection slots).

## What it covers

- The sequence model (why a singleton result silently collapses to a scalar) — the root of most JSONata surprises.
- Five progressive stages: navigate (paths) → filter (`[ ]`) → combine (operators) → reshape (object/array constructors) → **program** (blocks, bindings, first-class functions, the `$map`/`$filter`/`$reduce`/`$sift` trio, chaining `~>`).
- The gotchas that actually bite: singleton-collapse, `$exists` vs silently-false `= null`, filter-binds-tighter-than-dot.
- A grounded function cheat-sheet and an end-to-end worked transform.
- Notes for embedding JSONata in `actions.json` map slots (whole-string `{% %}` slots, `input`/`steps`/`item` bindings).

Every example is verified against a real JSONata engine. See "Test it, don't guess it" in `SKILL.md`.

## Contributing

This skill is meant to keep improving. Add worked transforms, more war-story gotchas, and host-boundary notes (Step Functions `{% %}` + `$states.input`, Node-RED `$flowContext`, Stedi/Truto/Kestra context). Ground every new example against a real engine before committing.

## License

MIT — see [LICENSE](LICENSE).
