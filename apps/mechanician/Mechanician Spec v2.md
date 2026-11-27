---
class: archive
category:
  - specification
affiliations:
created: 2026-11-25
modified: 2026-11-25
context:
  - "[[Mechanician Spec]]"
  - "[[Accession Spec v2]]"
  - "[[Apparatus Module Taxonomy]]"
  - "[[Apparatus Status Taxonomy]]"
  - "[[Library Accession Trigger Map]]"
  - "[[Entity.js Specification]]"
  - "[[Works.js Specification]]"
tags:
---

The interface base layer of Mechanician: the contracts external code builds against, the registry that admits new class values, the engine that operates on anything satisfying the contracts, and the stock classes that implement them.

## Contents
---

```toc
```

## 1. Scope and Purpose
---

- Mechanician exposes the shape of its classes as contracts. External code, written in TypeScript and compiled by CodeScript Toolkit, builds on them in one of two ways: a bare class implements the contracts, or a class extends a stock class (§2.6).
- The engine (§4) operates on anything that satisfies the contracts, so the write pipeline, schema merging, and Accession reach classes built outside Mechanician as they reach stock classes.
- The analysis surface is frontmatter: `class`, `category`, `affiliations`, status properties, and relational fields. A note written through the engine carries them in the order its layout defines, so Dataview queries and any other service that analyzes frontmatter apply to externally built classes unchanged. A note an implementer writes by other means is outside that guarantee.
- The primary external author is the curator; anyone else who builds on the contracts does so at their own risk. The API is robust regardless of how many authors use it: the versioning rules of [[Mechanician Spec]] §6.1 apply unchanged.
- This document replaces [[Mechanician Spec]]; content not yet carried over is tracked in §6.1. Accession is governed by [[Accession Spec v2]] in place of [[Accession Specification]].

### 1.1 Organization

The repository is a workspace of two packages: `contract`, which holds the contract types and is consumed by external builders, and `mechanician`, the plugin. Folders for content not yet built are created when the content lands.

**Rationale for a workspace.** The installed plugin is compiled JavaScript and carries no type declarations, so builders receive the contract as a separate package.

```
packages/
├── contract/
│   └── v1/
└── mechanician/
    ├── main.ts
    ├── registry/
    ├── engine/
    │   ├── pipeline/
    │   ├── accession/
    │   ├── frontmatter/
    │   ├── resolution/
    │   ├── errors/
    │   ├── schema/
    │   └── containers/
    ├── stock/
    │   ├── core/
    │   ├── creative/
    │   ├── library/
    │   └── publication/
    └── api/v1/
```

| Layer | May import |
| --- | --- |
| `contract` | Itself, and type-only imports from `obsidian`. |
| `registry`, `engine` | `contract`. |
| `stock` | `contract`, `engine`, `registry`, and the stock classes on its own inheritance chain. |
| `api`, `main` | Any layer. |

- `contract` contains no runtime values. Its files may reference one another.
- `engine` and `stock` import the contract types directly. A consumer facade is not ruled out.
- Stock classes claim their `class` values through the registry at load.
- A stock class imports another stock class only along its own inheritance chain. A leaf may import its parents, including a parent in another module (Library's `person` extends Core's), and may not import a sibling or another chain.

## 2. Contract
---

### 2.1 Shape

- The contract is a set of interfaces: the static side (§2.2), the instance side (§2.3), and descriptors (§2.4), alongside the options and result types of [[Mechanician Spec]] §5.1.
- Interfaces are erased at runtime. The registry and the engine verify that a class satisfies the contract with handwritten guards, as schema validation does ([[Mechanician Spec]] §4.1).
- Core code tests conformance structurally and never with `instanceof`, because an implementer does not extend a stock class and so cannot pass such a test.
- The contract types ship as a types package. How the package reaches CodeScript Toolkit is open (§6).

### 2.2 Static side

| Member | Kind | Purpose | Rationale |
| --- | --- | --- | --- |
| `classValue` | static | The `class` frontmatter value the class writes. | A class that extends a stock class keeps its parent's value, so queries keyed on `class` continue to apply. |
| `categoryOptions(container)` | hook | Closed category vocabulary, or `null` for free text. Receives the container note ([[Mechanician Spec]] §4.3). | — |
| `fieldLayout()` | hook | Ordered `FieldLayoutEntry` list, readable without running a constructor or a resolver ([[Mechanician Spec]] §4.4). | — |
| `resolveFolder(app)` | hook | Resolves the folder a note is created in. | — |
| `accessionRules()` | hook | Returns the class's `AccessionRule` list (§2.4). Optional; absent is equivalent to an empty list. | Optional, so a class with no relational fields declares nothing. |
| Constructor | static | `new (app, options)`, with options extending `BaseOptions` ([[Mechanician Spec]] §5.1). | — |

