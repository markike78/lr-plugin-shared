# Lightroom Classic SDK — Shared Reference

Applies to all Lightroom Classic plugin projects under this directory.

## Runtime environment

Plugins run in **Lua 5.1** — no `goto`, no `//` integer division, no bitwise operators, and **no `\xNN` hex string escapes** (that is Lua 5.2+). Use literal UTF-8 characters in strings (e.g. `—`, `•`, `→`, `…`) or decimal escapes (e.g. `\226\128\148` for —).

Target SDK: **14.0** (latest Lightroom Classic SDK; LR software version is 15.3). Minimum SDK declared in each plugin's `Info.lua`.

## Critical SDK rules

**Async rule:** Any code that reads from the catalog — including `getRawMetadata` — must run inside `LrTasks.startAsyncTask()`. Entry-point scripts (files listed in `LrLibraryMenuItems`) each start their own async task at the top level.

**Write rule:** All `setRawMetadata`, `addKeyword`, and `removeKeyword` calls must be inside `catalog:withWriteAccessDo("undo label", function() ... end)`. `getRawMetadata` cannot be called inside `withWriteAccessDo` (yielding is not allowed in that context) — pre-read any values you need before entering the write block. Always pass a timeout: `catalog:withWriteAccessDo("label", fn, { timeout = 30 })` — without it, any transient LR background write (auto-save, sync) causes an immediate "blocked by another write access call" error.

**pcall + withWriteAccessDo:** Never wrap `withWriteAccessDo` in standard Lua `pcall`. It interferes with LR's write-transaction mechanism and silently prevents catalog writes from committing — operations appear to succeed but nothing changes in the catalog. Repeated failures can also corrupt LR's internal write-lock state, requiring a Lightroom restart.

**Dialog rule:** `LrBinding.makePropertyTable(context)` requires a function context. Always wrap dialog code in `LrFunctionContext.callWithContext("name", function(context) ... end)`.

## Metadata API

**Keywords** cannot be set via `setRawMetadata`. Use `photo:addKeyword(kw)` and `photo:removeKeyword(kw)`. Compare keywords by name (`kw:getName()`) rather than object identity — the SDK may return different Lua wrapper objects for the same catalog keyword across separate calls.

**GPS** is read and written as `{ latitude = number, longitude = number }` via the key `"gps"` in `getRawMetadata` / `setRawMetadata`.

**Color label** is read/written via `getRawMetadata("colorNameForLabel")` — returns `"red"`, `"yellow"`, `"green"`, `"blue"`, `"purple"`, or `"none"`. The key `"label"` is not valid for `getRawMetadata`.

**Rating** — `getRawMetadata("rating")` returns `nil` for unrated photos (not `0`). Use `setRawMetadata("rating", nil)` to clear to unrated; `setRawMetadata("rating", 0)` is invalid.

**Capture time** — `dateTimeOriginal` is read-only; `setRawMetadata("dateTimeOriginal", ...)` throws "unknown metadata key". To copy capture time between photos: read with `getRawMetadata("dateTimeOriginalISO8601")` (returns ISO 8601 string; fall back to `"dateTimeDigitizedISO8601"` or `"dateTimeISO8601"`), then write with `setRawMetadata("dateCreated", isoString)`. This updates the catalog only — the file's EXIF is unchanged until the user saves metadata to file.

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

## Thumbnail API quirks

**`photo:requestJpegThumbnail()` is callback-based and requires busy-wait polling.** The callback sets a flag; the caller must loop with `LrTasks.sleep(0.1)` until the flag is set or a timeout is reached. There is no synchronous variant:

```lua
photo:requestJpegThumbnail(1024, 1024, function(data)
    jpegData = data
    done     = true
end)
local waited = 0
while not done and waited < 600 do   -- 60-second timeout
    LrTasks.sleep(0.1)
    waited = waited + 1
end
```

**The thumbnail renderer times out unpredictably.** Even with a 60-second wait, requests occasionally fail. Wrap in a retry loop (two attempts with a 2-second backoff) as a production safeguard.

**Assigning a child keyword auto-propagates ancestor keywords.** When you call `photo:addKeyword(childKw)`, Lightroom automatically adds all ancestor keywords up the hierarchy. If that is unwanted, explicitly call `photo:removeKeyword(ancestorKw)` afterwards.

## Keyword API quirks

**`kw:getChildren()` returns `nil`, not `{}`** when a keyword has no children. Always guard with `kw:getChildren() or {}`. Same applies to `kw:getSynonyms()`.

