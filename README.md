# dpgen

A configuration-driven command-line generator for Minecraft datapacks. It reads
YAML or JSON, validates the input, builds an in-memory file map and writes a
ZIP containing `pack.mcmeta`, functions, tags and supported JSON resources. It
does not install the pack, run Minecraft or verify that the generated content
loads in a particular game version.

> **Safety:** use trusted configuration and an output path in a disposable
> directory. `build` opens its output with `fs.createWriteStream`, which can
> truncate an existing file. The default output name comes from `pack.name`;
> specify `-o` explicitly. Resource paths permit dots and slashes, and the
> schema does not independently reject `..` path segments. Inspect ZIP entries
> before extracting and use extraction tooling that prevents path traversal.
> Generated commands are not sandboxed when Minecraft later executes them.

## Requirements and setup

- Node.js and npm. No Node version or engine range is enforced in
  [`package.json`](package.json); a particular Node release has not been
  certified here.
- Dependencies declared by the project: `ajv`, `archiver` and `js-yaml`.
  [`package-lock.json`](package-lock.json) records a dependency resolution.
- A trusted YAML or JSON configuration. The CLI treats a filename ending in
  `.json` as JSON; other filenames are parsed as YAML.

From a local checkout, review the dependency tree and install it according to
your environment's policy, for example:

```sh
npm ci --ignore-scripts
```

The repository does not declare an npm `bin` entry; invoke the checked-in
entry point with Node:

```sh
# Parses the configuration and constructs the in-memory file map; no ZIP.
node bin/dpgen.js validate examples/example.yaml

# A fresh temporary directory avoids overwriting an existing ZIP.
work_dir="$(mktemp -d)"
node bin/dpgen.js build examples/example.yaml -o "$work_dir/example.zip"
```

`validate` and `build` both call `buildFileMap`. Invalid input causes a
nonzero exit; errors from individual template builders may appear as a stack
trace. `build` defaults to `<pack.name>.zip` (or `datapack.zip` when no name is
available before validation). The code resolves the chosen path relative to
the current working directory. Only `-o <path>` is parsed as an output option.

## Configuration

The root requires `pack` and a non-empty `namespaces` array. Unknown keys are
rejected at the explicitly defined object levels. This is a compact example;
[`examples/example.yaml`](examples/example.yaml) and
[`examples/example-v2.yaml`](examples/example-v2.yaml) provide fuller examples.

```yaml
pack:
  name: test-pack
  description: Example generated pack
  min_format: 107
  max_format: 107
  dependencies:                  # optional dpgen convention, not vanilla enforcement
    - id: another_namespace
      version: "1.0.0"
      required: true

namespaces:
  - id: example
    functions:
      - path: init
        load: true
        commands: ["say loaded"]
      - path: tick
        tick: true
        commands: ["say tick"]
      - path: greet
        macro: true
    advancements:
      - path: root
        title: Example
        icon: minecraft:stick
        reward_function: example:init
    recipes:
      - path: gem
        type: shapeless
        result: minecraft:emerald
        ingredients: [minecraft:diamond]
    loot_tables:
      - path: chest
        kind: chest
        item: minecraft:gold_ingot
        count: 3
    predicates:
      - path: half_chance
        kind: random_chance
        chance: 0.5
    item_modifiers:
      - path: bonus_count
        kind: set_count
        count_min: 1
        count_max: 3
        predicate_ref: example:half_chance
```

### Fields and generated resources

- Namespace IDs match lowercase letters, digits, `_`, `.`, and `-`; resource
  paths additionally permit `/`. Item IDs use a `namespace:path` shape.
- `pack.min_format` and `pack.max_format` accept an integer or a two-number
  `[major, minor]` array; each defaults to `107`. Ordering checks compare the
  **major component only**, so a reversed minor range with equal majors is not
  caught. Legacy `pack_format` and `supported_formats` are not accepted by
  this input schema. Confirm format values and generated syntax against the
  Minecraft release you intend to use; a configured number is not proof of
  compatibility.
- Each function becomes `data/<namespace>/function/<path>.mcfunction`.
  `tick: true` and `load: true` add it to the corresponding
  `data/minecraft/tags/function/` tag. When `macro: true`, the generator emits
  a fixed `$(example)` skeleton rather than the supplied `commands` list.
- Advancements, shaped/shapeless recipes, block/chest/entity loot tables,
  predicates and item modifiers become files in their respective singular
  resource directories. Some template-specific required fields are checked
  after schema validation; see [`templates.js`](templates.js) for defaults and
  exact generated structures.
- Supported predicate kinds are `random_chance`, `entity_properties` and
  `match_tool`; supported modifier kinds are `set_count`, `enchant_randomly`
  and `set_name`. A modifier's `predicate_ref`, when provided, must match a
  predicate defined in the same configuration, including its namespace.
- The builder rejects duplicate namespace IDs and duplicate **function** paths
  within a namespace. It does not perform an equivalent duplicate check for
  every other resource type: repeated generated keys can overwrite earlier
  entries in the in-memory map. The generated dependency function can also
  collide with a user-specified `__dpgen_deps` function.

The validation is structural and partial. It does not query Minecraft's
registry, prove that item IDs exist, check the safety or correctness of
commands, or exercise a pack in game.

## Dependency convention

`pack.dependencies` is consumed by dpgen, not written as a vanilla datapack
dependency declaration. When it is non-empty, the generator creates
`__dpgen_deps.mcfunction` in the first namespace and prepends it to the load
tag. The function creates the `dpgen_loaded` scoreboard objective, marks that
first namespace as loaded, and checks the named dependency scores.

Other packs must independently set the same scoreboard convention; unrelated
packs are not detected. The optional `version` is displayed in a warning but
is **not compared**. A missing `required: true` dependency produces an
additional message saying the pack is disabled, but the generated function
contains no actual disable operation. Treat both required and optional entries
as warnings, not dependency enforcement. Dependency IDs are inserted into
commands, so review them and use only trusted input.

## Layout and maintenance

| Path | Purpose |
| --- | --- |
| [`bin/dpgen.js`](bin/dpgen.js) | CLI argument parsing, config loading and ZIP output. |
| [`schema.js`](schema.js) | AJV schema, defaults and name patterns. |
| [`builder.js`](builder.js) | Cross-field checks, generated file map and tag assembly. |
| [`templates.js`](templates.js) | JSON and function templates, including dependency warnings. |
| [`examples/`](examples/) | Sample YAML inputs. |

The `npm test` script is a placeholder that deliberately exits with an error;
there is no tracked automated test suite in this snapshot. In one isolated
check using Node 20.20.2 and npm 10.8.2, the dependency install completed with
`npm ci --ignore-scripts --no-audit --no-fund`. Both checked-in example
configurations passed `validate`, and building the first example produced a
nine-file ZIP with a valid CRC. The install warned
that transitive `inflight@1.0.6` and `glob@8.1.0` are deprecated; no dependency
security audit or remediation was performed. This check did not run Minecraft
or validate the generated pack in game.

Review and test outputs in a disposable world appropriate to the targeted
Minecraft version before using them elsewhere. No resource-pack generation is
implemented.

## License

The repository contains an [MIT license](LICENSE) attributed to Runtoolkit,
while `package.json` declares `ISC`. These declarations differ; maintainers
should resolve that metadata discrepancy rather than assuming they are the
same license.