### 2.3 Instance side

| Member | Kind | Purpose | Rationale |
| --- | --- | --- | --- |
| `filename` | instance | The note's name without extension. The `self` value of an Accession rule links to it. | — |
| `folder` | instance | The folder the note is created in. | — |
| `fieldValue(key)` | hook | The value written for a layout key ([[Mechanician Spec]] §6.1). | — |
| `getBody()` | hook | The note body. | — |
| `postCreate()` | hook | Optional for implementers. Whether the engine calls it for an implementer is open (§6). | — |

The exact instance member list is settled against the `BaseClass` specification (§6).

### 2.4 `AccessionRule`

```ts
interface AccessionRule {
	// Fields on the creating note whose links identify the target notes.
	from: string[];
	// Field read off each target; its links, resolved relative to that target, become the notes pushed into.
	through?: string;
	// Relational array field on the target that receives the push.
	targetField: string;
	// Values of `class` the pushed-into note may have.
	targetClass: string[];
	// `'self'` pushes a link to the creating note; `{ fields }` pushes the links held in those fields of the creating note. Every pushed link is a bare `[[basename]]`.
	values: 'self' | { fields: string[] };
	// When true, an unresolvable link or a class mismatch is an error; when false, the push is skipped and reported.
	strict: boolean;
}
```

- A class declares its rules and the engine performs them. Behavior of each member is specified in [[Accession Spec v2]] §2–§5.
- A subclass extends a parent's rules by returning a modified copy of them from its own `accessionRules()`.
- `targetClass` is required on every rule. A rule that names no class pushes into any note it reaches.

### 2.5 Contract evolution

- Within a version, changes are additive ([[Mechanician Spec]] §6.1). A member added to a contract interface after its version ships is optional, and the engine tolerates its absence. A required member ships as a new version namespace.

**Rationale.** An external implementer compiled against a shipped version must keep compiling and running.

### 2.6 Implement or extend

| | Implement the contract | Extend a stock class |
| --- | --- | --- |
| Author writes | The static side and the instance members. | Overrides of the hooks that differ from the parent. |
| Gets | The write pipeline, layout and schema handling, and Accession, through the engine. | Everything the engine provides, plus the defaults the stock class defines. |
| `class` value | A new value, registered (§3). | The parent's value, or a new registered value when the class is a new leaf under an intermediate such as `Works`. |
| Accession | Declared in `accessionRules()`, run by the engine. | Declared in `accessionRules()`, run by the engine. |

## 3. Registry
---

- The registry claims `class` values that no stock class owns.
- A class that extends a stock class keeps its parent's `class` value and claims nothing. Notes are found by frontmatter `class`, not by JavaScript class: an extender ships its own creation command (for example `pulpfiction-create-monograph`, which instantiates a `Pulp-Fiction-Monograph`), so no stock command has to discover the subclass.
- A new class, one that implements the contract or extends an intermediate such as `Works`, registers its new `class` value through Mechanician.
- Registration validates the class's static side with handwritten guards. A `class` value that is already claimed throws.
- The registration mechanism, the relationship to [[Apparatus Module Taxonomy]], and the module a new class belongs to are open (§6).

## 4. Engine
---

### 4.1 Functions

Existing `api.v1` namespaces for schema loading, layout, and resolvers are unchanged ([[Mechanician Spec]] §6.1).

| Member | Kind | Purpose | Rationale |
| --- | --- | --- | --- |
| `createNote(instance)` | function | Creates a note from any instance satisfying the instance contract. Plans the class's Accession rules, applies the write-time sequence of [[Mechanician Spec]] §4.4, writes the file (a path collision throws), then applies the planned pushes. | — |
| `runAccession(source, rules)` | function | Runs a rule list for a created note. Behavior is specified in [[Accession Spec v2]]. | Exposed directly so any code can run rules wherever it sets them up. |

### 4.2 Accession entry points

