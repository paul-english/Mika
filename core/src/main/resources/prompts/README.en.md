# prompts/: the prompts shipped with the app

All fixed text the model sees lives here; code no longer embeds it as strings. It is read
uniformly through `core.prompt.Prompts` / `core.prompt.Feedback`. A `{{name}}` in the text
is filled in by the code that reads it (limits, allowed values, on-screen names, etc.);
a missing value is an error, not a placeholder sent to the model.

## Layers

| Directory | What it is | Where it goes | Who reads it |
|---|---|---|---|
| `system/identity.md` | P0 identity and stance: act for the user, within their authorization, look before acting, don't pretend, say when you don't know | start of the system prompt | `SystemPrompt.identity()` |
| `system/principles.md` | P1 general principles: whose word counts, how to run a task, answer from evidence, obstacles, finding things, safety; true for every screen and capability | right after identity | `SystemPrompt.identity()` |
| `capabilities/` | P2 capability descriptions, one per capability; `general.md` is the general-tool rules (report failure, ask, confirm) and how a task ends — general tools are always present, so it always follows the principles; `groups.md` says which entry each group of tools comes in through (listing only groups offered in this task); `handover.md` is the hand-to-user / pre-checkout confirmation that appears only once a screen group is expanded; the phone, web and sandbox sections are only assembled when their group is expanded; `app-access.md` is the app-access rules, appended to the phone section only when the user has disabled apps | system prompt | `SystemPrompt.assemble()` |
| `tools/` | P3 one file per tool (the tool's definition `description`): how to use it, parameters, what to do with the result; when several tools share a template, the tool names its `promptFile` | tool definitions | the tools themselves |
| `entries/` | short description of an entry tool while its group is collapsed (one or two lines: what it does, what tools appear after you call it); once expanded it is replaced by the full `tools/` text, same parameters | tool definitions | `ToolRegistry` |
| `feedback/` | feedback from tool results and the main loop to the model: hints, error text (`failure.<errorCode>`), reminders, source labels for references; one key per entry | tool results, context messages | tools, `AgentLoop` |
| `roles/` | prompts for internal roles: verification, screen description, locating, memory extraction, route matching, skill drafts, app-knowledge enrichment, outcome judging after an answer is handed over | each role's own request | each role |

App, site and skill knowledge is not here — see `knowledge/` and `skills/`.

## Where to put things

- The higher up, the more general and neutral: `system/` names no concrete tool, app or
  technology; a rule that holds only for one capability goes in that capability's section.
- Anything bound to a tool (how to use it, parameter meanings, result handling) goes in
  `tools/<tool>.md`, never in `system/`.
- A rule that only holds when a capability is present goes in `capabilities/<capability>.md`,
  behind the same switch as that group of tools. When a line in a section needs a tool from
  another capability, start the line with `<!-- needs: toolName -->`: the line is kept only
  if that tool is offered.
- `system/` names no tool (enforced by tests; words like `tap`/`back` that are ordinary
  English don't count).
- Text in tool results goes in `feedback/tools.md`; text the main loop emits goes in
  `feedback/loop.md`. Keys are unique across the two files, and every key must be used by code.
- Each feedback entry stands alone: no leading/trailing whitespace, and code never
  concatenates it with other text (`+` or string templates); to combine, put the separator in
  the template and inject the other entry with `{{placeholder}}`.
- Within a task the system prompt changes only when a tool group is expanded (so each group
  invalidates the cache at most once); things that change every turn (step count, time, which
  app) do not belong in this fixed text.

## Format

- `tools/`, `system/`, `capabilities/`, `roles/`: one file is one block of source text;
  leading/trailing whitespace is stripped on read.
- `feedback/*.md`: may begin with a `<!-- comment -->`; each entry starts with a line
  `## key` (keys use only lowercase letters, digits, `_` and `.`), and the text runs to the
  next entry, possibly multi-line; leading/trailing blank lines don't count (so separator
  newlines must go inside the template).

## Size

The byte size (UTF-8) of each layer is recorded in the golden file
`core/src/test/resources/prompt-surface/sizes.txt`; it is for reference only, no hard limit:
growth on every change shows up in the commit diff. Before adding content, see whether
something old can be removed; when it grows noticeably, explain why in the commit.

## After changing something

There are golden files for everything the model sees:
`core/src/test/resources/prompt-surface/` (system prompt, tool definitions, feedback, roles)
and `loop-snapshots/` (message sequences replayed by a fake model). Regenerate for
intentional changes with `./gradlew :core:test -PupdateSnapshots` and review them in the
same commit; when a change affects model behavior, run the evals to compare first. The
`prompt_version` in run records is a digest of this task's system prompt plus tool
definitions; use it to tell prompt versions apart when comparing eval results.

## Tool names

- Phone and web each have a set of screen tools (`phone_*`, `web_*`, same verbs and
  parameters); the same verb shares one `tools/<verb>.md` template, whose `{{verb}}` is
  replaced with that set's tool name; when one screen needs its own text, write
  `tools/<screen>_<verb>.md`.
- `capabilities/general.md` names no capability's tools (enforced by tests); "which channel
  to use" goes in `channels.md`, assembled only when there are two or more screens.
- Tool names mentioned in knowledge, skills and prompts must be currently registered
  (enforced by tests). Old names from before a rename are mapped to the new ones by
  `LegacyNames`; this is only used to read old things (saved routes, run records, personal
  skills and how-tos), and when the model calls an old name it is told the new name rather
  than having it executed on its behalf.

---
*English translation of `prompts/README.md` (the original is in Simplified Chinese).*
