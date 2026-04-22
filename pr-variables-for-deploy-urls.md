# PR Description Variables for Deploy URLs

Let developers declare variables in a PR description and have them substituted into the
deployment URL produced by our `s3-deploy-action`. The first use case is letting a CLUE
developer specify the "unit" for the PR so the deploy URL opens that unit directly.

## Motivation

Our `s3-deploy-action` already supports a custom URL template where the deployed path of
the uploaded files gets substituted in. That's useful, but the URL is fixed per-repo — you
can't tailor it to the specific PR.

For CLUE, the natural thing a reviewer wants is a link that opens the build *with a
specific unit loaded*. Today that means either hand-editing the URL after the fact or
baking one unit into the repo default. Neither scales: different PRs touch different
units, and a reviewer shouldn't have to know the query-string incantation.

More generally, there are lots of per-PR knobs a developer might want to pin into the
deploy link: a feature flag, a problem set, a locale, an example document ID. All of these
are things the PR author knows and the reviewer shouldn't have to.

## Approach

Extend `s3-deploy-action` (or wrap it) so that, before URL substitution, it parses the PR
description for developer-defined variables and makes them available as substitution keys
alongside the existing deployed-path variable.

### Defining variables in the PR description

The variables need to be identifiable unambiguously so random prose in the description
doesn't accidentally get picked up. A few options:

- **Fenced code block with a label** — e.g. a ` ```deploy-vars ` block containing
  `key=value` lines. Pros: unambiguous, already rendered as a code block on GitHub, easy
  to parse. Cons: slightly more ceremony than plain text.
- **Section delimited by `---` / `<hr>`** — treat content between horizontal rules as the
  variables block. Pros: renders as a visual separator, minimal syntax. Cons: `---` is
  also YAML front-matter and is commonly used elsewhere in descriptions; risk of
  collision.
- **HTML comment block** — e.g. `<!-- deploy-vars: unit=foo -->`. Pros: invisible in the
  rendered description, no visual clutter. Cons: hidden from reviewers, which might
  actually be a downside — the point is that the reviewer can *see* what the link is
  going to do.

A fenced code block with a label feels like the best tradeoff: explicit, visible,
parseable, and GitHub already highlights it.

### Substitution

Once parsed, variables become additional keys the URL template can reference, for example:

```
https://example.org/clue/index.html?unit={{unit}}&deploy={{path}}
```

Missing variables should be a clear action error ("PR description is missing required
variable: `unit`") rather than silently producing a broken URL. Required vs. optional
variables could be declared in the action inputs.

## Open questions

- Where does the schema for allowed/required variables live? In the action inputs
  (per-repo config) seems right — the repo knows what its URL template needs.
- Should the action also post the resolved URL back as a PR comment, or is editing the
  existing deploy-status comment enough?
- What about values that need escaping (spaces, slashes, unicode)? The action should
  URL-encode values before substitution, but documenting that clearly matters.
- Does it make sense to support multiple deploy URLs per PR (e.g. "open unit A" and "open
  unit B") driven off a list variable? Probably yes eventually, but the simple scalar case
  is the right first step.
- Could the same variables feed other parts of the CI pipeline (e.g. which Cypress tests
  to run against the deploy)? Worth keeping in mind when designing the parser, so the
  parsed variables are easy to consume from other workflow steps.
