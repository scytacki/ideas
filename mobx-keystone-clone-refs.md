# Cloning mobx-keystone Subtrees That Contain References

**Status:** parked on 2026-10-04, to discuss before anything is reported upstream.

mobx-keystone's `clone()` gives every model in the copy a new id, but references inside the copy
keep the old ids. A copy made on its own has references that don't resolve. A copy added next to
the original has references that point back at the original's objects. This could be reported
to mobx-keystone as a bug or a feature request. But what a copy *should* do with references is
the same question CLUE answers with hand-written per-tile code. It needs more thought first.

The mobx-keystone technology review (concord-consortium/docs, `docs/Technology Reviews/
mobx-keystone.md`, "References") covers the background: why formal references help with copying,
CLUE's tile copy code, and how keystone's references behave.

## The behavior

Checked against mobx-keystone 2.3.0:

```ts
import { clone, idProp, Model, model, modelAction, rootRef, tProp, types } from "mobx-keystone"

@model("repro/Shape")
class Shape extends Model({ id: idProp }) {}

const shapeRef = rootRef<Shape>("repro/ShapeRef")

@model("repro/Group")
class Group extends Model({
  id: idProp,
  shapes: tProp(types.array(types.model(Shape)), () => []),
  selected: tProp(types.maybe(types.ref(shapeRef))),
}) {}

@model("repro/Doc")
class Doc extends Model({ groups: tProp(types.array(types.model(Group)), () => []) }) {
  @modelAction
  add(g: Group) {
    this.groups.push(g)
  }
}

const shape = new Shape({ id: "s1" })
const g = new Group({ id: "g1", shapes: [shape], selected: shapeRef(shape) })

const c = clone(g)
c.shapes[0].id      // a new id
c.selected!.id      // still "s1"
c.selected!.isValid // false: a standalone copy's reference doesn't resolve

const doc = new Doc({ groups: [g] })
const c2 = clone(g)
doc.add(c2)
c2.selected!.current === g.shapes[0] // true: the copy selects the original's shape
```

- **With `clone(g, { generateNewIds: false })`:** the copy keeps the old ids. Adding it to the
  same tree creates duplicate ids, which keystone resolves silently. Its rules are deliberate
  and tested: an ancestor wins, otherwise the most recently attached subtree. MST throws instead
  ("multiple candidates").
- **MST's `clone`** keeps ids, so a standalone MST copy's references resolve to its own objects.
- **Cause:** with `generateNewIds`, `internalFromSnapshotModel` (`model/newModel.ts`) calls the
  `idProp`'s default function for each model and keeps no record of old id to new id. A `Ref`'s
  `id` is a plain string prop, so it is copied as is.
- **Why keystone's tests miss it:** its clone tests in `rootRef.test.ts` use a `Country` model
  with a custom `getRefId`, whose ids never change.
- **Existing reports:** none, as of 2026-10-04. #158 is about plain cloning.

## Why it matters to us

When a copy includes both a reference and its target, we usually want the reference rewritten to
the target's new id. When the target is outside the copy, we want the reference kept. CLUE does
this with plain string ids and per-tile hooks:
- about 380 lines in all;
- three tile types regex-replace old ids in their stringified JSON.

It still misses cases: shared model ids, and ids inside tiles. With typed references, a library
could do this generically. If keystone's `clone` did, a CLUE-like app built on keystone wouldn't
need the per-type code, at least for data held in formal references.

## Questions to settle before reporting

- **Fix in keystone, or our own pass?** keystone snapshots mark every reference with its
  reference type's `$modelType`. So a generic pass over a snapshot could map old model ids to
  new ones and rewrite the references whose ids are in the map, with no per-type code and no
  change to keystone. Which is better: asking keystone to change `clone`, or owning that pass
  (perhaps alongside the planned migration pass)?
- **Default or option?** Changing what `clone` does by default could break code that relies on
  the copy pointing at the originals. A `CloneOptions` flag avoids that, but makes the useful
  behavior opt-in.
- **Which references can be rewritten?**
  - `rootRef`s whose ids come from `idProp` are clear.
  - References with a custom `getRefId`, and `customRef`s that resolve through your own function,
    may not map to model ids at all. They should probably be left alone.
- **Plain string ids aren't covered.** Much of CLUE's copy code rewrites ids that aren't formal
  references, such as data set, attribute and case ids. A keystone fix only helps with what's
  modeled as a reference. Would we move those to references, given how much MST's strict
  references have hurt? (The review's "Where MST's strictness hurt" covers that.)
- **Copying into a different tree.** When a copy goes into another document, a reference to a
  target outside the copy may point at nothing there. Keep it, drop it, or let the caller decide?
- **Duplicate ids.** Should keystone warn in development when ids are duplicated, rather than
  resolving silently? Fixing `clone` removes the main way duplicates arise.

## Draft issue

For use if we decide to report it. A suggested fix:
1. Keep a `Map<oldId, newId>` on `FromSnapshotContext` and fill it in `internalFromSnapshotModel`.
2. After the whole snapshot is created, set the new id on every `Ref` created in that call whose
   `id` is in the map. This has to wait for the end, because a reference can come before its
   target in the snapshot.
3. Leave references with a custom `getRefId` or `getId` unchanged.

If changing the default is a concern, offer it as a `CloneOptions` flag.

> **Title:** `clone()` gives cloned models new ids but leaves refs inside the clone pointing at
> the old ids
>
> With the default `generateNewIds: true`, `clone` gives every model in the copy a new id. A
> `rootRef` inside the copy keeps its old target id, though, so in a standalone clone the
> reference doesn't resolve, and in a clone added to the same tree as the original it points at
> the original object. *(Repro as above.)*
>
> **Expected:** a reference whose target is inside the cloned subtree is rewritten to the
> target's new id. A reference to something outside the subtree keeps its id.
>
> **Actual:** as above. The only workaround is `generateNewIds: false`, which creates duplicate
> ids when the copy goes into the same tree.
>
> *(Cause and possible fix as above.)*
>
> (Written with Claude's help. The repro was run against mobx-keystone 2.3.0.)

The style follows mobx-keystone #590, which we reported on 2026-10-02 and which was fixed and
released in 2.3.0 about a day later.
