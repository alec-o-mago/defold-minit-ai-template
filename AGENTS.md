# Agent Instructions

This repository is a **Defold** game project built from the **Minit Games** template. The project root is the folder containing `game.project`.

## Project map

- **Root config**: `game.project`
- **Game logic**: `main/` - `game.script` (all gameplay code) and `main.collection` (the scene)
- **Platform glue**: `minit_platform/` - audio, layout/input mapping, render script. **Do not edit** unless the task is explicitly about it: it is what makes the game work inside the Minit app.
- **Assets**: `assets/` (fonts, images)
- **Store metadata**: `meta.json` - description, controls, logic, config keys
- **Dependencies (read-only context)**: `.deps/`
- **Build output**: `build/` (generated, ignored by git)
- **Editor internals**: `.internal/` (ignored by git)

Key Defold settings from `game.project`:

- **Bootstrap collection**: `/main/main.collection`
- **Render**: `/minit_platform/minit.render`
- **Display**: `960x1480` - input coordinates are mapped against this size
- **Dependency**: `minit-defold` v1.17.0 (the Minit SDK, fetched by the editor into the Dependencies folder)

**Resource paths in `game.project`**: values like `main_collection`, `render`, `game_binding` use Defold resource identifiers. A trailing `c` suffix denotes compiled resources and is expected - do not treat it as a typo.

## Minit runtime rules

Everything gameplay-related lives in `main/game.script`.

- `init()` must end with `minit.loading_done()` - the game counts as ready only after that call.
- Call `minit.report_result(score, opts)` **exactly once**, when the game clearly ends. Use `flavor_text` for the result screen and `delay` (ms) to let the ending be seen. Pass `user_data` if you need to persist data between runs; read it with `minit.get_user_data()`.
- Call `audio.set_active(true)` on the first user input - the browser only unlocks audio after a tap.
- Call `layout.refresh()` in `update()` and rebuild positions from `layout.width` / `layout.height`. There is **no design resolution**: content is laid out in backbuffer pixels.
- Convert touch input to layout coordinates with `layout.to_world(action.x, action.y)`.
- Config values are declared in `meta.json` under `config`, then read with `minit.get_config_value("key", "")`.

Useful modules: `minit.minit`, `minit_platform.layout`, `minit_platform.audio`. Do not duplicate their logic in game code.

## Include directories

- Use `.deps/` as an include directory for resolving module references and understanding dependency APIs.
- **NEVER modify any files inside `.deps/`** - these are downloaded dependencies provided strictly as read-only context.
- Run the `defold-project-setup` skill after adding, removing, or changing dependency URLs in `game.project`.

## Defold file formats

- **Lua scripts**: `.lua`, `.script`, `.gui_script`, `.render_script`, `.editor_script`.
- **Metadata assets** (Protocol Buffer Text Format): `.collection`, `.go`, `.sprite`, `.tilemap`, `.tilesource`, `.atlas`, `.font`, `.particlefx`, `.sound`, `.label`, `.gui`, `.model`, `.mesh`, `.material`, `.collisionobject`, `.texture_profiles`, `.display_profiles`.
- **Manifests** (YAML): `.appmanifest`, `.manifest` - platform-specific libraries and build flags.
- **Buffers** (JSON): `.buffer` - streams of data (positions, colors, etc.) used as input for Mesh components.
- **Shaders** (GLSL): `.vp`, `.fp`, `.glsl`.
- **Project config** (INI): `game.project`.
- **Properties** (INI): `game.properties`, `ext.properties` - parameters available in `game.project`.
- **2D assets**: `.png`, `.jpg`.
- **3D assets** (GTLF): `.gltf`, `.glb`.
- **Sound assets**: `.ogg`, `.wav`, `.opus` (OPUS requires modification of the appmanifest).

## Editing Defold assets

When creating or editing Defold asset files, use the corresponding `defold-*-editing` skill to get the correct file format and structure. Always load the skill **before** writing or modifying the file:

- `defold-proto-file-editing` - `.collection`, `.go`, `.gui`, `.sprite`, `.atlas`, `.label`, `.font`, etc.
- `defold-scripts-editing` - `.script`, `.gui_script`, `.render_script`, `.editor_script`, `.lua`
- `defold-shaders-editing` - `.vp`, `.fp`, `.glsl`

When creating new screens or popups with `monarch` (only if that dependency is added), load `monarch-screen-setup` first. For performance-critical math with `xmath` (only if that dependency is added), load `xmath-usage` first.

