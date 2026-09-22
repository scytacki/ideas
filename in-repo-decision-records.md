# In-Repo Decision Records for AI-Assisted Development

Keep a register of design decisions in the repository, so that both developers and AI
assistants can discover them. The developer's job then includes reviewing each feature
request against that register, and adding a record whenever new work settles a decision.

## Motivation

When an AI writes most of the code, the developer's role shifts toward the product owner
and architect roles: checking how a requested change fits the existing product and the
existing code, and proposing alternatives when it doesn't fit.

Doing that requires knowing what the existing design decisions *are*. Today the usual
answer is "an expert developer reads the code and infers the patterns." That works, but:

- It is imperfect even for experts, and much worse for someone new.
- An AI infers patterns from whatever files it happened to read, so it will cheerfully
  violate a decision that is expressed three directories away.
- The rationale that would settle the question often exists only in a PR review thread or
  a closed ticket, where nobody will find it again.

A decision that isn't written down can't be reviewed against.

## Approach: a decision register in the repo

One directory, `docs/decisions/`, one file per decision, using the MADR template
(context, decision drivers, considered options, decision, consequences).

Conventions that make it work:

- **The number is the order of *recording*, not of *deciding*.** Numbers get cited from
  code comments and other records, so they are never reassigned. A decision made two
  years ago and written up today gets the next free number.
- **A `date:` field carries the chronology.** Any timeline view sorts by that, not by
  filename.
- **Status lifecycle:** `proposed`, `accepted`, `superseded`, plus `needs-backfill` for
  a known rule whose rationale hasn't been reconstructed yet. The one edit allowed on an
  accepted record is adding `superseded-by:`; the replacement carries `supersedes:`.
- **A `type:` field** — architecture, product, ux, convention — so one register can hold
  all of them and still be filterable.
- **The title states the rule, not the topic.** "Documents are described by axes, not
  types" is findable by search; "Document axes" is not. This matters more than any other
  convention here, because search is how an AI actually finds these.

## Records versus reference documentation

These are different genres and shouldn't be merged:

- A **decision record** is immutable and dated. It answers "why is it this way, and what
  did we give up?" You supersede it rather than editing it.
- A **reference document** (a domain model description, a schema, a subsystem overview)
  is rewritten continuously. It answers "what is true right now?"

They should link to each other: the reference points at the record for rationale, the
record points at the reference for the current state. The invariant itself is best
enforced in types and tests, so that violating it fails rather than merely contradicting
a document.

## Records versus design specs

This workflow already produces design specs — one per feature, at
`docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` — and several of them already carry an
explicit `## Decisions` section. So decisions are being written down today, just not in a
register.

A spec is not a substitute for a record, on three axes:

- **Scope.** A spec covers one ticket. A record covers a rule that applies across tickets.
- **Lifetime.** A spec describes a plan and goes stale once the code lands; nobody rewrites
  it afterward. A record is meant to stay true, or to be explicitly superseded.
- **Title.** A spec is titled by ticket or feature ("CLUE-639: Author-set fixed start view"),
  not by rule. Search won't surface the decision buried inside it.

A spec also carries far more than its decisions: current architecture, file-level design,
edge cases, testing, critical files. Most of that isn't worth recording, and some of it is
wrong within a month.

What's unresolved is what to do about the overlap. The plausible answers:

- **Extract.** The durable decisions graduate into thin, rule-titled records that cite the
  spec — the same relationship a backfilled record has to a commit or a PR.
- **Index.** The spec stays the citable artifact, and the register is only a pointer layer: a
  list of rules, each linking to the spec section that settled it.
- **Wait.** Decisions live in specs until something proves that wasn't enough — a later spec
  contradicts one, or an AI violates one — and only then does a record get written.

### Settle this by experiment, not by argument

This is the same question as "do AI assistants actually respect a register they can read,"
seen from the other side. The goal is the *minimal* workflow and set of documents that
delivers the value, not the most complete one.

