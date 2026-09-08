---
name: implement-jira-review-findings
description: >-
  Implement Jira issues produced by a branch review, using a separate
  implementation agent and an independent primary-agent review on the current
  feature branch. Use when the user asks to fix, address, or work through Jira
  review findings; not for creating tickets or implementing an unrelated issue.
---

# Implement Jira review findings

Resolve reviewed Jira findings on the user's current feature branch while
keeping implementation and acceptance review independent. The user retains
control of pushing and merging.

## Establish the work set

- Confirm the repository, current feature branch, base branch, and Jira issue
  keys. Do not switch branches merely because a ticket records an older branch.
- Read each issue from Jira, including acceptance criteria and the reviewed
  commit SHA. Load the repository's `AGENTS.md`, applicable rules, task history,
  and conventions before planning changes.
- Inspect the current code and revalidate every finding. A Jira issue is review
  evidence, not proof that its diagnosis or proposed remediation is still
  correct. Do not implement a stale, duplicate, already-fixed, or invalid
  finding. Explain the evidence and ask before changing that Jira issue.
- Order valid issues by risk and dependency. Prefer correctness and security
  prerequisites before performance or user-interface work, but adjust when one
  issue enables or substantially changes another.
- Report the proposed order and any overlap. Use one implementation stream when
  issues touch the same code or schema.

## Separate implementation from review

When agent delegation is available and the user has invoked this skill, assign
implementation to one non-root agent at a time. The primary agent remains the
reviewer and must not make concurrent edits to the same worktree.

Give the implementation agent one bounded Jira issue with:

- the issue key and current Jira description;
- repository, current branch, and base branch;
- applicable repository/task instructions;
- the requirement to re-check assumptions and report conflicts;
- the repository's required test/design workflow;
- authority to edit only what that issue reasonably requires;
- no authority to push, merge, or mutate Jira.

Reuse the same implementation agent for review corrections when practical.
Do not run overlapping issue implementations concurrently in a shared
worktree. Independent agents may be used sequentially when fresh context is
materially useful.

## Per-issue acceptance loop

For each valid issue:

1. Establish the pre-change state and isolate unrelated user changes.
2. Have the implementation agent follow the repository's test-first, task-file,
   migration, and design-review conventions. Do not weaken production behavior
   to make a test pass.
3. After implementation stops, independently inspect the actual diff and
   relevant surrounding code. Do not accept the agent's summary as evidence.
4. Check the Jira acceptance criteria, regression risk, authorization,
   concurrency and transaction boundaries, schema compatibility, performance,
   and consistency with comparable project code to the degree relevant.
5. Run or independently confirm proportionate focused tests. If review finds a
   defect, give the same agent concrete evidence and repeat implementation,
   review, and verification until accepted or genuinely blocked.
6. Ensure task/handoff records required by the repository are current.
7. Commit the accepted issue separately when repository conventions require
   commits or the user requested them. Include the Jira key in the commit
   subject. Do not include unrelated work.

Do not begin a dependent issue until its prerequisite is accepted. It is fine
to continue to the next independent issue after acceptance without asking for
another confirmation when the user authorized the whole displayed work set.

## Jira and repository boundaries

- Reading Jira issues does not authorize comments, edits, assignments,
  transitions, worklogs, or links. Perform those mutations only when the user
  explicitly requests them.
- Code-fix authorization does not authorize pushing, opening or merging a pull
  request, deleting branches, or rewriting history.
- Work on the current feature branch unless the user asks for another branch or
  worktree. Preserve unrelated changes already present there.
- If a ticket requires a materially larger product decision than its acceptance
  criteria establish, stop that issue and request direction rather than
  silently expanding scope.

## Final branch gate

After all accepted issues are implemented:

- run the full relevant test suite and any migration verification required by
  the repository;
- review the complete feature-branch diff against the base branch, including
  interactions between fixes;
- confirm every issue as fixed, invalid, already fixed, or blocked, with
  evidence;
- report commits, tests, remaining risks, and any Jira updates still requiring
  authorization;
- leave the branch unpushed and unmerged for the user unless they explicitly
  request otherwise.

Completion means the implementation and independent review both pass; a green
implementation-agent test run alone is not sufficient.