- Stock and extending classes: `BaseClass.create()` plans the class's rules before the file is written, and `BaseClass.postCreate()` applies them. How the plan reaches the apply step is open (§6.1).
- Implementers: `createNote()` plans, writes, and applies.
- Any code: `runAccession()`, which plans and applies for a note that already exists.

### 4.3 Errors

- The engine throws, with messages written for the user. The command's error boundary reports once through `reportError` ([[Mechanician Spec]] §7).
- A lenient skip is a non-fatal condition: one `Notice`, with `console.warn` when the detail helps ([[Mechanician Spec]] §7).
- Audit and replay of Accession are planned ([[Accession Spec v2]] §7).

## 5. Stock
---

### 5.1 Stock classes

- A stock class is any class that ships inside Mechanician: the shared bases (`BaseClass`, `BaseModal`), intermediates such as `Works`, and leaves such as `Scene` and `Monograph`. External code built on the contract is not stock.
- Stock classes implement the contract and extend `BaseClass`. The class vocabulary by module is in [[Apparatus Module Taxonomy]]; the classes the API serves are in [[Mechanician Spec]] §6.1.
- `BaseClass` returns an empty rule list by default. Its `create()` plans the class's rules and its `postCreate()` applies them. An override of `postCreate()` calls `super.postCreate()` so the pushes keep running.

### 5.2 Library rules

All rules are strict. The last column maps each rule to [[Library Accession Trigger Map]] §3.

| Declared on | `from` | `through` | `targetField` | `targetClass` | `values` | Map row |
| --- | --- | --- | --- | --- | --- | --- |
| `Collection` | `periodical` | — | `collections` | `periodical` | `self` | 1 |
| `Works` | `authors`, `editors`, `translators` | — | `works` | `author` | `self` | 2 |
| `Works` | `authors`, `editors`, `translators` | — | `collections` | `author` | fields: `collections` | 3 |
| `Works` | `collections` | — | `works` | `collection` | `self` | 4 |
| `Works` | `collections` | — | `authors` | `collection` | fields: `authors` | 5 |
| `Works` | `collections` | `periodical` | `authors` | `periodical` | fields: `authors` | 6 |

- `Interview` extends the `from` list of the two `Works` rules that read author fields (map rows 2 and 3) with `interviewer` and `interviewee`.

### 5.3 Entity rules

Declared when `Entity` ships. Strictness follows [[Entity.js Specification]] §5. The last column maps each rule to that section's rows.

| Declared on | `from` | `targetField` | `targetClass` | `values` | `strict` | Entity row |
| --- | --- | --- | --- | --- | --- | --- |
| `Person`, `Organization`, `Belief-System` | `part-of` | `membership` | `organization`, `belief-system` | `self` | true | 1 |
| `Person` | `allies` | `allies` | `person` | `self` | false | 2 |
| `Person` | `rivals` | `rivals` | `person` | `self` | false | 2 |
| `Event` | `participants` | `events` | `person`, `organization` | `self` | false | 3 |
| `Event` | `timeline` | `events` | `timeline` | `self` | false | 4 |
| `Artifact` | `possessor` | `possessions` | `person`, `organization` | `self` | false | 5 |
| `Organization`, `Belief-System` | `founder` | `founded` | `person`, `organization`, `belief-system` | `self` | false | 6 |

### 5.4 Other modules

This document specifies no Accession rules for Creative or Publication.

## 6. Open Items
---

### 6.1 Unresolved

