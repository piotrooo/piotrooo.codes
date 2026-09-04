---
title: 'Reviewing AI-Generated Code with herdr-reviewr'
date: 2026-09-03
draft: true
url: 'reviewing-ai-generated-code-with-herdr-reviewr'
description: Review an AI coding agent's diff, attach feedback to exact lines, and send it back without leaving Herdr.
tldr: |
  AI coding agents can write changes quickly, but a human still needs to understand and review them. herdr-reviewr adds
  a focused review pane to Herdr: inspect the diff, comment on a line or range, and place precise feedback back into the
  coding agent's input. It makes the review loop more convenient. It does not replace engineering judgment.
tags: [ 'ai', 'code review', 'developer tools', 'herdr' ]
---

AI coding agents have changed the speed of implementation. A task that once required an hour of typing can now produce
a working patch in a few minutes. That is useful, but it does not remove the need to understand the code.

In fact, it creates a new bottleneck.

The agent finishes its work, and now I need to inspect every changed file, understand the decisions it made, find the
details that require another pass, and explain my feedback with enough context for the agent to act on it.

I can use `git diff`, open the files in an editor, copy a code fragment, write a message, and paste everything into the
agent's chat. This works. It is also surprisingly awkward when I have more than one comment.

[herdr-reviewr](https://github.com/persiyanov/herdr-reviewr) makes this part of the workflow more convenient. It gives me
a terminal-based code review pane next to my coding agent. I can read the diff, attach comments to exact lines, and place
the complete feedback into the agent's input without rebuilding the context by hand.

## Video walkthrough

{{< youtube LNp4nmGEQbk >}}

## The bottleneck moved from writing to reviewing

Generating a patch and accepting a patch are two different activities.

The coding agent may be able to change ten files in one turn, but I am still responsible for questions such as:

- Does the implementation satisfy the actual requirement?
- Is the solution consistent with the architecture and conventions of the project?
- Are the tests meaningful, or do they only make the build green?
- Did the change introduce unnecessary complexity?
- Are failure cases, security concerns, and operational consequences covered?
- Would I be comfortable maintaining this code six months from now?

This is not a problem that should be solved by asking another model for an automatic approval. An AI reviewer can help,
but it does not carry my responsibility for the final decision.

The important part is keeping the human review loop efficient. I want to remain close to the code, point at an exact
place, explain the problem, and return the feedback to the same agent with as little friction as possible.

## What is herdr-reviewr?

[Herdr](https://herdr.dev/) is a terminal runtime for working with coding agents. It owns their terminal panes and lets
you keep the agent, your shell, and supporting tools in one workspace.

`herdr-reviewr` is a Herdr plugin and a standalone terminal application. When used as a plugin, it opens as a review
pane beside the agent. The pane follows one Git worktree and provides three main tabs:

- **Changes** shows the changed files for the selected diff scope, including added and removed line counts.
- **All files** lets you browse the current content of the worktree, including files that are not part of the diff.
- **PR** displays the branch's pull request or merge request, its status, checks, description, and comments. This view is
  read-only.

The Changes tab supports four scopes:

- **uncommitted** compares staged, unstaged, and untracked work with `HEAD`;
- **branch** compares the worktree with the merge base of the selected base branch;
- **last turn** shows changes detected since the coding agent's most recently observed turn started;
- **commits** lets you select one commit or a consecutive range of commits.

This distinction matters in real work. Sometimes I want to review everything on a feature branch. Sometimes I only want
to see the patch produced by the latest agent turn. When the agent commits its work in small steps, the commits scope
lets me inspect those steps without mixing them with current uncommitted changes.

The plugin does not edit the worktree, stage files, create commits, or post anything to a Git hosting platform. Its role
is narrower: help me inspect code and prepare precise feedback.

## Installation

At the time of writing, the plugin requires:

- Herdr 0.7.5 or newer;
- Git available on `PATH`;
- a truecolor terminal with Unicode box-drawing support;
- macOS or Linux.

Install the plugin with one command:

```shell
herdr plugin install persiyanov/herdr-reviewr
```

Herdr downloads a prebuilt release, so a Rust toolchain is not required.

You can open the review pane in the current workspace with:

```shell
herdr plugin action invoke open --plugin persiyanov.reviewr
```

The binary can also run without Herdr:

```shell
herdr-reviewr ~/some/repo
```

Standalone mode is useful for inspecting a repository, but it cannot send comments to an agent and it does not provide
the last-turn scope. Both features depend on Herdr.

## Put the review behind one shortcut

The manual command is useful, but I do not want to type it every time an agent finishes a task. I mapped the plugin's
toggle action to `Command+R` in my Herdr configuration:

```toml
[[keys.command]]
key = "cmd+r"
type = "plugin_action"
command = "persiyanov.reviewr.toggle"
```

This belongs in Herdr's user configuration, normally located at:

```text
~/.config/herdr/config.toml
```

Now the same shortcut opens and closes the review pane. It is a small improvement, but it changes how the tool fits into
my normal loop. Review is no longer a separate activity that I need to remember at the end. The agent completes a turn,
I press `Command+R`, and the diff is available next to the conversation.

## A real review example

For a small demonstration, I started with this Spring Cloud Function bean:

<!--@formatter:off-->
```java
@Component
public class UppercaseFunction implements Function<String, String> {
    @Override
    public String apply(String message) {
        String payload = hasText(message) ? message : "default";
        return payload.toUpperCase();
    }
}
```
<!--@formatter:on-->

Calling `String.toUpperCase()` without an explicit locale relies on the JVM's default locale. That makes the result
dependent on the environment in which the application runs. Case conversion can behave differently for
locale-sensitive characters, so the implementation should make the intended behavior explicit.

I sent the following prompt to the coding agent:

```text
Make the case conversion in the Function bean locale-independent instead of
relying on the JVM default locale, and add a test that pins the behaviour for a
locale-sensitive character.
```

The agent changed the conversion to use an explicit locale and added the requested test. The relevant implementation
change was straightforward:

```diff
-        return payload.toUpperCase();
+        return payload.toUpperCase(Locale.ROOT);
```

The requested behavior looked correct, but code review is rarely limited to the exact sentence in the task. While
reading the surrounding code, the fallback value caught my attention:

```java
String payload = hasText(message) ? message : "default";
```

I wanted the string literal to have a name instead of remaining embedded in the method. This is a small comment, but it
is a good example of the kind of follow-up that can make an agent workflow unnecessarily cumbersome. The problem is not
writing the sentence. The problem is transferring the file, location, and relevant code back to the agent accurately.

## Attach feedback to the code

In the Changes tab, I select `UppercaseFunction.java` and move the cursor to the fallback line. The default keyboard
workflow is:

1. Press `v` to start a line selection.
2. Use `j` or `k` if the comment should cover a range.
3. Press `c` to open the comment box.
4. Write the feedback.
5. Press `Enter` to save it.

For this review, my complete comment is:

```text
Extract the "default" into a constant.
```

The comment is now attached to that exact place in the diff. I can press `l` to inspect the complete list of current
comments, `e` to edit a comment under the cursor, or `d` to delete it. The `n` and `N` keys move between comments.

Mouse input is supported as well. Clicking or dragging over the line-number gutter selects lines for a comment, while
dragging over regular text creates a normal terminal text selection for copying.

The keyboard shortcuts are convenient, but they are not something I need to memorize before using the tool. The footer
shows the next available action, and `?` opens a helper containing the keys that work in the current context.

## Send the complete context back to the agent

This is the most useful part of the workflow.

After checking my comments, I press `s`. If the workspace contains one coding agent, reviewr targets it directly. If
several agents are running, the plugin opens a picker and lets me choose the destination.

For every comment, reviewr prepares a block containing:

1. the file path and line or line range;
2. the selected diff snippet;
3. the comment text.

The blocks are sorted by file and starting line. When there are several comments, one blank line separates them. I do
not need to copy the file name, reproduce the changed line, or explain where the feedback belongs.

The plugin places this text into the selected agent's input and focuses that agent. It deliberately does **not** press
`Enter`. I can still read the prepared message, add broader context, correct a mistake, or decide not to submit it.

Only after checking the input do I press `Enter` myself.

This detail is important. The tool reduces mechanical work, but the final action remains explicit. The agent writes the
code, reviewr prepares the feedback, and I stay in control of what is actually sent.

After a successful send, the current set of comments is cleared. If sending fails, the comments remain available, so a
temporary problem does not destroy the review.

## More than a diff viewer

Line comments and sending feedback are the core workflow, but several supporting features make the pane useful during a
longer review:

- `j` and `k` move through files and lines;
- `[` and `]` jump between diff hunks;
- `f` and `F` jump between files;
- `/` searches file names and code across the worktree;
- `Ctrl+F` searches inside the open file;
- `w` toggles line wrapping;
- `m` switches a Markdown file between source and rendered preview;
- `e` opens the current file at the selected line in a configured editor;
- `y` copies the complete set of comments to the clipboard instead of sending it to an agent.

The navigator can be moved to any side of the pane, resized, or hidden. The interface and syntax highlighting share one
configurable theme, with multiple dark and light palettes available.

Plugin-specific settings live in a separate file:

```text
~/.config/herdr/plugins/config/persiyanov.reviewr/config.toml
```

For example:

```toml
theme = "tokyo-night"
default_scope = "branch"
navigator_position = "right"
toggle_placement = "overlay"
toggle_direction = "down"
auto_open = false
editor = "code -g {file}:{line}"
```

The plugin reloads this configuration on refresh and toggle. A missing file or setting uses the default value. If the
file is invalid, reviewr rejects the complete configuration, displays the error, and can recover after the file is
fixed.

## Pull request context without publishing anything

The PR tab can read the open pull request or merge request for the current branch from GitHub, GitLab, or Azure DevOps.
It uses the corresponding authenticated command-line tool: `gh`, `glab`, or `az` with the Azure DevOps extension.

The tab can display the request state, checks, description, and comments. It is helpful when I want repository context
beside the local diff, but it is intentionally read-only. reviewr never posts a review, resolves a thread, changes the
request, or approves it.

That boundary keeps the local feedback loop separate from the formal pull request workflow. I can iterate with the
coding agent first and publish or approve changes later using the normal platform process.

## Current limitations

The project is useful today, but its boundaries are worth understanding before relying on it:

- **Comments are in memory.** Closing the pane loses comments that have not been sent or copied.
- **Export is all-or-nothing.** Send and copy operate on the complete current set, not on one selected comment.
- **There is no line-number rebasing.** A comment remains locatable through its captured snippet. If the code changes,
  reviewr can mark the comment as stale instead of silently moving it.
- **Last-turn tracking uses polling.** The default interval is two seconds. A turn that starts and finishes inside one
  interval may be missed, and the result can include changes made manually during the same time.
- **Large files are limited.** Files over 2 MB or 50,000 lines show a notice instead of their content.
- **Binary files have no diff.** They are reported, but there are no lines on which to comment.
- **Windows is not supported.** The published plugin targets macOS and Linux.
- **The PR view is read-only.** It requires an authenticated and supported forge CLI. Each comment surface is limited to
  its newest 100 rows.

There is also an important functional boundary: reviewr does not determine whether a change is correct. It does not run
tests, perform an architectural review, approve a pull request, or replace a developer's judgment.

## Human review stays in the loop

I do not want an AI coding workflow that hides the code from me. I want one that removes repetitive coordination work
while keeping the important decisions visible.

That is where herdr-reviewr fits well. It does not try to become another coding agent. It gives the human reviewer a
focused surface for reading the result, recording exact observations, and returning them with the context the agent
needs.

The complete loop is simple:

1. The agent implements a task.
2. I open reviewr with `Command+R`.
3. I choose the relevant diff scope and inspect the changes.
4. I attach comments to specific lines or ranges.
5. I press `s` to place the complete review in the agent's input.
6. I check the message and press `Enter` myself.
7. I review the next patch.

The agent remains fast, but speed is not confused with correctness. The code still receives a deliberate human review,
and the feedback loop becomes much less awkward.

If you work with coding agents inside Herdr, you can find the project, documentation, and releases in the
[herdr-reviewr repository](https://github.com/persiyanov/herdr-reviewr).