**`catalog:createKeyword()` requires its own `withWriteAccessDo`** before the new keyword can be used as a `setParent` target. Creating a keyword and reparenting into it in the same transaction fails — split into two separate write blocks.

**`kw:setAttributes()` is all-or-nothing** — partial updates are not supported. Read the current attributes table, modify the fields you need, then write the whole table back.

**Root keywords have `kw:getParent() == nil`** — there is no special root object. Parentless keywords are the tree roots.

**`kw:getSynonyms()` returns a plain array** — the SDK does not deduplicate or normalize. If you merge synonym lists, deduplicate manually with case-insensitive comparison.

## Dialog and binding quirks

**`LrBinding.negativeOfKey("propName")`** inverts a boolean property for use in `enabled` bindings — useful for disabling a control when a paired checkbox is checked.

**Dynamic dialog rows** use string-indexed properties (`"row_" .. i`) to bind generated controls to a property table. There is no array-binding API; name each property individually.

## HTTP and external APIs

**`LrHttp.post()` signature:**
```lua
local response, headers = LrHttp.post(url, body, headersArray, 'POST', timeoutSecs, maxResponseBytes)
```
Headers are an array of `{ field = "Name", value = "Value" }` tables. Check `headers.status ~= 200` for HTTP errors; a nil response or nil headers indicates a network failure.

**No built-in JSON module.** Bundle a pure-Lua JSON library (e.g. `JSON.lua`) in the plugin folder and `require` it. The SDK does not provide one.

**Base64 encoding for image upload.** The SDK has no built-in base64 encoder. Bundle a pure-Lua `Base64.lua`. To send an image to an API, get JPEG bytes from `photo:requestJpegThumbnail()`, encode with `Base64.encode(jpegData)`, and wrap as a data URI:
```lua
local dataUri = 'data:image/jpeg;base64,' .. Base64.encode(jpegData)
```

**Concurrency with LrTasks.** Spawn parallel API calls via `LrTasks.startAsyncTask`. Manage a concurrency limit with a shared active-count variable and `LrTasks.sleep(0.1)` polling — there is no built-in semaphore or Promise API.

## fal.ai / OpenRouter integration

Documented from the shuttertag project. Use as a template for any future vision-AI feature.

**Endpoint:** `https://fal.run/openrouter/router/vision`

**Authentication:** fal.ai uses `"Key <apiKey>"` in the Authorization header — not `"Bearer"`:
```lua
{ field = 'Authorization', value = 'Key ' .. apiKey }
```

**Request body:**
```lua
JSON.encode({
    model      = modelId,          -- OpenRouter model string
    prompt     = promptString,     -- instruct the model to return JSON only
    image_urls = { dataUri },      -- base64 data URI array
    max_tokens = 4096,
})
```

**Response schema varies by route.** Handle both OpenAI-compatible and fal.ai native formats:
```lua
local content
if parsed.choices and parsed.choices[1] then
    content = parsed.choices[1].message.content   -- OpenAI format
elseif parsed.output then
    content = parsed.output                        -- fal.ai native format
end
```

**The model sometimes wraps JSON in markdown fences.** Strip with three fallback patterns:
```lua
local jsonStr = content:match('```json%s*(.-)%s*```')
             or content:match('```%s*(.-)%s*```')
             or content:match('{.+}')
```

**Known failure mode — Python-syntax response.** Some routes return a Python dict (`{'key': 'value'}`) instead of JSON. Detect and surface a user-friendly error; retrying usually succeeds:
```lua
if jsonStr:match("^%s*{%s*'") then
    return nil, 'Unexpected response format — reprocessing usually resolves this.'
end
```

**Token truncation detection:**
```lua
local finish_reason = parsed.choices and parsed.choices[1] and parsed.choices[1].finish_reason
if finish_reason and finish_reason ~= 'stop' then
    -- response was cut off — reduce max_tokens or switch model
end
```

**Cost extraction.** OpenRouter reports cost inconsistently across response fields — check all locations:
```lua
local cost = tonumber(parsed.cost)
          or (parsed.usage and (tonumber(parsed.usage.cost) or tonumber(parsed.usage.total_cost)))
```

**Force JSON-only output in your prompt.** Vision models often add prose unless explicitly told not to:
```
Analyze this photograph and respond ONLY with a JSON object using exactly these keys: ...
Do not include any text outside the JSON object.
```