1. **Types package delivery.** How the contract types reach CodeScript Toolkit, and whether it resolves a types package. Spike required: ambient declarations of classes served through `api.v1`, including statics and `abstract`. The package name and how builders install it. How builders get class declarations for the stock classes `api.v1` serves, which are not in the contract package. The workspace split (§1.1) moves the template's configuration files into the plugin package, while §7.1 states they are used as-is apart from the docs loader and the strictness overlay.
2. **Instance contract member list.** Settled against the `BaseClass` specification, which this document has not reviewed.
3. **Names.** The interface names, the engine function names, and the namespace that serves them.
4. **Registry mechanism.** How registration works, its relationship to [[Apparatus Module Taxonomy]] (whether a new class value must be added there), and the module a new class belongs to. [[Mechanician Spec]] §6.1 states Mechanician does not load, register, or run code from outside itself; how class registration relates to that statement is unsettled. Whether an extender registers anything at all is unsettled.
5. **`postCreate()` for implementers.** Whether `createNote()` calls an implementer's optional `postCreate()`, and in what order relative to Accession.
6. **Rule identification.** Whether a rule carries an identifier so a subclass can extend a specific rule without matching on its fields.
7. **Plan handoff.** How a plan made in `BaseClass.create()` before the write reaches `postCreate()` after it.
8. **Migration from [[Mechanician Spec]].** Parked in §7: scope and identity, plugin shape, containers, strictness, dependencies. Not yet carried: module schemas (§4), the typing model (§5.1), commands and the plugin API (§6), error reporting (§7), and that spec's own open items (§9). The source layout (§2.1) is scrapped apart from its docs bundle note, parked in §7.3. Where content lands relative to §2–§5 is undecided. Section references to the old spec in this document are updated as content moves.
9. **Library documentation.** [[Library Accession Trigger Map]], [[Works.js Specification]] §5, and `Library Module` §3 restate the Library rules in prose. Once the rules in §5.2 are the source, those restatements need reconciling.
10. **Taxonomy gap.** `author` and `library` are class values in use and are absent from the class tables of [[Apparatus Module Taxonomy]].
11. **Modal contract.** Undecided. Under consideration: no modal contract, with an implementer building its modal over `BaseModal`'s builders, which makes `BaseModal`'s public surface part of the version contract. To settle before the version freezes: whether builder signatures stay positional or take an options object, since additions to a positional list are additive only as trailing optionals; whether `ControlName` is derived from the builders so the two cannot drift (`buildDateSetting` has no schema control, and it is unclear whether the array-valued `buildSingleSelectSetting` overlaps `dropdown`); whether the schema field renderer, absent from the `core` namespace table of [[Mechanician Spec]] §6.1, is exposed on `api.v1`; how an implementer's modal is held to layout order and the awaited submit; and the load-order effect of every external modal extending `BaseModal` at module load (unverified for CodeScript Toolkit).
12. **Interface granularity.** Proposed: one engine contract (static side and instance side) satisfied by every class, with no per-leaf version of it. The engine calls every class the same way and cannot use leaf-specific members, so a per-leaf contract would repeat the same members once per leaf as a second frozen declaration beside the class. Per-class shapes already exist as options interfaces (`MonographOptions extends WorksOptions`, [[Mechanician Spec]] §5.1).
13. **Shape guarantee for implementers.** The analysis guarantee in §1 covers the generic fields and whatever frontmatter an implementer's own layout produces. Options interfaces describe constructor input at compile time and do not ensure a class's frontmatter carries a given field set: a class extending `WorksOptions` can omit `authors` from its layout. Undecided: whether an implementer can claim a stock shape (for example Works) so queries keyed to that shape apply, at what granularity, and whether the claim is checked at registration against `fieldLayout()` or left to convention.
14. **Accession field reads.** Proposed: rules read their `from` and `values` fields through `fieldValue(key)`, so one read path serves frontmatter composition and Accession.
15. **Shared create pipeline.** Proposed: `BaseClass.create()` and `createNote()` run one pipeline, so a class's notes are identical whether it extends or implements.
16. **Accession-specific items** live in [[Accession Spec v2]] §7.

### 6.2 Porting the `.codex` Accession scripts

Planned. Behavior that differs when `Works.postCreate()` and its siblings become rule declarations run by the engine:

| Behavior | `.codex` today | Engine |
| --- | --- | --- |
| Resolution timing | Authors and collections resolve before any push; the periodical resolves inside the collection loop, after pushes have begun. | Every resolution, including `through`, completes before the note is written. |
| Strict link failure | The note is already written when the failure throws. | No note is written and nothing is pushed. |
| `through` link resolution | Resolved relative to the Work's path. | Resolved relative to the intermediate note's path. |
| Dedupe | Exact string match. | By note basename, comparing existing entries with alias, heading, and path removed. |
| Map rows 5 and 6 value | The Work's `authors` strings as written. | Each author resolved to its note and pushed as `[[basename]]`. |
| Class checks | None. | `targetClass` validated per rule. |
| Failure | The first throw ends the pass. | All pushes are attempted, then one aggregate error. |
| `interviewer`, `interviewee` | Read by `Works` with `?? []`. | Declared by `Interview` (§5.2). |
| Lenient miss (Entity) | Logged with `Log.error`. | Non-fatal `Notice` (§4.3). |

## 7. Needs Sorting
---

Carried from [[Mechanician Spec]] with heading levels and section references adjusted; wording is otherwise unrevised. Where each part belongs is undecided.