## Code style guidelines

### Lua scripts (.lua, .script, .gui_script, .render_script, .editor_script)

- **Indentation**: 4 spaces. The existing Minit template files use 4 spaces.
- **Naming**: `snake_case` for variables, functions, files, and folders. Keep resource paths absolute (`/assets/...`) where Defold expects them.
- **Comments**:
  - Use **LuaCATS** (`---@...`) annotations for types, module/public API docs.
- **Whitespace**:
  - Empty lines must be truly empty (no spaces/tabs).
  - Avoid trailing whitespace.
- **Defold API**: strictly follow the Defold API - always verify against the official documentation using the `defold-api-fetch` skill. There are no hidden or undocumented APIs - only use functions, messages, and properties that are explicitly described in the docs. For conceptual guidance on how Defold features work (components, physics, rendering, input, etc.), use the `defold-docs-fetch` skill. For practical implementation patterns and sample code, use the `defold-examples-fetch` skill.
- **Defensive checks**: do NOT assume data is missing or constantly re-check field existence in tables. If YOU set a field, it EXISTS. Similarly, do NOT check for standard Lua API availability (e.g., `io` and `io.open` always exist in standard Lua). Avoid unnecessary defensive programming.
- **Paradigm**: do not use metatables or imitate classes. Use functional, data-based structures only.
- **Logging**: use `print()` to look at the game state. Add logs for initialization, important events, and endings.
- **Script instance state**: in `.script`, `.gui_script`, `.render_script` files, store instance-specific state in the `self` table, NOT in local module variables. Local variables at the module level are shared across ALL instances of the script, which causes bugs when multiple instances exist. Use `self.my_variable` instead of `local my_variable`. Not applicable for local functions - keep them local. If you need to call a local function that is defined below, use forward declarations or reorganize the functions.
- **Local functions**: NEVER create local functions inside other functions. Local functions are only allowed at module scope. Anonymous lambda functions (inline callbacks) are acceptable.
- **require**:
  - Always call `require` with parentheses: `require("module")`, NOT `require "module"`.
  - Use dot notation for module paths: `require("minit_platform.layout")`, NOT `require("/minit_platform/layout")`.
  - Module paths are relative to the project root and use dots (`.`) instead of slashes (`/`) as separators.
  - Do NOT use leading slashes in require paths.
- **Hash values**: `hash("...")` can be left inline without premature optimization. It's acceptable to use `message_id == hash("trigger_response")` directly. If you need to reuse a hash value multiple times, declare it as a module-level constant in `UPPER_CASE` format.
- **Constants**: module-level constants can be declared as local variables in `UPPER_CASE` format: `local MAX_HEALTH = 100`.
- **msg.url format**: always remember the format `[socket:][path][#fragment]`:
  - `socket` - collection name (world)
  - `path` - game object instance id (can be relative or global)
  - `fragment` - component id
  - Shorthands: `"."` for current game object, `"#"` for current component
  - Examples: `msg.url("#my_component")`, `msg.url(socket, path, fragment)`, `msg.url(nil, go.get_id("physics"), "collisionobject")`

### `meta.json`

- Keep it valid JSON with `schemaVersion: 1`.
- `config` entries must match the keys read with `minit.get_config_value()` in `main/game.script`.
- `description`, `controls`, and `logic` are player-facing text; keep them accurate when gameplay changes.

### Python

- Write for Python 3.11. Do NOT write code to support earlier versions of Python. Always use modern Python practices appropriate for Python 3.11. Always use full type annotations, generics, and other modern practices.

## Shell

- **Windows**: use PowerShell.
- **Linux**: use bash.
- **macOS**: use zsh.

## Commands

All commands run from the project root (the folder with `game.project`).

- **Build & Run via editor** - use the `defold-project-build` skill. Requires the Defold editor to be running with the project open. Builds the project, returns compilation errors, and launches the game if the build succeeds.
- **Package for Minit upload** - in the Defold editor: `Project > Minit: Package for Upload`. It validates the project and writes `dist/<title>.zip`.

## Validation checklist

- Build via the running editor succeeds (`defold-project-build` skill).
- `main/game.script` ends the round with exactly one `minit.report_result()` call.
- `meta.json` stays valid JSON and its `config` keys match the code.

## Important repo-specific caveats

- **`minit_platform/` is SDK platform code**: change the game in `main/` instead.
- **Git commit messages**: use the following format: `Short description` in English language ONLY.
