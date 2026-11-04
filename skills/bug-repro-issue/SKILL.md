---
name: bug-repro-issue
description: Use when the user describes a suspected bug (often one line, maybe a file:line) and wants it investigated, proven with a failing test, and written up as an issue to file. Not for fixing the bug.
---

# Bug Repro → Issue

The user gives a suspected bug. You:
1. Check whether it is real.
2. Prove it with a test that fails today.
3. Hand back an issue text for the user to file.

You do **not** fix the bug, and you do **not** commit unless asked.

## 1. Investigate

- Start at the cited location, or find it. Follow the data to the observable result: a return value, a response, a stored row, a decision. Cite each step as `file:line`.
- Explain it as a **generic flow**: any caller or input, not the names in the test fixtures. State the general preconditions, e.g. "the second writer can be any client".
- Give a verdict, and be honest about it:
  - **Valid.**
  - **Partly valid:** say which part.
  - **By design:** say so, cite the comment, doc or commit that shows the intent, and describe the trade-off instead of calling it a bug.
- Look for a worse variant than the one described. The real impact is often bigger than the reported symptom.
- Separate what is deterministic from what depends on something outside the code: timing, concurrency, an external service, an LLM, a mock. Say which one the bug rests on.

## 2. Reproduce with a test

- Put it in the existing test file for that code path, and reuse its fixtures, helpers and mocks. Match its style.
- Use real infrastructure when the bug needs it. Concurrency, transaction and ordering bugs need a real database and real separate transactions, not mocks.
- Go through public entry points, so that the real side effects happen: validation, rejections, defaults, auto-discovery.
- **Make the test fix-agnostic.** Assert the symptom ("must not return X", "must include Y"), not one fix's mechanism ("must return Z via field W").
  - Before adding an assertion that only some fixes would pass, tell the user which fix options it rules out and ask.
- Make the suspect path decide. If a mock or an override can hide the result, configure it so the code under suspicion makes the call.
- The test must **fail now, for the right reason**:
  - Read the failure message; don't just check that it failed.
  - If it passes unexpectedly, find out which other path it took. Adjust the setup, not the assertion.
  - To see internal state, add temporary prints or logs, back up the file first, and restore it afterwards. Confirm with `git diff`.
- The docstring or comment holds a one-line flow (`a → call → b → call`) plus why it breaks.
- Follow the project's own rules (CLAUDE.md, AGENTS.md, CONTRIBUTING, linter config).
- Run the project's linters and type checkers on the file. Report only new errors as yours; name the errors that were there before.

## 3. Issue text

Write it to a temporary or scratch file, `issue_<topic>.md`, and give its full absolute path. Use this structure, and keep it terse:

```
## Summary          2–4 sentences: trigger → wrong result → impact.
Location: file:line (+ where it takes effect)

## Why / Terms      only the concepts needed to follow the flow.
                    State that it is general, not specific to the test setup.

## Repro flow       table: Step | Event | relevant state. Mark key calls in bold.

## Proposed fix     or "Fix options (undecided)" when the design is the user's call.
                    Give each option's cost and what it breaks.
                    Rejected: options with the reason.

## Tests (currently failing)
                    test names, what each asserts, the exact command to run them.

## Acceptance       new tests pass, named existing tests still pass, linters clean.
```

## Discussion after the draft

- The user may edit the test to experiment. Predict the outcome from the code ("this should be empty because …"), and restore your version when asked.
- If the user says behaviour is intentional, keep the test proving the symptom. Reframe the issue around the trade-off, and list the options as undecided.
- Answer follow-up questions with evidence. If a claim was not verified, say so, or verify it with a quick probe.
- Update the draft only when the user agrees.

## Report to the user

Keep it short:
- the verdict and mechanism, with `file:line`;
- the test name and its actual failure line;
- the linter/type-check status;
- the draft path;
- what was not committed.