The cost is concrete and lands on every feature: requiring a decision record alongside each
design spec is more work for the developer, in the same PR, every time. That is only worth
paying if the record demonstrably does something the spec doesn't — gets found, gets
respected, prevents a violation. Until that's measured, adding the step is speculative
process.

## Backfilling past decisions

Most of the value at the start is in writing up the *right* decisions already made — which
makes picking them, and trusting the result, the whole problem.

### Where the evidence lives

Rationale that never became a record was usually written down somewhere else:

- **Git commit messages**, which often carry the reasoning for a constraint in the same
  change that introduced it, and which are searchable with `git log --grep` and `git log -S`.
- **GitHub** pull request review threads and issues.
- **Jira** tickets and their comment threads.
- **Slack**, where the argument frequently happened before any ticket existed.
- **Pivotal Tracker**, for decisions predating the Jira migration, if a backup or export of
  the old stories can be recovered.

All of these are searchable, and an AI can draft a record from them. That is the cheap half
of the work.

### Two rules keep backfilled records honest

- **Mark them as reconstruction** (`retrospective: true`). They are a best reading of why
  the code looks this way, not minutes of a meeting.
- **Cite the evidence** — the commit, the PR, the ticket, the Slack thread. Often the single
  most valuable line in the record is the commit that introduced the constraint.

### Expert review is the bottleneck

A drafted record has to be confirmed by a developer who was there. An AI assembling a
rationale out of an archive produces something plausible, and plausible is not the same as
correct — the stated reason may be a reasonable-sounding reconstruction rather than the one
that actually drove the decision. Drafting scales; that review does not.

### So only backfill decisions with evidence of violation

Write up a decision when something shows it wasn't understood:

- **Violated and corrected** — pushback in a review thread, a revert, a bug filed against
  the change.
- **Violated and ignored** — the code has drifted and nobody caught it.

A decision nobody has ever violated is probably obvious enough from the code that it doesn't
earn scarce review time. This is the same friction-driven trigger as the **Wait** option
under design specs above, applied to history instead of to new work.

The principle behind it: **a wrong record is worse than a missing one.** A missing record
leaves the status quo — infer the pattern from the code. A wrong one gets cited from code
comments, enforced by a check, and repeated back by an AI with far more confidence than an
inferred pattern would ever carry.

### Finding them: violation-first and rule-first

Searching for violations directly only finds the ones that left a trace. There is a second
direction that reaches the silent ones:

- **Violation-first.** Start from a correction — a revert, pushback in a review, a bug filed
  against the change — and write up the rule it implies.
- **Rule-first.** Start from a *statement* of how things should be, in a Slack message, a PR
  comment, or a commit message, then check the code for whether it actually holds. This is
  what turns "violated and ignored" from an undetectable state into an audit: once the
  candidate rule is written down, there is something concrete to test the codebase against.

The rule-first pass is also where the executable-check idea pays for itself twice. The check
written to audit the code for compliance is the same check that enforces the rule going
forward — see *Make decisions executable where possible* below.

The highest-value thing to search for is **a reviewer accepting a known violation** — "fine
to leave it this way for now." A single comment like that carries all three parts at once:
the rule exists, this code breaks it, and the breach was knowingly tolerated. Variants worth
grepping the archive for: "not worth changing now", "we should really", "ideally this
would", and `TODO`/`HACK` comments left in the code itself.

### Many of these aren't decisions

A rule recovered this way was often never decided. Nobody put alternatives on the table and
picked one — someone stated in passing how things obviously are, nobody disagreed, and it
became a principle, a convention, or an invariant without ever being a decision.

That changes the shape of the record, not just its name. MADR's "considered options" and
"decision drivers" sections come out empty for a rule nobody argued about, and filling them
in anyway invents a deliberation that never happened, which is exactly the fabrication the
expert review is there to catch. The honest form is thinner: the rule, why it holds, and how
it's enforced. A Y-statement is close to the right size.

