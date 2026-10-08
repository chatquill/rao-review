# Rao Review

A Claude plugin that reviews a branch, diff or pull request the way Rao does: line by line, with a handful of small, concrete asks. Most findings are a replacement snippet or "remove these lines".

It is tuned for an Ember and TypeScript front end with some Rails. It reviews code shape, not production risk.

## What it does

- Reads the PR title, description and linked ticket, then asks you what the change is for if that is still unclear.
- Searches the repo before it writes a finding: unused additions, code left behind after a move, missed renames, helpers that already exist, CSS classes with no rule.
- Asks to delete or simplify code, move logic to the object that owns it, fix names, use Bootstrap utilities and Ember idioms (tasks, `.gts`, modifiers, zod).
- Names the exact missing test case instead of "add a test".
- Raises a bug only when it can state the input and the wrong result.
- Skips formatting, copy wording, speculation and production-risk remarks.
- Does not post to GitHub unless you ask.

### Example output

```text
ember/app/components/form/note.gts:18 [simplify] forwarding getter, read @answer.hasNote
  suggestion: remove these lines
ember/app/components/form/note.gts:42 [naming] task should end in Task
  suggestion: saveNoteTask
ember/tests/acceptance/note-test.gts:10 [tests] cleared default not tested
  suggestion: add a test that clears the default and asserts the request body

3 findings; the missing test should block the merge.
```

## Install

### Option 1: Claude Code plugin from GitHub

Run these two commands inside Claude Code:

```text
/plugin marketplace add chatquill/rao-review
/plugin install rao-review@rao-review
```

Then run `/reload-plugins` or start a new session.

### Option 2: Copy the skill folder into Claude Code

```bash
git clone https://github.com/chatquill/rao-review.git
mkdir -p ~/.claude/skills
cp -r rao-review/skills/rao-review ~/.claude/skills/
```

For one project only, copy it into that project's `.claude/skills/` instead and commit it so your team gets it too. Start a new Claude Code session afterwards.

### Option 3: Upload the skill file to Claude.ai

1. Download [`rao-review.skill`](https://github.com/chatquill/rao-review/releases/latest/download/rao-review.skill) from the latest release. It's a zip file that contains the skill folder.
2. In Claude, open **Settings**, find **Skills** (under **Capabilities** on most accounts) and choose **Upload skill**.
3. Make sure the skill is switched on. Skills need code execution to be turned on.

## How to use it

In Claude Code, from the repository you want reviewed:

```text
/rao-review              review the current branch against origin/master
/rao-review 1234         review pull request #1234
/rao-review ember/app/components/form
                         review only that path
```

If you installed it as a plugin (Option 1), the full command is `/rao-review:rao-review`; typing `/rao-review` and picking it from the list works too.

Reviewing a PR number needs the [`gh` CLI](https://cli.github.com/) signed in to your account.

## Updating

- **GitHub plugin:** run `/plugin marketplace update rao-review` in Claude Code.
- **Copied folder:** `git pull`, then copy `skills/rao-review` again.
- **Uploaded skill:** delete the old skill in Settings and upload `rao-review.skill` from the latest release.

## Files

```text
.claude-plugin/
├── plugin.json              Plugin manifest
└── marketplace.json         Lets this repo work as its own marketplace
skills/rao-review/
└── SKILL.md                 The instructions Claude follows
```

To build the `.skill` file yourself:

```bash
cd skills
zip -r ../rao-review.skill rao-review
```

## Releasing

Raise `version` in `.claude-plugin/plugin.json` (and in `marketplace.json`) and push to `master`. The release workflow builds `rao-review.skill` and publishes release `v<version>` with it attached.

## Feedback

Found a finding it should not have raised, or a review comment it missed? [Open an issue](https://github.com/chatquill/rao-review/issues) with the diff and what you expected.

## Privacy

The plugin has no code and sends no data anywhere. See [PRIVACY.md](PRIVACY.md).

## License

[MIT](LICENSE)