**Concurrency recommendation from production use:** serialize thumbnail rendering (1 concurrent — LR queues renders internally) and parallelize API calls (8 concurrent — pure network I/O).

## fal.ai Image Editing Endpoints

Documented from the AI-Edit plugin project. Covers the native fal.ai edit model endpoints (not OpenRouter).

**Endpoints:**
```
https://fal.run/fal-ai/flux-2-pro/edit
https://fal.run/fal-ai/flux-2/edit
https://fal.run/fal-ai/nano-banana-2/edit
```

**Parameter name is `image_urls` (plural array) — not `image_url` singular.** Sending `image_url` returns 422 Unprocessable Entity.

**Base64 data URIs are accepted** in `image_urls` — confirmed via test. No external image hosting required:
```lua
image_urls = { 'data:image/jpeg;base64,' .. Base64.encode(jpegBytes) }
```

**Response shape** (different from OpenRouter vision route):
```lua
-- parsed.images[1].url → temporary fal.media CDN URL (must fetch separately)
-- parsed.seed          → seed used
-- No choices/output wrapping
local resultUrl = parsed.images and parsed.images[1] and parsed.images[1].url
```

**Result image is a CDN URL, not inline bytes.** Must make a second HTTP GET to download the image before saving.

**CDN retention:** 7 days default. Set short expiration with request header to auto-expire after download:
```lua
{ field = 'X-Fal-Object-Lifecycle-Preference', value = '{"expiration_duration_seconds":300}' }
```

**Force-delete API** — call after downloading to remove immediately:
```
DELETE https://api.fal.ai/v1/models/requests/{requestId}/payloads
Authorization: Key YOUR_API_KEY
```
The `requestId` comes from the `x-fal-request-id` response header (not the response body).

**Model pixel limits:** FLUX.2 Pro accepts ≤ 9 MP input (~3000×3000 or 4000×2250). Images must be resized before submission — use `LrExportSession` with `LR_size_maxHeight`/`LR_size_maxWidth` set to target px, not `requestJpegThumbnail`.

**Output resolution ≠ input resolution.** Edit models generate at their own output resolution (controlled by `image_size`/`resolution` params), not the submitted image size. A 4K input does not produce a 4K output.

**Concurrency for image editing:** 2 concurrent max for 2–4K images; 1 for 8K. Edit jobs take 15–60 s each (vs. < 5 s for vision calls). Higher concurrency hits fal.ai queue limits without speed benefit.

**Downloading the result image** — use `LrHttp.post()` with `'GET'` method (LrHttp.get() signature is unreliable for large payloads):
```lua
local data, hdrs = LrHttp.post(resultUrl, '', {}, 'GET', 120, 25 * 1024 * 1024)
```

## Custom metadata

Declared via **`LrMetadataProvider`** in `Info.lua` pointing to a metadata definition file (not `LrMetadataDefinitionFile` — that key is silently ignored and the schema will never register):
```lua
LrMetadataProvider = 'MyMetadata.lua',
```

The definition file returns a table with `schemaVersion` (integer) and `metadataFieldsForPhotos`. Each field uses `title` (not `label`) for the display name. Supported `dataType` values: `"string"`, `"enum"`, `"URL"` only.

**Changing any field property requires two version bumps:** increment the top-level `schemaVersion` AND add/increment a `version` key on the specific field that changed. Missing the field-level `version` causes a "could not upgrade catalog" error when re-adding the plugin:
```lua
{ id = 'myField', dataType = 'string', searchable = false, version = 2 }
```

**Searchable string fields are capped at 512 bytes.** Non-searchable fields have no size limit. Use `searchable = false` for any field that may hold long text (prompts, descriptions).

**Read/write custom metadata with `setPropertyForPlugin` / `getPropertyForPlugin`**, not `setRawMetadata`. The first argument must be `_PLUGIN` (the global LrPlugin object LR injects automatically), not a string identifier:
```lua
photo:setPropertyForPlugin(_PLUGIN, 'fieldId', value)        -- correct
photo:setPropertyForPlugin('com.example.myplugin', 'fieldId', value)  -- wrong, silently fails
photo:setRawMetadata('fieldId', value)   -- wrong, unknown metadata key error
```
Both calls must be inside `catalog:withWriteAccessDo`.

Boolean `dataType` is unreliable across SDK versions — use `"string"` with a sentinel value (e.g. `"1"`) instead.
