# Control Which Jira Issues a PR Links To

The GitHub-for-Jira integration links a PR to every Jira issue key it finds, in the PR's
description as well as its title, branch and commit messages. We want to name related stories
in PR descriptions, so reviewers know about follow-up work, without the PR showing up on each
of those stories' development panels.

## What we saw

In neural-pathways, PR #41 (NPW-38) named NPW-49 and NPW-62 in its description as places
deferred work was filed. The integration added its footer links for both, so the PR is
attached to them. A commit message naming a new chore did the same. #37's description named
several follow-up stories, but only one of them got a footer, so what it picks up isn't
entirely predictable.

For now we accept the extra links: naming the stories in descriptions is worth more, and no
automation moves an issue when its linked PR merges.

## Things to look into

- Whether the integration has a setting to scan only the title and branch, or to ignore the
  description.
- Whether a key can be written so people still see it but the integration skips it (in
  backticks, in a link with other text, with a zero-width character).
- Why some keys in #37's description were linked and others weren't.
- Replacing it with our own integration, such as a GitHub Action that links a PR only to the
  issue in its title or branch, and posts nothing for keys mentioned elsewhere.
