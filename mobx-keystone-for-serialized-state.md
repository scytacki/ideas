# mobx-keystone for Serialized MobX State

Use mobx-keystone instead of MobX-State-Tree (MST) for MobX state that we save and load.
Its class-based models are easier to write and type than MST's `types.model()` chains,
and it provides the same things we rely on MST for: patches, snapshots, action recording
and replay, and runtime type checking. It also has built-in undo.

Neither library gives us what we need most, which is migrating old saved documents to a
new shape. mobx-keystone's design does make it easier to add that ourselves, later, as a
separate pass that runs before loading. Until then, the one thing to do from the start is
put a version field on every independently saved piece of state.

The first place to try this is neural-pathways (NPW-30), where the state is small: each
lesson view keeps a query, a selected conversation id, some step progress, and a few
flags, and one piece of state is shared between views. The findings below come from CLUE,
CODAP v3, and a close reading of mobx-keystone 2.2.0, so the idea is meant to carry over
to those projects too.

## Motivation

Our documents contain models whose saved state changes over time: a tile gains a feature,
a field is renamed, data moves from inside a tile to a shared model. Loading an older
document means converting its saved JSON into the current shape. Sometimes that
conversion has to look at the whole document, not just the model being loaded. The
standard example is a table tile that used to hold its own data and now points to a
shared data set at the document level. Migrating it means adding an entry at the document
root and linking the tile to it.

MST's snapshot processors (`preProcessSnapshot`, `postProcessSnapshot`,
`types.snapshotProcessor`) are the only migration mechanism it offers. They see only the
snapshot of the model they are attached to, and they can only return that model's own
subtree. Everything that crosses a model boundary ends up as hand-written code somewhere
else. We have also hit enough bugs in the processor mechanics that CLUE keeps a test file
(`src/models/mst.test.ts`) just to pin down MST's behavior.

Our fork of MST (`@concord-consortium/mobx-state-tree`) does not address any of this. Its
only remaining difference from upstream is an `onAction` option, `allActions`, that
reports nested actions.

## What we have hit with MST

Evidence is from CLUE (`collaborative-learning`) and CODAP v3 as of September 2026. File
paths may have moved since.

**Migrations that cross model boundaries are split into two steps.** In CLUE, a
per-model pre-processor stashes the legacy data inside the model. Then an `afterAttach`
reaction or `afterCreate` finishes the job with MST actions, once the shared model
manager exists. Examples:

- A table's legacy data set is parked in an `importedDataSet` prop, then turned into a
  `SharedDataSet` (`src/models/tiles/table/table-content.ts`).
- Geometry links to tables are kept in a `links` prop and resolved later. Unresolved
  ones sit in `remainingLinks` and are retried, because nothing guarantees the table's
  migration ran first.
- Tile titles move into shared data set names in
  `BaseDocumentContent.afterCreate` (`src/models/document/base-document-content.ts`).

CLUE also added a `tileSnapshotPreProcessor` hook
(`src/models/tiles/tile-content-info.ts`) because a tile's content model cannot see its
tile. Loading an old document therefore changes it with ordinary actions, after load,
with the order decided by MobX reactions.

**There are no versions, so migrations guess the version from the JSON.** Neither app
has a document-level schema version:

- CODAP's `isLegacyDataSetSnap` recognizes five older data set shapes
  (`v3/src/models/data/data-set-conversion.ts`).
- In CLUE only Drawing, Diagram, and Question carry a version.
- CLUE's drawing export dropped its version field, so the drawing migrator now relies on
  a `looksV1` heuristic (`src/plugins/drawing/model/drawing-migrator.ts`).

Migrations that detect old shapes run on every load forever and can never be retired.

**The processor mechanics have bugs and surprises:**