So the register holds two genres:

- **A choice made under tension** — real alternatives, real trade-offs, something given up.
  The full MADR shape fits, and the consequences section is the valuable part.
- **A principle held as obvious** — nothing was weighed. Rule, rationale, enforcement. The
  valuable part is the rule statement itself, because that is what a person or an AI will
  match against.

"Decision record" stays as the umbrella term. It is the name the prior art uses, and MADR
already stretched it once — from *architectural* to *any* — for the same reason.

## Discovery

The numbering does very little work. These do:

- An index at `docs/decisions/README.md`, grouped by area rather than by number.
- A pointer from the file the AI reads by default (`CLAUDE.md` / `AGENTS.md`).
- Links from the code itself, at the place where the decision is most likely to be
  violated: `// see ADR-0007`.
- Links from the related reference docs.

## Make decisions executable where possible

Prose is only read when someone goes looking. A decision enforced by a check is
rediscovered automatically, by a person or an AI, at the moment it's broken. This is the
"fitness function" idea from *Building Evolutionary Architectures*. In a JS/TS repo that
means custom ESLint rules, import-boundary rules, `dependency-cruiser`, types that make
the wrong state unrepresentable, and tests.

The failure message should name the record, so the check and the rationale stay connected.

## Workflow

1. A feature request arrives. The developer reviews it against the register.
2. If it conflicts with a decision, decide which gives way: the decision or the request.
   Prototyping both the requested approach and the alternative is a good way to make that
   case concrete.
3. If the work settles a new decision, the record lands in the same PR as the code.

## Prior art

- **ADR** (Michael Nygard, 2011): title, status, context, decision, consequences.
- **MADR** — "Markdown Any Decision Record"; the rename from "architectural" was
  deliberate, to cover product and smaller decisions.
- **Y-statements** (Zimmermann): a one-sentence decision record.
- **arc42 section 9** ("Design Decisions") and **ISO/IEC/IEEE 42010**, which makes
  decisions and rationale a required part of an architecture description.
- **Tooling:** `adr-tools`, `log4brains` (browsable site with a timeline), Backstage
  TechDocs.
- **Product side:** Confluence's decision register template and macro; Shape Up pitches,
  whose "rabbit holes" and "no-gos" sections record what was deliberately excluded.

## Open questions

- How much do AI assistants actually respect a register they can read, versus needing the
  decision enforced by a check? This is worth measuring before investing heavily in prose.
- What is the minimal set of documents that delivers the value? Design specs already exist
  and already contain decisions, so a separate record only earns its per-feature cost if it
  gets found and respected where the spec doesn't. Run that experiment before adopting the
  step.
- Which decisions in a spec are even candidates for a record? The working filter: those that
  outlive the ticket, and that future work could plausibly violate. Everything else stays in
  the spec.
- Who writes the product and UX records if the people making those decisions won't touch
  the repo? Does the register need to sync with Confluence, or does a developer transcribe
  them?
- Is a sequential number the right identifier, versus a date-prefixed slug or the ticket
  key? Numbers are the easiest to cite from code, which is the main argument for them.
- How do stale records get caught? A periodic audit, or a CI check that every record cited
  from code still exists and isn't superseded?
- Does the register get too large to be useful? Per-area indexes might be needed, or a
  convention that only decisions someone could plausibly violate get recorded.
- What's the minimum tooling worth building: index generation, front-matter validation,
  supersede-link checking?
- Could the register be generated, at least in draft, from the existing archive — commit
  messages, PR threads, Jira, Slack, an old Pivotal export? That's where much of this
  rationale is already written down. The real question is whether an AI draft plus expert
  review is cheaper than the expert writing the record directly.
- How well does the rule-first pass actually work in practice? It depends on principles
  having been stated in writing somewhere, and on the compliance audit being cheap enough to
  run. A rule that was never articulated at all stays invisible to both directions of search.