### 7.1 Scope, identity, and development kit

Source: [[Mechanician Spec]] §1.

Mechanician builds and runs the LVNACY Apparatus without being it. It is written for the curator's own vaults and is not marketed. External code built on its contracts is covered in §1.

| Concern | Behavior |
| --- | --- |
| Mobile | Commands are registered with `addCommand()` in a plugin with `isDesktopOnly: false`, so they run on Desktop and Mobile. |
| Distribution | One plugin install per vault, updated through the plugin lifecycle. |
| Command location | Each module registers its own commands from its own folder. No central command directory exists. |

**Not goals**

- The plugin does not depend on CodeScript Toolkit for its own commands. It does not remove or forbid CodeScript Toolkit, which compiles external code built on the contracts (§1).
- The plugin does not manage anything outside the objects it owns, and it does not take over the curator's explorer sorting. `Story` and `Manuscript` write a `sorting-spec` property into the notes they scaffold, for the Custom File Explorer sorting plugin to consume; no other `sorting-spec` is read or written.
- The plugin does not migrate existing notes. Frontmatter migration is Archivist's, and body reconciliation is manual (Existing Note Migration Worksheet §3 and §5). Migrating existing notes is a planned future update, preferably before `Entity` ships.

**Identity.** The display name is **Mechanician**: a builder and operator of machines, for a plugin that builds and runs the Apparatus without being it. The id is `mechanician`, final: it cannot change once the plugin is accepted into the community hub, and in practice it cannot change once generated notes exist, because the id is the handle those notes use to reach the plugin's API ([[Mechanician Spec]] §6.1).

**Development kit.** The plugin is built from the curator's plugin template:

- esbuild bundles `main.ts` to CommonJS in `dist/` with `obsidian` external.
- `tsc --noEmit` runs before every production build.
- ESLint and Vitest (against `obsidian-test-mocks`) are configured.
- The code meets the conditions for acceptance into the community plugin hub.
- The template's `package.json`, `tsconfig.json`, `eslint.config.js`, `esbuild.config.js`, and `vitest.config.js` are used as-is, apart from the edits stated in §7.3 and §7.5.
- The `package.json` files inside `.codex/` and `.commands/` carry only `type: module`, are unrelated to the template, and are ignored.

### 7.2 Plugin shape

Source: [[Mechanician Spec]] §2, without §2.1.

- One plugin. Core is always on.
- Library, Creative, and Publication are per-vault toggles. A vault may be story-only, stories plus libraries, or all three; one vault currently uses Publication.
- A vault is a container of modules, not a module. A single vault may hold many stories, libraries, and publications.
- Toggle state lives in the plugin's `data.json`, which is per-vault.
- Each module exposes a `register(plugin)` function that calls `addCommand()` for its commands. The plugin's runtime calls each enabled module's `register`. `main.ts` is lifecycle only.
- All code is free of Node and Electron APIs.

**Rationale for one plugin.** Library and Creative both extend `BaseClass`. Separate plugins would force either duplicate copies of Core (breaking `instanceof` and shared state) or runtime class lookup across plugins (load-order dependence, no static imports).

**Ship order**

1. The first release ships Core, without `Entity`, and Creative.
2. `Entity` has not been generated and ships in a later release.
3. The Library pass waits until `Entity` has been generated. Library is then reviewed, fixed, and shipped in a later release.
4. Publication tooling has not been started.

`Entity` is added to this spec when it ships. The Library specification is drafted as a supplement that adds on to this one.

**First-release commands:** `core-create-image`, `creative-create-story`, `creative-create-scene`, and `creative-create-scene-templates` ([[Mechanician Spec]] §6.2).

**Settings.** One settings tab with a toggle per shipped module, plus the settings named in [[Mechanician Spec]] §4.7 and [[Mechanician Spec]] §6.2. Core has no toggle. Only Creative appears in the first release. A module's commands are visible only while its toggle is on, enforced in each command's `checkCallback`.

**Bundled documentation.** Every vault and module has an `ARCHIVE/`. The documentation bundle gives a user the nomenclature and taxonomy needed to make informed decisions when naming custom category values.

