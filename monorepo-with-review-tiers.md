# Split Packages for Review Tiers, Keep One Repo

Break a growing codebase into packages so review effort can scale with a change's blast
radius. Keep those packages in a single repository. The package split delivers nearly all
of the value; the repository split is a separate, smaller, and far more reversible
decision that should be made one package at a time, later, against explicit criteria.

The examples throughout come from CLUE, which is where the question came up concretely.
Nothing in the argument is CLUE-specific, though — it applies to any of our large
repositories, CODAP as much as CLUE. Read the CLUE references as illustrations, not as
scope.

## Motivation

As a codebase grows, every change starts to look equally expensive to review, because a
reviewer can't cheaply tell what a change can reach. A fix inside one tile and a change to
the document model both arrive as "a PR," and both get the same scrutiny — which means
either the tile change is over-reviewed or the model change is under-reviewed. Usually
both.

This matters more when an AI writes most of the code. The volume of changes goes up and
the cost of producing them goes down, so human review becomes the bottleneck. Spending
that scarce review capacity uniformly is the wrong allocation. We want to spend it where a
mistake propagates.

Splitting the codebase into packages makes blast radius legible: a package's dependents are
enumerable, so "what can this change break?" has an answer that doesn't require reading the
code. That is the core benefit, and it is entirely a property of the *package* split.

## Monorepo does not mean one PR per change

The argument for separate repositories usually arrives in this shape: *a change that spans
the framework and a leaf should be two PRs at two review levels, and a monorepo tempts us
to make it one.*

This conflates two things. A monorepo **permits** an atomic cross-package PR; it does not
require one. Landing a framework-layer PR and then a leaf PR stacked on top of it is
entirely possible in a monorepo — and it is *easier* there, because there is no
publish → version-bump → re-point cycle between the two.

Separate repositories don't grant the ability to split. They **mandate** it, including on
the changes where splitting makes review worse. A framework change reviewed with no visible
consumer is harder to review, not easier: the reviewer can't see whether the new interface
is actually usable, only that it compiles. Splitting is worth doing when both halves are
independently meaningful, and it's a cost when they aren't.

So on this axis the monorepo strictly dominates. It offers both options. Separate
repositories offer one.

What we actually want — review policy as a function of the set of paths a change touches —
is a tooling problem, not a topology problem. Separate repositories supply it for free at
the coarsest possible granularity, and that granularity is wrong in both directions: a
typo fix in the framework repo draws heavy review, while a leaf change that alters an API
other leaves consume draws light review.

## Two kinds of review

"Less review" needs to be said more precisely, or the rule will quietly erode something
that shouldn't be eroded. There are at least two distinct jobs happening in a review:

- **Blast-radius review** — does this break dependents? Is this interface right? Is this
  the correct layer for this logic? This genuinely scales with how many things depend on
  the code, and a leaf package really does need less of it.
- **Correctness review** — does this work? Is this the behavior we want? A bug in a leaf
  tile is still a bug a student hits. This does **not** get cheaper because a package has
  no dependents.

Only the first is what package tiering optimizes. A tiering scheme should say so out loud,
or "leaf packages need less review" will be read as "leaf packages need less care."

## Deriving the tier

The tier should be computed, not maintained by hand. A hand-written list of "framework"
and "leaf" packages goes stale the first time a dependency is added, and it goes stale
silently.

The inputs that matter:

- **Internal fan-in.** How many packages in the repo depend on this one, transitively.
  This is exactly what a dependency graph gives us, and CLUE already runs
  `dependency-cruiser`, so the graph is available today.
- **Publication.** A package published to npm is never a leaf, whatever its fan-in inside
  the repo. Consumers you can't see are the largest blast radius there is.
- **Whether the change touches the public surface.** An internal-only change to a
  high-fan-in package is less dangerous than a signature change to its exported API. This
  is the sharpest signal of the three and the hardest to compute; an approximation is
  whether the diff touches the package's entry points or its exported types.

The mechanism on top of that is ordinary GitHub machinery: `CODEOWNERS` scoped per package
directory, plus a CI check that computes the highest tier the diff touches and applies the
corresponding policy — required approvals, a label, or a bot comment that tells the
reviewer what depth of review is expected and why. Telling the reviewer *what kind* of
review this change needs is probably worth more than mechanically enforcing an approval
count.

Stacked PRs stay encouraged but not mandatory. When a cross-package change splits cleanly
into "extend the framework" and "use it in the tile," splitting it gets the framework half
proper scrutiny and lets the leaf half move fast. When it doesn't split cleanly, it lands
as one PR and is simply reviewed at the higher tier. Worth noting that an AI can do the
mechanical work of splitting a branch into a stack cheaply, which undercuts the argument
that separate repositories are needed to *force* a discipline that is otherwise too
tedious to follow.

## What the monorepo buys

- **Atomic cross-package change.** One commit, one CI run, one revert, and a history that
  stays bisectable. A change that spans a boundary can be verified as a whole before it
  lands, rather than verified in halves and hoped about in combination.
- **No version-bump dance.** A framework change consumed by five leaves is one PR, not six
  PRs and five dependency bumps.
- **No duplicate singletons.** CLUE leans on registries and shared singletons — the
  document registry, tile registration, MST shared models. If two packages resolve to
  different versions of the framework, the bundle ends up with two copies of a registry,
  and those bugs are miserable to diagnose. A monorepo makes that structurally impossible;
  separate repositories make it a standing hazard managed by peer-dependency discipline.
- **One tooling surface.** One base `tsconfig`, one ESLint config, one Jest setup, one CI
  definition. Every cross-cutting upgrade already survived — React 18, the Jest upgrade —
  is one PR here and N coordinated PRs across N repositories.