- Wrapping a type in `types.snapshotProcessor` rewrites the base type's `create()`
  (mobxjs/mobx-state-tree#1897, still open).
- `getType()` on a processed instance returns the base type.
- `applySnapshot` skips `types.snapshotProcessor`'s pre-processor (#1317), so CLUE
  migrates by hand first.
- Pre-processors run twice per create.
- For `types.maybe` to work, a pre-processor has to return `undefined`, which its
  TypeScript type forbids.
- A pre, post, pre round trip lost a table's data. The table's post-processor was removed,
  and every saved table still carries `importedDataSet`.

**Saving can't read the live model.** An MST post-processor sees only the snapshot, not
the instance. CODAP keeps attribute values out of MST for performance (#1683) and must
write them back into props before each save. It runs `prepareSnapshot()` and
`completeSnapshot()` around every save (`v3/src/models/document/serialize-document.ts`,
`v3/src/models/data/attribute.ts`). Those are MST actions, so saving changes the tree:

- They need `withoutUndo` and reserved action names in the history monitor.
- They have to support async work, because a plugin's state is fetched over iframe
  messaging.

`types.frozen` values are frozen only in development, which once caused a
production-only bug.

**Export and copy are separate hand-written serializers.** CLUE's `exportJson` builds
strings per tile, and `snapshotWithUniqueIds` rewrites ids by hand. Both have drifted
from the stored shape: they lost a version field, and nested tile ids were lost, which
orphaned shared-model references.

**History patches are never migrated.** CLUE stores history as JSON patches. Removing
the dataflow `programZoom` property broke playback of old patches that referenced it, so
the property was put back as dead state
(`src/plugins/dataflow/model/dataflow-content.ts`). Snapshots tolerate a changed schema;
patches do not.

**Children are created lazily, so hook order depends on access.** A parent's
`afterCreate` could run before its children existed (#1951, #2058).

## What mobx-keystone offers

- **Class models.** `class Foo extends Model({...})` with `this`, inheritance through
  `ExtendedModel`, and `@modelAction`. Recursive and circular models don't need
  `types.late` plus hand-written interfaces.
- **Snapshots and JSON patches,** including inverse patches.
- **Action recording and replay.** `onActionMiddleware` observes actions and
  `serializeActionCall` turns them into JSON. `applySerializedActionAndTrackNewModelIds`
  and `applySerializedActionAndSyncNewModelIds` replay them; they were designed for
  client/server sync.
- **Built-in undo/redo** (`undoMiddleware`), with grouping, `withoutUndo`, a step limit,
  and extra state attached to each step.
- **Transactions,** which roll an action back if it throws.
- **Runtime type checking** through typed props (`tProp`), tested below.
- **References** that resolve within a tree or through a resolver you write, with
  back-references. Tested below.
- **Children are built when their parent is loaded**, and `onAttachedToRootStore` runs
  top-down once the whole tree exists.
- **Yjs and Loro bindings** (`mobx-keystone-yjs`, `mobx-keystone-loro`) that keep a store
  synchronized with a CRDT document. This is relevant to CLUE's collaborative editing
  plans.

It is actively maintained: 2.2.0 was released in September 2026 and supports MobX 7 and
both legacy and standard decorators.

## How mobx-keystone serialization works

- **Where state lives.** A model's data is `$`, an observable plain object that is part
  of the tree. Each node keeps its own saved form next to the instance. That snapshot is
  updated incrementally on every change and shared by reference with its parent's
  snapshot. `getSnapshot` returns the cached object, so the result keeps the same
  identity while nothing has changed.
- **Choosing a class on load.** The class comes from the snapshot's `$modelType`, looked
  up in a global registry of model names. Loading a model whose `$modelType` is not
  registered throws, and there is no fallback.
- **Load order is top-down.** The root's `fromSnapshotProcessor` receives the whole raw
  subtree before any child is created. Children are then processed the same way.
- **Processors have strict rules.** They must be pure and deterministic, and the current
  output snapshot must pass through `fromSnapshotProcessor` unchanged. The library may skip
  the processor when the input equals the current output, and it caches
  `toSnapshotProcessor` results.
- **One-way upgrades are allowed.** An old shape is never current output, so a processor
  may upgrade it without a way back. The library's own `UndoStore` does this.
- **What a processor can't change.** It can't change its own `$modelType` or id. It can
  rewrite anything below it, and nothing outside its own subtree.
- **Saving still can't read volatile state.** `toSnapshotProcessor` receives the instance,
  but it reruns only when `$` changes and its results are cached, so CODAP's save-time
  problem is not solved.
- **Patches bypass processors.** Patch paths use the raw `$` keys. If a processor changes
  the saved shape, patches and saved snapshots stop matching.
- **Unknown props are kept.** Extra keys in a loaded snapshot are kept, not rejected.
- **No built-in versioning.** The maintainer's advice (xaviergonz/mobx-keystone#52,
  February 2026): use `fromSnapshotProcessor` for local changes. For structural ones
  (split, merge, re-parent), migrate "at the lowest common parent (often root) or in a
  pre-load snapshot migration function before `fromSnapshot`". In practice: "keep a root
  schema version and run pure snapshot migrations vN -> vN+1 before loading."

### Checked by experiment

These were run against mobx 7.0.5 and mobx-keystone 2.2.0 in a throwaway script.

**`$modelType` is required in the saved JSON.**

- Loading can leave it out where a `tProp` names exactly one class, or for the root when
  calling `fromSnapshot(Type, snapshot)`.
- Output always includes it, and the key name is a hardcoded constant.
- A custom type field such as `kind` can work in two ways:
  - A `types.or` dispatcher: output still gets `$modelType` added, and the list of types
    is fixed when the union is defined.
  - A processor on the parent prop that maps `kind` to `$modelType`: the saved JSON
    round-trips, but patches still carry `$modelType`.
- Either way, the names act as permanent ids.

**Tile types can be registered at runtime.**

- A prop typed with a base class, `tProp(types.array(types.model(TileContent)))`,
  creates whichever registered subclass the snapshot names. That includes subclasses
  registered after the document class was defined and used.
- A model that is not a `TileContent` is rejected.
- The base class must itself be registered with `model()`. Otherwise `types.model()`
  fails, and a tile snapshot without `$modelType` loads as a plain `TileContent`.
- Tile modules register themselves when imported. Since TypeScript can drop type-only
  imports, `registerModels(...)` exists to force registration.

**Type checking on load.** Loading a snapshot with a wrong-typed value in a `tProp`
threw a `SnapshotTypeMismatchError` in every mode, including production and
`AlwaysOff`. The error names the path and the chain of models above it. Writes after
load are checked according to `modelAutoTypeChecking`: development only by default, or
everywhere with `AlwaysOn`. Untyped `prop()` values are never checked. Only a wrong
primitive was tested; refinements and deep nesting were not.

**References.**

- **`rootRef`** resolves by id within the same tree. It is saved as
  `{ id, $modelType: "<ref name>" }`.
  - `onResolvedValueChange` fired when the target was removed, and
    `getRefsResolvingTo` found the back-reference.
  - A dangling reference on load does not throw: `isValid` is `false`.
  - It does not resolve across two separate trees.
- **`customRef`** resolves through a function you write. It worked against a model in a
  separate tree and against plain data outside any tree, and it picked up changes to the
  target.

## Where our MST problems go

| Problem | With mobx-keystone |
|---|---|
| Processor bugs and surprises | Mostly gone: processors are model options, not wrapper types |
| Hook order from lazy creation | Gone |
| Migrations that cross models | Possible only at the root or in a pass before load |
| Versioning | Not provided |
| Saving volatile state | Not solved |
| Export drift | Not addressed |
| Migrating history patches | Not addressed |

It also brings new constraints: an unknown `$modelType` throws, a model can't change its
own type or id, and `$modelType` names are permanent.

## Approach

**Now.** Adopt mobx-keystone where serialized state is small, starting with
neural-pathways:

- Put a version field on each independently saved piece of state. In neural-pathways
  that means each view's state, which becomes that interactive's Activity Player state,
  and the state shared between views.
- Keep the saved form identical to the keystone snapshot. Don't use
  `toSnapshotProcessor`s that change its shape, so that patches and undo history match
  what is stored.
- Use namespaced `$modelType` names (`npw/TraceACaseState`) and never rename one, the same
  rule we follow for view ids.
- Use `tProp` for everything that is saved, so it is type-checked on load.
- Refer to data outside the tree, such as conversations from the dataset, by a plain id
  string, not a `Ref`. Use `customRef` for links between separate trees.
- Anything that should be saved goes in props. Don't depend on a save-time hook to
  gather it: that is what led to CODAP's `prepareSnapshot`.

**Later, when a migration is first needed.** Add a pure JSON migration pass that runs
before `fromSnapshot`, so keystone only ever sees the current shape:

- Each step takes the whole document, keyed by its version (v1 to v2, v2 to v3, and so
  on), and does nothing at the current version.
- Because it sees the whole document, a step can do what MST processors can't: add a
  shared model at the root and link to it, rename a `$modelType`, fill in missing ids, or
  replace an unregistered type with an "unknown" wrapper model.
- Each step can be tested as fixture in, fixture out.
- Once documents are saved again at the current version, old steps can be dropped or
  moved to an offline upgrader.

keystone's processors would not be used for versioning at all. The pass needs nothing
from the library, which also keeps the switch away from mobx-keystone cheap if it doesn't
work out.

## Costs

- The team knows MST and not mobx-keystone, so reviewers will need to learn it.
- It has one main maintainer, although he is very active.
- `$modelType` appears in every saved model and has to be treated as a permanent id.
- An unknown `$modelType` throws. Tolerating unknown or newer content needs the pass
  before load.
- Processors are not inherited by `ExtendedModel` subclasses; share a function instead
  (#492).
- Decorators need `experimentalDecorators` or standard decorators, which are supported.

## Prior art

- **mobx-keystone docs:** class models and snapshot processors, references, runtime
  type checking, the MST migration guide, and the undo and action middleware pages.
- **mobx-keystone issues:** #52 (versioning; the maintainer recommends a root schema
  version with pre-load migrations), #285 (how processors replaced MST's
  `snapshotProcessor`), #492 (processors are not inherited).
- **MST issues we have filed or hit:** #1897, #1317, #1683, #1951, #2058, #1519.
- **Versioned migration chains elsewhere:** `createMigrate` in redux-persist, Rails
  Active Record migrations and Flyway for databases, and MongoDB's schema versioning
  pattern. All of them record a version with the data and run numbered, ordered steps.
- **CLUE's history framework** (`docs/history-framework.md` in collaborative-learning),
  for how patches are stored and replayed today.

## Open questions

- Should versions belong to the document, to each model type, or both? One document
  version is simple, but it couples every tile type, and CLUE's tiles act like plugins
  that change on their own schedule. A hybrid would let each model type register ordered
  steps, run by the document pass. Each step would get read access to the whole document
  and a way to add root-level entries such as shared models.
- How should recorded history survive a schema change? The options are migrating
  patches, stamping each patch with the version it was recorded under, or replaying from
  migrated snapshot checkpoints.
- What should the "unknown type" fallback look like, and should the migration pass or a
  root processor apply it?
- Is there a keystone-friendly answer to CODAP's save-time problem, or is "anything saved
  must live in props" the right rule even when it costs memory?
- Should the migration pass be a shared package once a second project needs it?
- How far does type checking on load go? Refinements, nested models, and unions were not
  tested.
- Is neural-pathways enough of a trial to judge the switch for CLUE or CODAP, given that
  its state has no tiles, shared models, or history?
