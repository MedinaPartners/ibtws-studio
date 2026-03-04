# Smart Git Committer Prompt

## Purpose and Background

This prompt is designed to help AI assistants create meaningful, well-organized git commits. Instead of dumping all changes into a single "add" commit, this prompt guides the AI to analyze the changes, group them logically, and create separate commits with descriptive messages that explain the "why" not just the "what".

When working on a project, developers often make multiple unrelated changes in a single session. Committing everything together with a vague message like "add" or "update" makes the git history useless for understanding what happened. This prompt ensures that each commit represents a cohesive unit of work with a clear purpose.

## Communication Language

When communicating with the user, always use Traditional Chinese (繁體中文). Commit messages should be written in English, following standard conventions.

## Execution Flow

When the user asks you to help with git commits, follow this process.

### Step 1: Analyze the Current State

First, examine the current git status and changes. Run the following commands to understand what has been modified:

```bash
git status
git diff --stat
git diff
```

For untracked files that are new, also examine their contents to understand what they add.

### Step 2: Categorize the Changes

After reviewing all changes, categorize them into logical groups. Common grouping strategies include:

By feature or functionality: Group changes that work together to implement a single feature. For example, if you modified `slack.py` across multiple repos to add a `send_code_block` method, that's one logical unit.

By file or module: Sometimes changes to a single file or module should be committed separately, especially if they address different concerns.

By type of change: Separate bug fixes from new features from refactoring from documentation updates.

Present your analysis to the user in a clear format, showing which files belong to which commit group and explaining why you grouped them that way. For example:

```
我分析了這次的改動，建議分成以下幾個 commit：

Commit 1: [slack] Add send_code_block with auto-snippet for long messages
- pm/pkg/slack.py
- thetav2/pkg/slack.py
- beacon/pkg/slack.py
- brain/pkg/slack.py
- brain-v2/pkg/slack.py
- fmpv2/pkg/slack.py
- hrp_signal/pkg_slack/slack.py
原因：這些都是同一個功能的改動，跨多個 repo 加入長訊息自動轉 snippet 的功能

Commit 2: [trading_report] Add report date to snippet filename
- pm/flex/scripts/trading_report.py
原因：這是 trading_report 專屬的改動，讓 snippet 檔名包含報告日期
```

### Step 3: Get User Confirmation

Before making any commits, ask the user to confirm the grouping strategy. The user may want to adjust the grouping or combine/split certain commits. Only proceed after explicit confirmation.

### Step 4: Execute Commits

For each commit group, execute the following steps:

1. Stage only the files for that specific commit using `git add` with explicit file paths. Never use `git add .` or `git add -A` as this may accidentally include unintended files.

2. Create the commit with a descriptive message following this format:
   - First line: Short summary (50 chars or less), starting with a scope tag in brackets like `[slack]` or `[trading_report]`
   - Blank line
   - Body: Detailed explanation of what was changed and why (wrap at 72 chars)
   - Include `Co-Authored-By: Claude <noreply@anthropic.com>` at the end

Example commit message:
```
[slack] Add send_code_block with auto-snippet for long messages

Slack has a ~4000 character limit for code blocks. When exceeded,
the formatting is automatically removed. This change adds automatic
detection of message length and switches to files.upload API with
snippet_type="text" for messages >= 3500 characters.

Changes applied to all repos: pm, thetav2, beacon, brain, brain-v2,
fmpv2, hrp_signal.

Co-Authored-By: Claude <noreply@anthropic.com>
```

3. After each commit, run `git status` to verify the commit was successful and show remaining changes.

### Step 5: Push to Remote

After all commits are made, ask the user if they want to push to remote. If yes, run:

```bash
git push
```

If the branch doesn't have an upstream set, use:

```bash
git push -u origin <branch-name>
```

## Commit Message Guidelines

Good commit messages should:
- Start with a scope tag that identifies the affected area: `[slack]`, `[trading_report]`, `[config]`, etc.
- Use imperative mood: "Add feature" not "Added feature" or "Adds feature"
- Explain the motivation for the change, not just what changed
- Reference any relevant context (bug reports, feature requests, etc.)

Avoid:
- Vague messages like "fix bug", "update code", "add stuff"
- Messages that only describe what files changed without explaining why
- Overly long first lines (keep under 50 characters)

## Handling Multiple Repos

When the user asks for help with git commits, first scan ALL repos under the Medina directory to find which ones have uncommitted changes. Run `git status` in each repo to identify pending changes.

The complete list of repos to check:

```
/Users/chouwilliam/Medina/beacon
/Users/chouwilliam/Medina/brain
/Users/chouwilliam/Medina/brain-v2
/Users/chouwilliam/Medina/brainv3
/Users/chouwilliam/Medina/cortexv2
/Users/chouwilliam/Medina/dashboardv1
/Users/chouwilliam/Medina/fmpv2
/Users/chouwilliam/Medina/homepagev2
/Users/chouwilliam/Medina/hrp_signal
/Users/chouwilliam/Medina/ibfm
/Users/chouwilliam/Medina/modelsv1
/Users/chouwilliam/Medina/pm
/Users/chouwilliam/Medina/hub
/Users/chouwilliam/Medina/thetav2
/Users/chouwilliam/Medina/tejpipev2
```

For each repo, run:
```bash
cd /Users/chouwilliam/Medina/<repo> && git status --short
```

Then report to the user which repos have changes:

```
我掃描了所有 Medina repos，以下有未提交的變動：

1. pm - 2 files modified
2. thetav2 - 1 file modified
3. beacon - 1 file modified
...

以下 repos 沒有變動：brainv3, cortexv2, dashboardv1...

要我開始處理嗎？
```

Handle each repo separately:
1. Complete all commits and push for one repo
2. Move to the next repo and repeat
3. Keep track of which repos have been processed

## Safety Rules

Never force push unless explicitly requested by the user.

Never commit files that may contain secrets like `.env`, `credentials.json`, or config files with API keys. If such files appear in the changes, warn the user before proceeding.

Always show the user what will be committed before actually committing.

If a commit fails (e.g., due to pre-commit hooks), fix the issue and create a new commit rather than amending.

## After Completion

Once all commits are pushed, provide a summary showing:
- Number of commits created
- Brief description of each commit
- Which repos were updated
- The git log showing the new commits

This helps the user verify that everything was committed correctly and provides a record of what was accomplished.
