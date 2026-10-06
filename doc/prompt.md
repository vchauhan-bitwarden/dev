# Daily Dev Flow: Jira Ticket → Merged PR (Claude Code + Bitwarden Plugins)

Run these prompts in Claude Code, in order. Replace `PM-12345` with your ticket key.

## Main Flow

| #  | Step                             | Prompt to paste                                                                                          | Plugin / Skill used                                   |
| -- | -------------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| 1  | **Research**                     | `Deep dive PM-12345 — summarize scope, acceptance criteria, linked tickets and blockers. Save it to ~/Bitwarden/dev/doc/PM-12345.md.` | atlassian-tools · `researching-jira-issues`           |
| 2  | **Still valid?** (bugs/old tickets) | `Is PM-12345 still relevant? Verify against the current code.`                                        | atlassian-tools · `assessing-jira-issue-relevance`    |
| 3  | **Find the code**                | `Which files/services in this repo are involved in PM-12345?`                                            | built-in (Explore agent)                              |
| 4  | **Plan**                         | `Plan the implementation for PM-12345 — give options with trade-offs and recommend one. Save it to ~/Bitwarden/dev/doc/PM-12345-plan.md.` | delivery-tools · `architecting-solutions`             |
| 5  | **Test plan**                    | `For PM-12345, what tests should I add and at which layer?`                                              | testing-tools · `recommending-test-layers`            |
| 6  | **Branch**                       | `Create a branch for PM-12345 from latest main.`                                                         | built-in (git)                                        |
| 7  | **Implement**                    | `Implement PM-12345 using the plan above. Follow existing patterns and ask me if anything is unclear.`    | software-engineer agent                               |
| 8  | **Write tests**                  | `Write the tests from the test plan and run them.`                                                       | software-engineer agent                               |
| 9  | **Repo checks**                  | `Run lint:fix, prettier, test:types and the tests for the changed files. Fix any failures.`              | built-in                                              |
| 10 | **Security check**               | `Do a security review of my local changes.`                                                              | security-engineer · `perform-security-review`         |
| 11 | **Preflight**                    | `Run preflight on my changes.`                                                                           | delivery-tools · `perform-preflight`                  |
| 12 | **Commit**                       | `Commit my changes for PM-12345.`                                                                        | delivery-tools · `committing-changes`                 |
| 13 | **Self review**                  | `/bitwarden-code-review:code-review-local` → then `Fix the valid findings.`                              | code-review · `code-review-local`                     |
| 14 | **Raise PR**                     | `Open a draft PR for this branch.`                                                                       | delivery-tools · `creating-pull-request`              |
| 15 | **CI**                           | `Check CI for my PR and fix any failures.`                                                               | built-in (gh)                                         |
| 16 | **Review comments**              | `Help me address the review comments on PR #<number>.`                                                   | code-review · `addressing-code-review-comments`       |
| 17 | **Push fixes**                   | `Preflight, commit and push the review fixes.`                                                           | delivery-tools                                        |
| 18 | **Merge**                        | You do this yourself in GitHub after approval.                                                           | —                                                     |
| 19 | **Wrap-up doc**                  | `Write a wrap-up for PM-12345 to ~/Bitwarden/dev/doc/PM-12345-summary.md: the problem, root cause, the fix (each changed file and why), tests added and how I verified, PR link, review comments and how I addressed them, follow-ups, and what I learned. Use the ticket, the plan doc, git diff and the PR — don't guess.` | built-in (no plugin skill does this) |

## Optional Steps

| When                                  | Prompt                                                                              |
| ------------------------------------- | ----------------------------------------------------------------------------------- |
| A requirement is unclear mid-work     | `This part of PM-12345 isn't specified. List the options so I can ask the tech lead.` |
| You found a separate bug              | `File a Jira bug for <issue>, linked to PM-12345.`                                  |
| You added an npm package              | `Review the new dependency for supply-chain risk.`                                  |
| You're reviewing a teammate's PR      | `/bitwarden-code-review:code-review-local <PR number>`                              |
| You changed `.claude/` or CLAUDE.md   | `/claude-config-validator:validate-ai-local`                                        |
| End of the day / after a big task     | `Run a retrospective on this session.`                                              |

## Tips

- **Use one session per ticket.** Steps 1–4 stay in context, so step 7 can just say "the plan above."
- **Always review the plan (step 4) before step 7.** Fixing a bad plan is cheaper than fixing bad code.
- **Step 14 shows a preview before anything is pushed.** Check the title and body, then confirm.
- **`code-review-local` writes to local files; `code-review` posts to GitHub.** Use the local one for self-review.
- **Save docs as you go (steps 1, 4, 19).** If the session gets compacted or you come back the next day, the files still have the context, and step 19 can build on them.
- **Closed the terminal? Resume the session.** From the same folder you worked in (sessions are saved per folder), run `claude -c` for the last session or `claude -r` to pick one. Inside Claude Code, use `/resume`. Name sessions with `/rename PM-12345` so they're easy to find. If context was lost, say `Read ~/Bitwarden/dev/doc/PM-12345-plan.md and continue from there.`
- **Merging is always manual.** Don't ask Claude to merge, approve or close PRs.

## References

- Marketplace: https://github.com/bitwarden/ai-plugins
- Plugin READMEs: https://github.com/bitwarden/ai-plugins/tree/main/plugins
- Claude Code plugin docs: https://code.claude.com/docs/en/plugins.md
