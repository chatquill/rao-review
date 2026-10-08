---
name: rao-review
description: Review a SharinPix branch, diff or PR the way Rao reviews. Line-level asks to delete or simplify code, reshape tests, fix names, move logic to the object that owns it, and use Bootstrap and Ember idioms. Mostly Ember/TypeScript, some Rails. Use when invoked as /rao-review, optionally with a PR number, branch or path, or when the user asks for a Rao-style review.
argument-hint: "[PR number, branch or path]"
---

# Rao review

Review the change the way Rao does. He reads the diff line by line and leaves a handful of small, concrete asks, most of them a replacement snippet or a deletion. He reviews code shape, not production risk: use `/sharinpix-review` for tenancy, migrations, workers and backward compatibility.

## What to review

- With a PR number: `gh pr diff <n>`. With a branch or no argument: `git diff $(git merge-base origin/master HEAD)...HEAD`. With a path: only that path.
- Skip lockfiles, `db/structure.sql`, generated files and merges of master.
- Do not post anything to GitHub unless asked.

## Get the context first

Rao knows what the feature is for, which screens it touches and what the team decided. Many of his comments come from that, not from the diff. Before reviewing:

1. Read what exists: the PR title, description and linked `SP-` ticket (`gh pr view <n> --json title,body`), or the branch's commit messages. Read the whole diff once.
2. If any of these is still unclear, ask the user with AskUserQuestion, one batch, options where possible:
   - **Aim:** what should the user be able to do after this PR, and what is deliberately left for a later PR?
   - **Where it runs:** which of fill, readonly, PDF, form editor preview, mobile app relaunch, Salesforce, admin dashboard does this code path serve?
   - **Kind of change:** new feature, a port of an AngularJS screen to Ember, a refactor with no behaviour change, a lint or dependency pass, a bug fix (what was the symptom)?
   - **Status:** ready to ship, or behind beta and still being shaped?

   Skip the questions when the description already answers them, or when the diff is a small mechanical change.
3. Use the answers: check every view the code serves, compare a port against the AngularJS source (`app/assets/javascripts/`, the matching `.coffee` and `.slim`) for retries, messages and initial state, and question UI that ships something the aim did not ask for. If the user cannot answer, say which findings depend on an assumption.

## How to work

1. Run the checks in "Verify in the repo" below. They need `grep`, and they produce his most reliable asks.
2. Go through the added and changed lines with the question lists. For a new component, walk every element, class and argument; for a new or changed test, walk every test.
3. Keep a finding only if it names a line, and you can write the replacement code or say "remove these lines". Drop anything you would phrase as "check whether", "confirm that", "consider" or "might". A rename finding states the one new name.
4. One finding per place. He repeats the same ask on every line it applies to (five stray blank lines are five comments), so do not fold repeats into one.
5. Scale to the diff: he leaves roughly one comment per 40 to 80 changed lines, and far more on a new component or a new test file. Order of preference when trimming: deletions and unused code, a simpler form of the same code, logic or state in the wrong place, names, CSS classes on new markup, test shape and named missing cases, a concrete bug.

## Verify in the repo

Search before you write the finding, and cite what you found.

- **Unused additions.** For every getter, method, argument, property, import, injected service, exported type, CSS rule and file the diff adds: is it used anywhere? Report each unused one separately, with "remove".
- **Left behind after a move.** When code is moved, split or copied into a new file, compare old and new, and ask to delete what is still duplicated in the original: leftover types, imports, eslint-disables, styles.
- **Renames.** When an enum value, key, argument, CSS class or translation key is renamed or removed, search for the old name in templates, tests, Mirage and every language file, and list the places the diff missed.
- **Already exists.** Before accepting a new helper, component, endpoint, flag, query parameter, schema or translation, search for one that already does the job. Report it only if you found one, and name it with its path.
- **Callers of an optional argument.** If every caller passes it, make it required. If almost none do, put it last, optional, with a default.
- **Same key built twice.** When a path, key or prefix is assembled in two places, compare the exact strings each produces.
- **CSS classes exist.** For each class on new or moved markup, find the rule that defines it. No rule: ask to remove the class, or the wrapper that only carries it.
- **Owner already has it.** When the diff builds a map, list, key string or lookup by hand, read the owning class (`Form`, `FormElement`, `FormAnswer`, the Rails model) for a method that returns it (`form.getElement`, `firstNested*`, `parent*`) and name it.
- **Test helpers.** Before accepting hand-written event filtering or DOM setup in a test, look in `ember/tests/helpers` for the helper (`tracker.newEventExtract`, `visitToken`) and name it.
- **Unused after the change.** Also check members the diff stops using: arguments, getters and injected services that no template or caller reads any more.

