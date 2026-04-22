# Markdown Review via GitHub PRs

A tool for collecting structured feedback (comments and suggested changes) on markdown files
stored in a GitHub repository. The primary use case is sharing documentation with reviewers
who may not use git, and giving them a way to leave line-level comments and propose edits.

## Motivation

GitHub doesn't have a built-in way to review files outside of a PR diff. You can't leave
inline comments on arbitrary files in a repo. PRs are the closest mechanism — they support
line-level comments, suggested changes, and threaded discussion — but only on lines that
appear in the diff.

## Approach: PRs as the Review Surface

Use a PR as the container for a review session. The PR is not intended to be merged — it
exists purely as a place to collect feedback.

### Comments (referencing specific lines)

The GitHub API (both REST and GraphQL) only allows line-level comments on lines within the
PR's diff context. The GitHub web UI can place comments on lines outside the diff via an
internal API, but this is not available through the public API.

**Workaround:** Use PR-level comments (or the PR description) that reference specific line
numbers, e.g., "Line 42: this term needs a definition." This avoids the diff limitation
entirely. A tool could format these with permalinks to the specific lines in the file.

**Alternative workaround:** Create the PR by deleting and re-adding the target files. This
puts every line in the diff, making all lines commentable via the API. Downside: the diff
view becomes a full-file addition rather than a clean set of changes.

### Suggested Changes

Reviewers who want to propose edits would have their changes committed to the PR branch.
The tool would create a branch, apply the suggested changes as commits, and open the PR.
The diff then shows exactly what the reviewer wants to change, and the author can review
the suggestions in the normal GitHub PR flow.

### Threading Discussion

GitHub PR comments support replies, but threading is limited:

- **PR-level comments** are a flat list. To build threads, the tool would need to use a
  convention (e.g., quoting the original comment) or parse reply chains.
- **File-level and line-level comments** in PR reviews do support reply threads natively.
  These are the best option for structured discussion, but require the lines to be in the
  diff (see workaround above).

A tool could combine both: file-level review comments for lines in the diff (with native
threading), and PR-level comments with a thread convention for everything else.

## GitHub API Findings

Tested in April 2026 against the GitHub REST and GraphQL APIs:

- **REST `POST /pulls/{id}/comments`**: Line-level comments require the line to be within
  the diff context. Returns 422 "line could not be resolved" otherwise. File-level comments
  (`subject_type: "file"`) work on any file in the changeset but aren't line-specific.
- **GraphQL `addPullRequestReviewThread`**: Same limitation. Returns null thread (silent
  failure) for lines outside the diff. Works for lines within the diff.
- **Legacy `position` parameter**: Maps to an offset within the diff hunk, not a file line
  number. Not useful for arbitrary line targeting.
- **GitHub web UI**: Can place comments on any line in any file in the changeset, even
  outside the diff context. Uses an internal API not available publicly.

## Open Questions

- Would GitHub Discussions be a better fit for some of this? They support threaded
  conversation but lack line-level file references.
- Could a GitHub App or Action automate the "delete and re-add files" trick to make all
  lines commentable?
- Is there a way to use the GitHub web UI's internal API programmatically (e.g., via
  browser automation)?
- What's the best UX for a non-technical reviewer? They probably shouldn't need to interact
  with git or the GitHub PR UI directly.
