# Automated PR Review and Test Pipeline

A command (triggered by a PR comment) that runs tests, handles AI code review, and requests
human review — all automatically.

## Basic version

Add a comment to a PR (or use a label/slash command) that triggers:

1. Run the test suite against the PR.
2. If tests pass, automatically request a review from a specified developer.

This removes the manual step of waiting for CI, checking results, and then remembering to
request a review.

## Advanced version

A more sophisticated pipeline:

1. **Request a Copilot review** on the PR.
2. **Have Claude review the Copilot review.** For each comment:
   - If the comment isn't worth fixing (stylistic nits, false positives, etc.), resolve it
     without making changes.
   - If the comment is substantive, leave it open (or fix it).
3. **Add a `run regression` label** to trigger the full test suite.
4. **Once tests pass**, automatically request a review from the specified developer.

## Trigger mechanism

- A PR comment like `/review @developer-name` could kick off the pipeline.
- Could also work via a label or a GitHub Actions workflow dispatch.

## Open questions

- How to authenticate and orchestrate the Copilot review request + Claude's review of it?
  Likely needs a GitHub App or Action with appropriate permissions.
- Should Claude's review of Copilot comments be transparent (e.g., leave a reply explaining
  why a comment was resolved) or silent?
- How to handle the case where Claude decides a Copilot comment *is* worth fixing — should
  it auto-fix and push, or just leave the comment open for the author?
- What's the right threshold for "not worth fixing"? This probably needs to be configurable
  or learned over time.