- The bundle is `Apparatus Docs/`. It holds the Module, Status, and Affiliations taxonomy guides and the vault-level schema note.
- When the plugin is enabled, it copies the bundle into the `ARCHIVE/` at the vault root, creating `ARCHIVE/` if it does not exist.
- The guides are refreshed when the plugin updates.
- Class specifications are not part of the automatic copy. A settings button, which the curator presses, copies the class specifications of the shipped modules (Core and Creative in the first release) into `ARCHIVE/Apparatus Docs/`. On a plugin update, class specifications found in that folder are updated; if none are found, nothing is copied.
- Each class specification is imported from beside its class through the same `.md` loader as the guides (§7.3).

### 7.3 Docs bundle build note

Source: [[Mechanician Spec]] §2.1.

- **Docs bundle.** The guides and the vault-level schema note are embedded through esbuild's `text` loader for `.md` files (`loader: { '.md': 'text' }`), with a `declare module '*.md'` type declaration and a small Vite plugin in the Vitest config so tests resolve the same imports. These are the only edits to the template's `esbuild.config.js` and `vitest.config.js`.

### 7.4 Containers

Source: [[Mechanician Spec]] §3.

- A **container** is a folder whose root holds a note with `class: library`, `class: story`, or `class: publication`.
- All module operations are scoped to their container, except the commands that scaffold the container itself.
- Containers nest without limit and across all module types. The container type stays correct at each level: library commands run only inside a library, story commands only inside a story, and no scene is creatable inside a library's `works` folder. A library nested in a story exposes library commands inside the library, while the story's own folder structure continues to default to story commands.
- One folder never holds two container notes. If one does, the operation is cancelled: an error is thrown and surfaced as an Obsidian `Notice`.
- Scaffolding needs only a folder containing a blank note. It is anchored on an active note (for example the README of a git-submodule folder), so a module stays portable as a self-contained folder.

#### Command visibility

| Command kind | Visible when | Examples |
| --- | --- | --- |
| Operational | The active file is inside a container of the matching type. | `library-create-article`, `creative-create-scene`, `creative-create-scene-templates` |
| Scaffold | The active note sits in the folder that will become the container, that folder holds no container note of any type, and the nearest enclosing container is not of the same type. | `library-create-library`, `creative-create-story` |
| Container-free | Always. | `core-create-image` |

- Only the nearest enclosing container decides scaffold visibility. Inside a story, `creative-create-story` is hidden and `library-create-library` is still visible. Inside a library nested in a story nested in a library, `creative-create-story` is visible, because the nearest container is a library.
- A folder that already holds a container note shows no scaffold command of any type, so the plugin cannot create a second container note in a folder.
- The check runs again when the command executes, because the folder may have changed since the palette was drawn.

#### Gate resolver

The gate resolver is a separate function that:

1. Starts at the active file's parent folder and walks toward the vault root, at any depth.
2. At each folder, looks only at that folder's direct children through `metadataCache`.
3. Returns `{ type, note }` for the nearest folder holding a container note of any module class, or `null`.
4. Never throws and never reads file contents.

Operational commands compare the returned `type` to their own. The throwing resolvers remain for use after a command has already passed its gate.

A `checkCallback` can neither throw nor show a `Notice`. So the resolver reports a folder holding two container notes as a conflict result rather than throwing. The command stays visible, and running it throws the one-note error and shows the `Notice`, so the failure is explained instead of silently hiding commands. The scaffold check in [[Mechanician Spec]] §3.1 stops the plugin from creating this state; a second container note added by hand, or a folder merged in from elsewhere, still produces it.

### 7.5 Strictness

Source: [[Mechanician Spec]] §5, the strictness bullets and §5.2.

- Compiler strictness is a **floor**: the template's configuration is the minimum, and the code is held to more where that is reasonable to manage.
- The strictness configuration (`tsconfig.json` and `eslint.config.js`) is not edited; the overlay extends it. The build configuration takes the docs loader edit (§7.3).

#### Strictness floor and local overlay

The template enables `noImplicitAny`, `strictNullChecks`, `strictPropertyInitialization`, `strictFunctionTypes`, `noUnusedLocals`, and `noUnusedParameters`, but not `strict: true`. It lints at `warn` for `no-explicit-any` and `no-non-null-assertion`, has no `no-unsafe-*` rules, and has no `no-floating-promises` or `no-misused-promises`.

