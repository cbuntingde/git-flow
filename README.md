# git-flow

![License](https://img.shields.io/badge/license-Apache_2.0-blue.svg)
![Version](https://img.shields.io/badge/version-0.4.0-blue)
![Platforms](https://img.shields.io/badge/platforms-Modelize%20%7C%20Claude%20Code%20%7C%20Cursor%20%7C%20Copilot-7e57c2)
![GitHub](https://img.shields.io/badge/github-cbuntingde%2Fgit--flow-181717?logo=github)

A plugin that turns every code change into a clean, reviewable update on
GitHub. Ask your assistant for a change in plain English — it creates an
isolated line of work, opens a review request, waits for your automated
checks to pass, and lands the change for you.

Works on **Modelize** and **Claude Code** out of the box. Cursor and
GitHub Copilot (CLI and VS Code) load the same `plugin.json` layout.
OpenAI Codex CLI is **not** supported — Codex has no plugin format;
see "Codex" below for a manual workaround.

## What you get

- **Isolated changes by default.** Every request gets its own line of
  work (a branch), so unrelated changes never get tangled together.
- **Automatic review requests.** Your assistant opens a GitHub pull
  request with a description and pushes the change for review.
- **Hands-off landing.** The plugin waits for your automated checks (the
  tests and other checks GitHub runs on your behalf) to pass, then lands
  the change and switches your workspace back to the main line of work.
- **Local sanity check.** Even if your repository has no automated
  checks configured, the plugin detects your project's stack (Node,
  Python, Rust, Go, or Make) and runs the equivalent of CI on the
  working tree before pushing — so obvious failures don't reach the
  review request.
- **Optional CI scaffold.** If your repo has no `.github/workflows/`
  file at all, you can ask the plugin to propose a minimal one with
  `/git-flow:setup-ci`. It shows you the file before writing anything
  and waits for your approval.
- **Remote branches kept by default.** After a change lands, its branch
  stays on GitHub so you can revisit, audit, or point someone at the
  exact line of work it came from.

## Install

The directory contains both manifest formats Modelize and Claude Code
look for. Pick the editor you use; the rest of the layout is the same.

### Modelize

Modelize discovers this plugin automatically once the directory is in a
plugin root. Either:

- **User scope** (every project): copy or clone this directory into
  `~/.modelize/plugins/git-flow/` and turn it on in
  **Settings → Plugins**.
- **Project scope** (one repository): copy or clone this directory into
  `<repo>/.modelize/plugins/git-flow/` and turn it on in
  **Settings → Plugins** (a project-scope plugin is off by default).

After enabling, reload the plugin panel — there should be no errors and
the `git-flow` skill plus the `/git-flow:*` commands should appear in
the composer's `/` menu. Modelize also has a `session_start` hook
configured that primes the model to load the skill on session start.

### Claude Code

Claude Code looks for `.claude-plugin/plugin.json`. You need a Claude
Code-aware install path — the recognized locations are similar to
Modelize's. Pick the scope you want:

- **User scope** (every project): copy or clone this directory into
  `~/.claude/plugins/git-flow/`.
- **Project scope** (one repository): copy or clone this directory into
  `<repo>/.claude/plugins/git-flow/`.

Claude Code is on-by-default for user scope and off for project scope;
flip the switch in **Settings → Plugins**. Reload the plugin panel and
open the `/` menu — `git-flow` and the `/git-flow:*` commands should
appear. The same `hooks/hooks.json` registers a `SessionStart` entry
for Claude Code, mirroring the Modelize event so the skill loads on
session start there too.

### Cursor

Cursor reads the same `plugin.json` at the directory root, but its
plugin format is narrower than Modelize's and Claude Code's. Place the
directory inside Cursor's plugin path
(`~/.cursor/plugins/git-flow/` for user scope, or
`<repo>/.cursor/plugins/git-flow/` for project scope). Slash command
names and the `gh`/`git` workflow inside `SKILL.md` work identically;
verify the latest Cursor plugin docs for current command and hook
support before relying on the session-start hook.

### GitHub Copilot (CLI and VS Code)

Copilot reads `plugin.json` at the root via the Agent Plugins format.
Installation paths vary — follow your Copilot client's plugin
instructions and point it at this directory. Slash command syntax may
differ (Copilot often uses `/git-flow branch` without the namespace
colon); the underlying `gh` and `git` commands inside `SKILL.md` are
unchanged.

### Codex (not supported)

OpenAI Codex CLI does not have a plugin format. The skill body in
`skills/git-flow/SKILL.md` and the slash commands in `commands/`
cannot be auto-loaded. A manual workaround that gives you most of the
behavior:

```bash
cat skills/git-flow/SKILL.md >> ~/.codex/instructions.md
```

This appends the skill body to Codex's per-user instructions file. It
will not register the `/git-flow:*` slash commands, and the
session-start hook has no equivalent in Codex — but the workflow
itself (branch → commit → local check → push → PR → wait → merge) is
just shell commands and will fire when the model reads your prompt in
the context of those instructions.

## Requirements

- `git` available on your `PATH`.
- `gh` (the GitHub command-line tool) installed and signed in. Confirm
  with:

  ```bash
  gh auth status
  ```

- A git repository hosted on **github.com**. GitHub Enterprise Server,
  GitLab, Bitbucket, and self-hosted GHE instances are **not** supported
  in this release; `gh` auth must succeed against `github.com` for the
  workflow to run end-to-end.

## Quick start

1. Ask your assistant to make a change, for example: *"rename the login
   function to `authenticate`"*.
2. The assistant creates an isolated branch, commits the change, and
   opens a review request on GitHub.
3. The assistant waits for your automated checks to pass, lands the
   change, and returns your workspace to the main line of work.

That's it. You can keep working while the change is reviewed and merged
in the background.

## Commands

If you want finer control, the plugin exposes the following slash
commands. Most users won't need them — your assistant uses the full
workflow automatically when you ask for a change.

| Command | What it does |
|---|---|
| `/git-flow:branch <name>` | Create an isolated branch and stop. Use when you want to set up a line of work before describing the change. |
| `/git-flow:pr` | Push the current branch and open a review request. |
| `/git-flow:watch` | Wait for the automated checks to pass. |
| `/git-flow:merge` | Land the change and return your workspace to the main line of work. |
| `/git-flow:status` | Show the current branch, review-request URL, and check status. Read-only. |
| `/git-flow:back-to-main` | Abandon the current branch and return to the main line of work. |
| `/git-flow:setup-ci` | Propose a minimal `.github/workflows/ci.yml` matching your stack and open a review request for it. Opt-in only — never runs automatically. |

> Copilot clients may render the same commands as `/git-flow branch`,
> `/git-flow pr`, etc. (no namespace colon); the underlying
> command files are unchanged.

## Configuration

All settings are optional. The defaults work for most repositories.

| Variable | Default | What it does |
|---|---|---|
| `MODELIZE_GIT_FLOW_WATCH_TIMEOUT_MIN` | `30` | How long, in minutes, to wait for automated checks before giving up. |
| `MODELIZE_GIT_FLOW_MERGE_STRATEGY` | `squash` | How to land the change. `squash` collapses it into one commit; `rebase` keeps every commit; `merge` adds a merge commit. |
| `MODELIZE_GIT_FLOW_DELETE_REMOTE_BRANCH` | `0` | Set to `1` to delete the branch on GitHub after the change lands. Default `0` keeps it. |
| `MODELIZE_GIT_FLOW_BASE_BRANCH` | _(auto-detect)_ | Override which branch counts as the main line of work. |
| `MODELIZE_GIT_FLOW_SKIP_LOCAL_CHECK` | `0` | Set to `1` to skip the local sanity check that runs before pushing. The plugin still reads it on `/status`, `/watch`, and `/merge`. |
| `MODELIZE_GIT_FLOW_LOCAL_CHECK_TIMEOUT` | `300` | Per-check timeout, in seconds, for the local sanity check. |

The `MODELIZE_GIT_FLOW_*` prefix is set in stone for the 0.4.0 series —
renaming it would break any user who already exported the old names.
A future major version may introduce a `GIT_FLOW_*` short form.

## Limitations

- **One change at a time.** The plugin handles a single change end to
  end before starting the next. If you ask for several changes, they
  run one after another.
- **No background hooks beyond session start.** The plugin runs when
  you invoke it, when your assistant uses it on your behalf, and
  through the `session_start` (Modelize) and `SessionStart` (Claude
  Code) hooks that prime the model to load the `git-flow` skill. There
  are no other unattended triggers.

## License

Apache-2.0.