- **Global refactoring stays possible.** Renaming an exported symbol can find and fix
  every consumer in one pass.
- **Cheap boundary revision.** This is the strongest one, and it is specific to our
  situation: we are *discovering* the package decomposition, not implementing a known-good
  one. Some boundaries will be wrong. In a monorepo, moving a module to a different package
  is a `git mv` and an import fix. Across repositories it's a migration with history loss
  and a release cycle. Keep the boundaries cheap to revise until they stop moving.

## What separate repositories buy

Stated as strongly as they deserve, because these are real:

- **Boundaries are physical, not merely forbidden.** In a monorepo, nothing stops someone
  from deep-importing `../../other-package/src/internal`. This is preventable with an
  `exports` field, `dependency-cruiser` rules, and import-boundary lint rules, but that is
  configuration we have to write and maintain. Separate repositories make the violation
  impossible instead of detectable.
- **A published package forces an API contract.** A monorepo makes it easy to keep
  everything on whatever is currently in the tree and never discover that a contract broke.
  This is a real benefit when there are external consumers, and a pure cost when there is
  one consumer in the same tree.
- **Independent release cadence.** The genuinely legitimate reason to separate. If a
  package's consumers need releases on a different schedule than CLUE's, a shared repo
  starts to chafe.
- **CI scoping.** A monorepo runs a large CI unless we add affected-detection. Real, but
  solvable with path filters or a tool like Turborepo or Nx rather than by splitting
  repositories.
- **Smaller surface for an AI agent.** A small repository is easier to hold in context and
  lower-risk to let an agent roam. This cuts both ways: in a monorepo the agent can *see*
  the consumers of the API it's changing, which usually improves the result more than the
  smaller context helps. An agent can also be pointed at one subdirectory.
- **Access control.** Relevant only if we ever want contributors scoped to one package.

## Publishing is orthogonal to repository topology

If one or two packages — a tile framework, a shared models layer — are meant to be used by
other Concord projects, that does not by itself argue for separate repositories. Packages
can be published to npm straight out of a monorepo workspace, with a tool like
`changesets` handling versioning and changelogs.

That gets the API contract and the semver discipline for external consumers while keeping
atomicity for CLUE's own development. Being published and living in its own repository are
independent properties, and they get conflated constantly.

The consequence for review is the tiering rule above: once a package is published, it
carries an invisible set of consumers, and it should be reviewed at the top tier no matter
where it sits in the internal dependency graph.

## When a package has earned its own repository

The point of writing these down is so the question gets re-answered against a standard
rather than re-argued from scratch each time. A package should move out when one or more
of these is true:

- Its external consumers need a release cadence that actively conflicts with CLUE's.
- Its API has been stable long enough that atomic cross-package changes have stopped
  happening, so the main benefit of co-location no longer applies.
- It needs different access control — outside contributors, a different license, a
  different visibility.
- Its CI or tooling needs have diverged far enough that sharing a configuration costs more
  than it saves.

Notably absent: "it's conceptually separate," "it has its own tests," or "the repo is
getting big." Those are arguments for a package, and we already have one.

The asymmetry matters. Extracting one package from a monorepo is a well-trodden operation
that preserves history (`git filter-repo`, or `git subtree split`). Starting with separate
repositories and later discovering we needed atomicity means merging histories and undoing
a release process. Starting together and extracting what earns it is the lower-risk
direction.

## Relationship to decision records

The choice itself, and the extraction criteria above, are exactly the kind of thing that
belongs in a decision register — see
[In-Repo Decision Records](in-repo-decision-records.md). "Packages live in one repository
unless they meet these criteria" is a rule someone could plausibly violate, which is the
test for whether a decision is worth recording. The import-boundary rules that keep the
package boundaries honest are the matching executable check.

## Prior art

- **Large-scale monorepos** — Google, Meta, and Twitter have all written about running
  one. *Software Engineering at Google* covers the version-control chapter and the "One
  Version Rule," which is the clearest articulation of why duplicate dependency versions
  are the thing to avoid.
- **JavaScript ecosystem monorepos** — Babel, Jest, and React all consolidated into
  monorepos after starting split, and Babel in particular wrote up the reasoning.
- **Workspace tooling** — npm, Yarn, and pnpm workspaces; Turborepo, Nx, and Bazel for
  affected-target detection so CI doesn't run everything on every change.
- **Publishing from a monorepo** — `changesets` and Lerna, for versioning and releasing
  individual workspace packages.
- **Stacked PRs** — Graphite, `git town`, and `spr`, which exist precisely because splitting
  a change into a reviewable stack is valuable and tedious by hand.
- **Boundary enforcement** — `dependency-cruiser` (already used in CLUE), ESLint
  import-boundary rules, and the package `exports` field.
- **Review routing** — GitHub `CODEOWNERS`, and path-filtered required checks.

## Open questions

- What's the right approximation of "this change touches the package's public surface"?
  Entry points and exported types are a starting point, but a change to a widely used
  default value is just as visible to dependents without touching either.
- Should the tier drive *required approvals*, or just *a message to the reviewer*? The
  second is cheaper to get wrong and may capture most of the benefit.
- How do we keep tiering from being gamed, in the ordinary way where a change gets
  arranged to touch only leaf paths so it can merge faster?
- Does the review tier of a leaf change depend on who wrote it? A human-written change to
  a well-tested leaf and an AI-written one may not deserve the same treatment, and it's
  unclear whether that belongs in the same scheme or a different one.
- What does the first split actually look like? Extracting a framework package and a single
  leaf is the smallest experiment that would tell us whether the tiering pays off.
- Is the boundary enforcement worth its configuration cost before the packages exist, or
  does it only become worthwhile once there is something to violate?
- How much CI does affected-detection actually save on a repository this shape, and is that
  saving large enough to justify the added build complexity?
