# Lightroom Classic SDK — Shared Reference

Applies to all Lightroom Classic plugin projects under this directory.

## Runtime environment

Plugins run in **Lua 5.1** — no `goto`, no `//` integer division, no bitwise operators, and **no `\xNN` hex string escapes** (that is Lua 5.2+). Use literal UTF-8 characters in strings (e.g. `—`, `•`, `→`, `…`) or decimal escapes (e.g. `\226\128\148` for —).

Target SDK: **15.3** (Lightroom Classic, current as of 2026). Minimum SDK declared in each plugin's `Info.lua`.

## Critical SDK rules

**Async rule:** Any code that reads from the catalog — including `getRawMetadata` — must run inside `LrTasks.startAsyncTask()`. Entry-point scripts (files listed in `LrLibraryMenuItems`) each start their own async task at the top level.

**Write rule:** All `setRawMetadata`, `addKeyword`, and `removeKeyword` calls must be inside `catalog:withWriteAccessDo("undo label", function() ... end)`. `getRawMetadata` cannot be called inside `withWriteAccessDo` (yielding is not allowed in that context) — pre-read any values you need before entering the write block.

**Dialog rule:** `LrBinding.makePropertyTable(context)` requires a function context. Always wrap dialog code in `LrFunctionContext.callWithContext("name", function(context) ... end)`.

## Metadata API

**Keywords** cannot be set via `setRawMetadata`. Use `photo:addKeyword(kw)` and `photo:removeKeyword(kw)`. Compare keywords by name (`kw:getName()`) rather than object identity — the SDK may return different Lua wrapper objects for the same catalog keyword across separate calls.

**GPS** is read and written as `{ latitude = number, longitude = number }` via the key `"gps"` in `getRawMetadata` / `setRawMetadata`.

**Color label** is read/written via `getRawMetadata("colorNameForLabel")` — returns `"red"`, `"yellow"`, `"green"`, `"blue"`, `"purple"`, or `"none"`. The key `"label"` is not valid for `getRawMetadata`.

**Rating** — `getRawMetadata("rating")` returns `nil` for unrated photos (not `0`). Use `setRawMetadata("rating", nil)` to clear to unrated; `setRawMetadata("rating", 0)` is invalid.

**Pick/Reject flags** are per-folder or per-collection context and are not portable metadata. Do not sync them between photos.

**Formatted vs raw metadata:** `getFormattedMetadata` is for display strings (e.g. `"fileName"`, `"title"`, `"caption"`). `getRawMetadata` is for values you intend to write back. Prefer `getRawMetadata` in the diff/write pipeline for consistency.

## Source types

| `source:type()` | Meaning |
|-----------------|---------|
| `"LrFolder"` | A folder in the Folders panel |
| `"LrCollection"` | A user-created collection |
| `"LrSmartCollection"` | A smart collection |
| `"LrPublishedCollection"` | A published collection |

Use `catalog:getActiveSources()` to get the currently selected sources. Always validate the source type before proceeding.

## Photo ordering

`catalog:getTargetPhotos()` returns photos in the current **filmstrip display order** when nothing is selected. When photos are selected it returns only the selection. This is the correct API for order-dependent operations.

`source:getPhotos()` returns photos in internal catalog order, which does **not** match the displayed sort order. Do not use it when filmstrip order matters.

## Stack metadata (folder stacks)

| Key | Meaning |
|-----|---------|
| `stackPositionInFolder` | `1` = stack top; `2`, `3`, … = child position |
| `stackInFolderMembers` | Array of all photos in the stack including the top |

These keys apply to **folder stacks only**. Collection stacks have no equivalent SDK keys.

## Development workflow

1. In Lightroom Classic: **Plug-in Manager → Add** → point at the `.lrdevplugin` folder.
2. Edit any `.lua` file.
3. In Plug-in Manager: click **Reload Plug-in** (or remove and re-add).

Lua syntax errors surface as a red error banner in the Plug-in Manager.

Log file locations:
- macOS: `~/Library/Logs/Adobe/Lightroom/LrClassicCC.log`
- Windows: `%APPDATA%\Adobe\Lightroom\Logs\`

There is no automated test runner. All testing is manual inside a live LR catalog.

## Keyword API quirks

**`kw:getChildren()` returns `nil`, not `{}`** when a keyword has no children. Always guard with `kw:getChildren() or {}`. Same applies to `kw:getSynonyms()`.

**`catalog:createKeyword()` requires its own `withWriteAccessDo`** before the new keyword can be used as a `setParent` target. Creating a keyword and reparenting into it in the same transaction fails — split into two separate write blocks.

**`kw:setAttributes()` is all-or-nothing** — partial updates are not supported. Read the current attributes table, modify the fields you need, then write the whole table back.

**Root keywords have `kw:getParent() == nil`** — there is no special root object. Parentless keywords are the tree roots.

**`kw:getSynonyms()` returns a plain array** — the SDK does not deduplicate or normalize. If you merge synonym lists, deduplicate manually with case-insensitive comparison.

## Dialog and binding quirks

**`LrBinding.negativeOfKey("propName")`** inverts a boolean property for use in `enabled` bindings — useful for disabling a control when a paired checkbox is checked.

**Dynamic dialog rows** use string-indexed properties (`"row_" .. i`) to bind generated controls to a property table. There is no array-binding API; name each property individually.

## Custom metadata

Declared via `LrMetadataDefinition` in `Info.lua` pointing to a metadata definition file. Fields are plugin-namespaced (e.g. `com.example.myplugin.fieldId`). Boolean `dataType` is unreliable across SDK versions — use `"string"` with a sentinel value (e.g. `"1"`) instead.
