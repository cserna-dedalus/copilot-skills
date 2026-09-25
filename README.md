# copilot-skills

Shared skills for [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/use-copilot-agents/use-copilot-cli).

Skills are Markdown "recipes" (`SKILL.md`) that teach Copilot CLI a repeatable, structured
procedure for a task. Copilot CLI automatically detects and offers a matching skill when
your request fits its description — you don't need to invoke it by an exact command name.

## Available skills

| Skill | Description |
| --- | --- |
| [`copilot-cli-usage-report`](.github/skills/copilot-cli-usage-report/SKILL.md) | Generates a per-model, monthly Copilot CLI consumption report (requests, tokens, cache tokens, and an unverified request-multiplier/AI-credit proxy), sourced from the local `session-store.db` telemetry on the machine where it runs. |

## Installation

Copilot CLI loads skills from two kinds of locations:

### Option A — Repository-scoped (recommended for teams)

1. Clone this repository, or add it as a Git submodule / copy the relevant folder into your
   own project under `.github/skills/`.
2. In Copilot CLI, run `/add-dir` and point it at the folder containing this repo (or your
   project root if you copied the skill in), so Copilot CLI trusts and loads its
   `.github/skills/**/*` content.
3. That's it — any skill under `.github/skills/<skill-name>/SKILL.md` becomes available in
   that session automatically. No restart needed.

If you just want to browse or manage what's loaded, use `/skills` inside Copilot CLI.

### Option B — Personal, available in every session/repo

Copy a skill folder into your personal Copilot CLI skills directory so it's available
everywhere, regardless of which repo you're in:

**macOS / Linux:**
```bash
mkdir -p ~/.copilot/skills/copilot-cli-usage-report
cp .github/skills/copilot-cli-usage-report/SKILL.md ~/.copilot/skills/copilot-cli-usage-report/SKILL.md
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\.copilot\skills\copilot-cli-usage-report" -Force
Copy-Item ".github\skills\copilot-cli-usage-report\SKILL.md" "$env:USERPROFILE\.copilot\skills\copilot-cli-usage-report\SKILL.md"
```

Restart Copilot CLI (or start a new session) and the skill will show up in
`/skills` and in the `<available_skills>` list Copilot CLI consults automatically.

## Using a skill

You don't need any special syntax. Just ask for what you want in natural language, e.g.:

> "Give me my Copilot CLI usage breakdown by model for this month"

Copilot CLI matches your request against each installed skill's `description` and loads the
matching one automatically. You can also mention a skill explicitly by name if you want to
force it.

## Adding a new skill

1. Create `.github/skills/<your-skill-name>/SKILL.md`.
2. Add YAML frontmatter with at least `name`, `description`, and `compatibility` (what
   tools/environment it needs). See the existing skill for the expected structure.
3. Write the procedure the agent should follow: inputs, steps, guardrails, and what to
   report back to the user.
4. Add a row to the table above and open a PR.

## Contributing

Keep skills:
- **Self-contained** — no secrets, no hardcoded paths specific to one person's machine.
- **Explicit about limitations** — if a skill relies on local-only data or an unofficial
  API/field, say so directly in the `SKILL.md`, not just in this README.
- **Read-only by default** unless the task genuinely requires making changes; call out any
  destructive/mutating step clearly in its own section.