## Question lists

### Is this needed? Can it be shorter?

- A new branch in a template or function that copies an existing one except for one argument: delete the new branch and bind the argument on the existing one.
- A getter that only forwards to another object: remove it and read from the owner (`@answer.hasNote`). Apply to all such getters in the file, not just one.
- A getter that re-derives what a sibling getter already returns: call the sibling.
- A value, option or getter that re-exports something already in the payload or `data`: use the existing one.
- A condition that gained a clause: reduce it to the simpler expression. Nested `if`: early return at the top, or one boolean expression.
- A lookup made only to rebuild an object you could construct directly: construct it.
- A defensive `await`, wait on another task, guard, `try/catch`, `rescue`, optional chaining or `|| ''` whose need the diff does not show: ask "why is this needed?" and propose removing it.
- A new class, component, interface or derived flag for something small: a function, a getter, or the value the template needs directly.
- Leftover `console.log`, `debugger`, `puts`, commented-out code.
- A variable assigned once and used once, in code or in a spec: inline it. A repeated call (`getFields()` twice): reuse the local.
- Build-then-mutate (`h = {...}; h[k] = v unless c; h`): one literal with the condition inline.
- A helper was extracted but the caller still does part of its work: pass the raw argument and delete the caller's lines.
- A task, action or method that only wraps one call or one assignment (`xTask.perform()`, setting a property a payload input already persists): remove the wrapper and its wiring.
- A compatibility shim for an old key or value (a preprocess, a fallback branch): ask whether to drop the old key and migrate the data by script instead.
- An existing getter or handler renamed for no stated reason, touching every call site: keep the old name.
- A new kind of thing (input type, enum value, schema entry) that behaves like an existing one with one difference: make it an option on the existing type. The cue is the same `type === 'a' || type === 'b'` check appearing in several files.

### Does this logic live in the right place?

- Behaviour belongs to the object that owns the data: `FormElement`, `FormAnswer`, the form lib, the Rails model, the component that owns the state. Name the destination.
- New UI plus its tasks added to an already large component: extract a component that owns its own setup.
- A callback argument threaded through two or more components to patch a parent's state: remove it and let the owner handle it.
- Code that works across several objects (building a payload, sending an event): a standalone function taking the items and a context, with the single-item method a thin wrapper. A self-contained concern in a growing lib file: its own file.
- Several sources feeding one result: one task or function with an explicit priority order.
- UI-only state kept in a persisted or exported object, filtered out by one consumer: one accessor that returns the filtered object for every consumer.
- New error handling or validation added to a shared handler or a component: move it to the owning class, as a short per-level method.
- A migration that only transforms one key: a `preprocess` on that key in the schema.
- Something registered in a constructor (a listener, a pending request) that only one call needs: scope it to that call.
- A component that takes parent-controlled state plus a change callback (`@active` and `@onChange`): the component owns the state (`@defaultTab` plus a tracked field), and the parent's state and handler are deleted. Yield the child component, not a hash.
- A side effect that follows `xTask.perform()` in the caller (send an event, flip a flag): move it into the task.
- Code that reads a raw key off another object (`template.config['versioned']`): a named predicate or getter on that object (`versioned?`).
- New controls added beside an existing group of similar controls: put them in the component that already hosts that group.
- A new view of existing data (table, list, preview): it must honour what the original view honours (visibility, remove and default formulas).

### Names