- The template's ESLint configuration enforces list style through `@stylistic/list-style` on TypeScript files: lists of more than two items break one per line, and interface bodies are always multi-line. No suitable lint rule for alphabetizing exists, so grouping and alphabetizing declarations, as the Code Style Guide requires, is done by hand.
- The display name "Mechanician" goes in the `brands` option of `obsidianmd/ui/sentence-case` in `eslint.config.js`.
- Flexibility outranks maximum strictness: a flag that makes the codebase hard to evolve stays out of the overlay. Optional function parameters are unaffected by the overlay. `exactOptionalPropertyTypes` is left out, because it would prevent assigning `undefined` explicitly to an optional property.

The overlay is local and leaves the template files untouched:

| Overlay | Contents |
| --- | --- |
| `tsconfig.strict.json` extending `tsconfig.json` | `strict`, `noUncheckedIndexedAccess`, `noImplicitOverride` |
| ESLint overlay | `no-explicit-any` and `no-non-null-assertion` at `error`; the `no-unsafe-assignment`, `-member-access`, `-argument`, `-return`, and `-call` rules; `no-floating-promises`; `no-misused-promises`; `no-restricted-globals` for `Buffer`, `global`, `__dirname`, `__filename`, and `require` in `src/`; `no-restricted-imports` for the import rules in §1.1 |

Consequences of the overlay:

- `noImplicitOverride` requires `override` on every `getBody`, `getFrontmatterFields`, `postCreate`, and `onOpen` override.
- `noUnusedParameters` rejects a base `postCreate(file) {}` with an empty body. No underscore-prefixed identifiers are used anywhere. Privacy is expressed in code with `private`, `protected`, or a non-exported module member, not by naming convention. The empty hook is typed without an unused parameter, or the base implementation uses it.
- `no-misused-promises` requires every `async` call site, including `BaseModal.buildSubmitButton()` calling `onSubmit(this.result)`, to await or otherwise handle the returned promise.
- `noUncheckedIndexedAccess` applies to every indexed access, such as `this.result[resultKey][index]` in `buildNoteListSetting()`.
- `lib` is `ES2020`, so `replaceAll` and `Array.prototype.at` do not typecheck.
- `src/` contains no JavaScript files. The template's ESLint configuration lints the config files, and `tsconfig.json` includes only `**/*.ts`.
- `"types": ["node"]` lets Node globals typecheck. `process.env.*` is compile-time substitution in the template's build (`define` in `esbuild.config.js`). The ESLint overlay restricts Node globals in `src/` so a stray use fails lint.

The overlay is invoked through two scripts beside the template's own. `build` gates on `typecheck` (`tsc`) only, so the strict scripts do not block a release.

`package.json` scripts:

```json
"scripts": {
	"dev": "node esbuild.config.js",
	"typecheck": "tsc --noEmit --skipLibCheck",
	"typecheck:fast": "tsgo --noEmit --skipLibCheck",
	"typecheck:strict": "tsgo --noEmit --skipLibCheck -p tsconfig.strict.json",
	"lint": "eslint .",
	"lint:strict": "eslint -c eslint.strict.config.js",
	"test": "vitest run",
	"test:coverage": "vitest run --coverage",
	"test:ratchet": "pnpm test:coverage && node .coverage/check-coverage-ratchet.js",
	"build": "pnpm typecheck && node esbuild.config.js production",
	"version": "node version-bump.js && git add manifest.json versions.json"
}
```

### 7.6 Dependencies

Source: [[Mechanician Spec]] §8.

The plugin depends on no other plugin and no runtime package. Validation is handwritten, and the `package.json` carries no runtime dependencies.

Generated notes reference other plugins. These are references, not dependencies: without one, the note loses that function and the plugin still works.

| Plugin | Referenced by | Without it |
| --- | --- | --- |
| Dataview (DataviewJS) | Query and dashboard blocks in generated bodies | The blocks do not render. |
| Templater | The scene template command writes `<% tp.date.now() %>` into the generated scene templates when the template engine setting is Templater ([[Mechanician Spec]] §6.2) | The placeholder stays literal, and the curator enters the created date manually. |
| Longform | `Manuscript` writes a `longform` configuration | The property has no effect. |
| Custom File Explorer sorting | `Story` and `Manuscript` write `sorting-spec` | The property has no effect. |
| CodeScript Toolkit | Extension scripts ([[Mechanician Spec]] §6.1). Notes generated before the plugin call `requireAsync`. | Extension scripts do not run. Those older blocks fail. New notes use the plugin API. |

## Appendix: Specification Change Log
---

| Date | Change |
| --- | --- |
| 2026-11-25 | Initial spec. |