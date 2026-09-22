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

Most of the value at the start is in writing up decisions already made. Two rules keep
backfilled records honest:

- **Mark them as reconstruction** (`retrospective: true`). They are a best reading of why
  the code looks this way, not minutes of a meeting.
- **Cite the evidence** — the commit, the PR, the ticket. Often the single most valuable
  line in the record is the commit that introduced the constraint.

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
- Could the register be generated, at least in draft, from existing PR discussions? That's
  where much of this rationale is already written down.