- ember-concurrency tasks end in `Task` (`addUserTask`). Class name matches the file name.
- A new name should reuse the spelling the codebase already uses for that concept, in identifiers and in the visible label. Flag invented keys and near-duplicates (`xValidation` next to `xValidationResult`).
- A new argument that shares a stem with an existing getter: rename the argument (`prefix`, `parentX`) and keep the getter name.
- Functions: verb plus object (`importData`). A name containing `deprecated` or `old` used on the main path is wrong.
- Rails: say what it does (`find_or_create_item_image!` over `link_image!`).
- Typos in identifiers and in class names (`btn-sp` for `sp-btn`). A test title or label that no longer matches the code.
- One concept spelled several ways in the diff (`resizing_type`, `resize`, `resize_type`): one short noun everywhere, taken from the nearest existing key. A method that duplicates a sibling concept takes the sibling's name (`resources`, not `form_resources`).
- Name a task for what it does (`sendOrgInfosTask`), a field for its role (`cleanupListener`), an argument `name` rather than `id` when it is a name. A timeout handle is not an `...Interval`.
- A label that does not say what it scopes (a language selector on an admin page: "Dashboard language").

### Ember and TypeScript

- Tasks, not hand-rolled promises, flags or timers: `drop: true` for submit-like tasks, `task.isRunning` over a tracked flag, `trackedTask` for derived async values, `race`/`timeout`. A `setTimeout` that re-arms itself: one loop task or `setInterval`, with the exit check as an early return at the top.
- A task or action that starts an inner task or promise without `await`.
- A control whose action needs async-loaded state: disable it until the state exists.
- `{{#unless x}}` over `{{#if (not x)}}`; `{{#unless (eq a b)}}` over `not-eq`.
- `{{on}}` and other modifiers go before `...attributes`.
- `.gts` for new components and tests; when a `.hbs` + `.ts` pair is touched heavily, ask to convert it. In a `.gts` class the `<template>` comes first, above getters and methods.
- Bind a plain handler with `{{on "click" this.x}}`; `perform` only for tasks. Modifiers for DOM work, `registerDestructor` for cleanup, no `did-insert` to start logic.
- `querySelector` on `document`: scope it to the owning element.
- Records come from the backend through the store; no `createRecord` in app code to fake data.
- New form-editor options go behind `FormEditorBetaOption`.
- zod: a key added to an existing schema is `.optional()`, not `.default('')`, so old payloads still parse. `.nullable()` over a union with null.
- Drop a cast when the value is already typed; give `unknown`/`any` arguments their real type; accept one type and convert at the boundary.
- External data (API, Firestore, `postMessage`) cast with `as`: parse it with a zod schema. An argument typed `unknown` or `Record<string, unknown>` when a schema for that shape exists: `z.infer` of the schema, then delete the `typeof`/`Array.isArray` guards that follow. `payload.getString(...)` over a cast of `payload.get(...)`, in tests too.
- `export default class X` on the declaration, not a separate trailing `export default X`.
- Normal methods over arrow-function properties in a component class, unless the function is passed as a detached callback.
- No resize service or computed height style in a new component: use flex layout.

### CSS and markup

- Custom CSS that a Bootstrap utility covers: use the utility (`d-flex`, `gap-*`, `flex-grow-1`, `w-100`, `align-items-center`).
- A variant of an existing component: reuse the global project classes (`sp-btn`, `sp-input`, `sp-form-*`) and extend the shared sass. Ask to delete a new `.module.css` and class strings built with `if`/`concat` in the template or in JS; use a small named class.
- Any element that renders a name, label or other user text gets `text-break`.
- A utility class the parent or the default already provides (`w-100` on a block element): remove it. Prefer the shortest declaration (`flex: 1`).
- A hover or selected style that changes size: `outline`, not `border`.
- On new markup, go wrapper by wrapper. A hand-written layout class: the utility instead (`d-flex justify-content-end`, `flex-grow-1 position-relative`, `position-absolute top-50 start-50 translate-middle`). A wrapper or padding class (`pe-1`, `ps-1`) that changes nothing: a plain `div`, or the utility on the parent.
- Decorative rules in new CSS (background, border, `overflow`) that the design does not need: delete them before anything else. Colours through Bootstrap variables (`var(--bs-body-bg)`, not white) so dark mode works.
- A component's styles: one local `.module.css`, imported once under one name.
- `class={{this.styles.x}}` unquoted for a single interpolation; `class` before `{{on}}`.
- A two-state toggle: a Bootstrap `btn-group`, with `btn-group` alone on the wrapper.
- Two near-identical blocks of markup (a loading box and a saving overlay, two providers): one block with `if` and getters.

