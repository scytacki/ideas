# Balance Browser Test Shards Without Cypress Cloud

Split Cypress and Playwright specs across parallel CI jobs ourselves, using each spec's recorded
run time, so we don't depend on Cypress Cloud to balance them. Have one job build the app for the
test jobs, so each one doesn't install and compile it again.

## Motivation

Our Cypress Cloud account is maxed out, so it no longer spreads specs across the parallel
`cypress (N)` jobs. One job gets the whole suite. On activity-player that job takes about 13
minutes while the other five finish in about 1.5, and the S3 deploy waits for all of them. A
deploy now lands about 15 minutes after a push, and codecov/project reports a failure until the
slow job uploads its coverage.

Some of our Playwright projects need sharding now or will soon. Playwright's own `--shard=i/n`
(checked in 1.65) is deterministic: every job works out its share from `i` and `n`. But it splits
by number of tests, not by how long they take. The undocumented `PWTEST_SHARD_WEIGHTS` variable
can only give whole shards different weights, not individual tests.

Kirk added a local parallel runner to CODAP (`npm run test:cypress:parallel`, in
`v3/scripts/cypress-parallel/`, CODAP PR #2832):
- **Specs.** `findSpecs` lists them from the same glob and exclusions as `cypress.config.ts`.
  The pattern is copied, and a test fails if the copies drift apart.
- **Times.** `.cache/cypress-parallel/timings.json` maps each spec's path to how long it last
  took, in milliseconds. Only the last run's time is kept. A spec that crashed keeps its earlier
  time. The time covers the whole Cypress process, since each spec runs in its own `cypress.run()`.
- **Order.** `orderSpecs` puts specs with no time first, then the rest longest first.
- **Workers.** `runPool` runs a fixed number of workers, each taking the next spec from one
  in-memory queue. In a single Node process, taking from the queue needs no locking, so locally
  the shards adjust as they run.

CI can't share an in-memory queue between jobs, so it needs either a plan made before the jobs
start or a way for jobs to claim specs safely (both below). The spec list, the time format and
the ordering carry over.

## Existing tools

Checked on npm and GitHub in November 2026:

- **[cypress-split](https://github.com/bahmutov/cypress-split)** (Gleb Bahmutov, MIT, about
  400k downloads a week, released this month). A Cypress plugin: each job passes `SPLIT` and
  `SPLIT_INDEX`, from `strategy.job-total` and `job-index`, and runs its share of the specs.
  With `SPLIT_FILE`, it splits by a timings JSON so each job's total time is about equal. A spec
  missing from the file gets the average time. It writes the times back after a run, and
  `cypress-split-merge` combines the files from parallel jobs. `cypress-split-preview` shows the
  split without running anything. Each job works out its own share, so every job has to read the
  same timings file. That fits the planning job below: restore the timings once, then pass the
  same file to every job.
- **[@nsxbet/playwright-orchestrator](https://github.com/NSXBet/playwright-orchestrator)** (MIT,
  v2, about 2.5k downloads a week, 4 GitHub stars). Almost exactly the design below, for
  Playwright: an `orchestrate` job computes every shard's tests once, the matrix jobs each run
  their list through Playwright's `--test-list` flag (1.56 and later), and a final job merges
  the times into the `actions/cache`. It splits individual tests, not just files. It is small and
  maintained by one company, so it's worth reading, but risky to depend on.
- **Playwright itself** only splits by test count. A PR to balance shards with a timing file
  (microsoft/playwright#30388) was closed without merging in 2024.
- **Services that hand out specs while the jobs run:**
  - [sorry-cypress](https://github.com/sorry-cypress/sorry-cypress) is a self-hosted stand-in for
    Cypress Cloud's director. It runs on our own servers with a database, it is Cypress only, and
    its last release was in January 2024.
  - [Currents](https://currents.dev) is a paid hosted service for Cypress and Playwright.
- **Splitters for any language:** the `split-tests` GitHub Actions and Depot's `tests run`
  (beta) split by timings from JUnit reports.

So for Cypress, cypress-split could do the splitting and keep the times, leaving us only the
workflow around it. Playwright has no well-used tool, so either we adopt the orchestrator or we
write the small part ourselves.

## Approach

### Record how long each spec takes

After each run, every test job writes the time of each spec it ran. Cypress reports a duration for
each spec in its run results, and Playwright's JSON reporter does the same for each test, which can
be summed per file.

Concurrent runs must not lose or mix up each other's data. Two ways to store the times:

- **One object per run, never changed.** For example, `timings/<repo>/<run-id>.json` in S3. A reader
  takes the median of each spec's recent times, say from the last 10 master runs. No two runs
  write the same object, so nothing can be overwritten.
- **One rolling file in the GitHub Actions cache.** Cache entries can't be changed once saved, and
  a restore by key prefix returns the newest. After a run, a job merges its times into the file it
  restored and saves the result under a new key. When two master runs finish together, one run's
  times are lost, but every saved file is complete. A lost sample barely moves a median. Branches
  can read master's cache, and this needs no AWS access, so it works in repos without an AWS role.

Only master runs should count, so a branch with a broken or skipped spec doesn't distort the times.

Spec times should be keyed by path, as Kirk's runner does, not by a hash of the spec's content.
How long a spec takes depends on the app code and the runner more than on its own text, and a
hash would throw away good data after a trivial edit.

A spec with no time can't go first in a plan made in advance the way it does in Kirk's queue,
since the plan has to give it a weight. It gets the median spec time instead.

CI runs a shard's specs in one Cypress process, so a spec's time there leaves out the startup
that Kirk's runner counts for each spec. The two sets of times shouldn't be mixed.

### Plan the shards in one job

One job lists the spec files from the checkout, so new specs are included. It reads the times
once and gives the specs to the shards: longest first, each to the shard with the least total so
far. It outputs the assignment as JSON, and the test jobs build their matrix from it:

```yaml
cypress:
  needs: build
  strategy:
    matrix:
      shard: ${{ fromJSON(needs.build.outputs.cypress-shards) }}
  steps:
    - run: npx cypress run --spec "${{ join(matrix.shard.specs, ',') }}"
```

Every test job gets its list from the same snapshot of times, so no spec runs twice or is left
out. Each test job doesn't read the times on its own: a run finishing in between would give them
different data.

The same plan can produce Playwright's file lists. A Playwright shard then runs
`npx playwright test <files>` instead of `--shard`.

### Build once for the test jobs

The test jobs have to wait for the planning job, so that job might as well build the app too. The
build costs that job a few minutes, and each test job saves its own `npm ci` and compile. The test
jobs then serve the built files with a static server instead of running `webpack-dev-server`.

Today on activity-player the app is built three times on each push:
- The Jest job runs `npm run build`, but only to check the build works. Its output isn't kept.
- Each of the six Cypress jobs, and the Playwright job, compiles a dev build in its own dev server.
- The S3 deploy job runs `npm ci` and builds again.

CODAP and CLUE already build once and upload the result, but only their deploy job downloads it.
Their Cypress jobs still each run `npm start`.

Other jobs need a build too, which could argue for building in one of them instead. Better to
give the build its own job that builds the app, plans the shards and
uploads artifacts. The deploy job downloads its build from there instead of building again, and
Jest moves to its own job, since it doesn't need a build.

One catch is coverage. Cypress gets its coverage from code instrumented by istanbul at build time
(`CODE_COVERAGE=true` in `npm start`), and that build shouldn't be deployed. Playwright gets its
coverage from Chromium's own measurements (V8), so it can use the deploy build, as long as the
build has source maps. Unless Cypress coverage moves to V8 as well, the build job makes two
builds: one to deploy and one instrumented build for Cypress.

Running the tests against a production build rather than the dev server also tests what we ship.
It may expose tests that depend on the dev server's behavior. The first repo to try this would
find those.

### Shards that adjust while they run (later, only if needed)

A plan made in advance can be off when a spec's time changes, or when one runner is slower than
the others. If the shards still finish far apart, the test jobs could claim specs while they run.
S3 conditional writes make this possible without a queue service. A `PutObject` with
`If-None-Match: *` creates the object only if it doesn't exist, and otherwise returns 412. Each
job walks the planned order, longest first, and tries to create `claims/<run-id>/<spec>`. If the
create succeeds, that job runs the spec; on a 412 it moves on to the next one. Test this before
relying on it. Amazon's SQS would also work, but it needs a queue made and removed for each run,
IAM rules for it, and a way to deal with a job that dies part way through a spec. GitHub has no
queue service.

The plan made in advance is worth building first, and measuring, before this.

## Shared pieces

- One script that takes the times, the spec list and the shard count and prints each shard's spec
  list, unless cypress-split and the Playwright orchestrator (above) are adopted instead. Our own
  script would build on Kirk's `findSpecs` and `orderSpecs`. The new part is short: give each
  spec, longest first, to the shard with the least total time. CI uses it for Cypress and
  Playwright. Kirk's runner lives in CODAP's `v3/scripts`, so sharing it with other repos means
  moving it into a package.
- One reusable workflow, or a composite action, for the build-and-plan job and the step that
  records times. It could live in `s3-deploy-action` or a new repo of shared CI actions.

## Open questions

- Should CI keep only the last time for each spec, as Kirk's runner does, or a median of recent
  runs? A median is steadier when one run hits a slow runner.
- S3 or the Actions cache for the times? The cache needs no AWS access, but entries are removed
  after 7 days unused, which matters for a repo that rarely pushes to master.
- How many shards? With the plan made in advance, a run can never finish faster than its slowest
  single spec takes. Past that point, more shards only cost runner minutes.
- How do Cypress's times get recorded without Cypress Cloud? Kirk's runner already uses the Module
  API (`cypress.run()`), whose results include each spec's duration. A CI shard could run its
  list the same way and write out the times.
- Can activity-player's Cypress coverage move to V8 like Playwright's? Then one build would serve
  both test runners and the deploy.
- Codecov's project check would still fail until every shard has uploaded. With balanced shards
  they would all finish at about the same time, so that gap should be short.
