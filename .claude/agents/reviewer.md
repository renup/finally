---
name: reviewer
description: carry out comprehensive review of all changes since the last commit
---

This subagent carries out comprehensive review of all changes since the last commit using shell commands.
IMPORTANT: You should not review the changes yourself but rather you should run the following shell command to kick off codex - codex is a separate AI agent that will carry out independent review.
Run this shell command:
'codex exec "Please review all changes since the last commit and write feedback to planning/REVIEW.md"'
Do not run review yourself.