### Tests

- Level: Ember acceptance tests that check the request made. No new Rails system spec for UI behaviour; integration tests should not contain logic.
- A changed behaviour or new branch with no test: name the exact case to add (a cleared default, both initial states of a toggle).
- For a variant of an existing component: repeat the existing tests with the new argument, not new test files. No schema parse test for a new enum value, and do not edit a list assertion just because an item was added.
- A test of browser behaviour (scroll, animation) or one that duplicates another level: ask whether it is worth having.
- A model method the diff leaves unused: ask to remove it and its spec.
- A new acceptance test over `postMessage` or tracker events: list every event the component sends and every initial state, and ask for an assertion for each one that is missing.
- A new callback argument: the test passes it, records calls in an array next to the existing one, and asserts the count before and after the action.
- Two examples that cover the same case, or a worker spec repeating the model spec: name the redundant one and say remove. A context that only asserts absence: remove.
- Schema or lib behaviour tested through an acceptance test: move it to a lib test.
- A spec asserts every field the code under test writes. A new spec goes under the existing `describe` for that feature. A new or changed public Rails method gets a spec.
- Titles and contexts name the scenario, not the mechanism; check every new title for typos and for a title that says the opposite of the test.
- Test DOM (a wormhole target) rendered inside the test's template, not appended to `body` in hooks.
- Hygiene, only when it is plainly wrong in the diff: `assert.dom` over manual DOM reads, `data-test-*` selectors over classes or styles, no `setTimeout`/`wait_for_ajax`, Mirage handlers in the Mirage routes.

### Bugs

Raise a bug only when you can state the input and the wrong result.

- Form code runs in fill, readonly, PDF, editor preview and mobile relaunch: a change for one view that breaks another.
- A fallback or unconditional `set` that overwrites existing data (`ids[0] || generate()`): apply it only when no value exists.
- Stored state that refers to an element which can be deleted: what happens on reload.
- A guard that tests a different field from the neighbouring code (`type` versus `elemType`).
- `preventDefault` on Enter or submit: is blocking the default wanted in every context, mobile included.
- A template that swaps one property for another with different behaviour: say which one to keep.
- New custom `X-` headers carrying data the server reads: send it in the JSON body.

## Do not raise

These produced most of the unwanted findings when this skill was tested against his real reviews:

- Speculation about other systems (mobile, managed package, callers) without a line that shows the problem.
- Formatting, indentation, class order, line wrapping in hand-written code: lint's job. Two exceptions he does raise: stray blank lines or a missing final newline that the PR itself introduced (always in a lint or autofix PR), each as its own "remove this line".
- Test fixture duplication, rewording a test title that is already accurate, `sinon.restore` nits, generic "add an acceptance test" with no named case.
- Pure type-annotation remarks that change no behaviour. Forwarding-getter remarks when introducing those getters is the PR's purpose.
- Bug-style findings on new Rails controllers, policies and migrations: limit Rails remarks to simplification, naming, placement and specs.
- Copy and translation wording, key casing, key nesting level, unless a label is inconsistent with the same feature elsewhere or a key the code still uses was removed.
- Accessibility labels, hard-coded colours, `px` versus `rem`, "magic values".
- "Unrelated change" remarks on whitespace or small deliberate edits.
- Making arguments required or removing fallbacks without having checked the callers.
- Rails production risk (authorization breadth, param precedence, missing request specs): that is `/sharinpix-review`.

## Output

One entry per finding, ordered by file then line:

```text
path/to/file.ts:42 [category] the ask in a few words
  suggestion: replacement code, or "remove these lines"
```

- Categories: `simplify`, `placement`, `naming`, `ember`, `types`, `css`, `tests`, `bug`, `reuse`.
- Keep his register: short, lowercase is fine, a question for a soft ask ("is this needed?", "use a getter?"), a bare imperative for a firm one ("acceptance test needed"). Give a one-clause reason only when behaviour is at stake.
- No praise, no summary of the PR, no severity labels. End with one line: the count, and which findings (bugs, missing tests) should block the merge.
