# Plan: Custom `storage_id` config option

## Goal

Allow users to provide a custom function that returns a string used to scope
bookmark storage. This enables jj bookmarks, hg branches, or any arbitrary
scoping without the plugin needing VCS-specific code.

## Design

New config option:

```lua
---@field storage_id? fun(): string|nil
```

- When set, `get_storage_path()` calls `config.storage_id()` and uses the
  returned string as the **entire** storage key (hashed to a filename).
  The plugin does NOT prepend repo root or anything else — the function
  owns the full key.
- If the function returns `nil`, falls back to existing git-based scoping.
- `per_branch_bookmarks = false` still disables all scoping (takes
  precedence).
- Git branch detection remains the default when `storage_id` is not set.

Example user configs:

```lua
-- jj bookmark scoping
require("haunt").setup({
  storage_id = function()
    local root = vim.fn.systemlist("jj root")
    if vim.v.shell_error ~= 0 or not root[1] then return nil end
    local bm = vim.fn.systemlist("jj bookmark list -r @ -T 'name ++ \"\\n\"'")
    if vim.v.shell_error == 0 and bm[1] and bm[1] ~= "" then
      return root[1] .. "|" .. bm[1]
    end
    return root[1] .. "|__default__"
  end,
})

-- scope by environment variable
require("haunt").setup({
  storage_id = function()
    local env = os.getenv("MY_PROJECT_ENV")
    return env and (vim.fn.getcwd() .. "|" .. env) or nil
  end,
})
```

## Changes

### 1. `lua/haunt/config.lua`

- Add `storage_id` to `HauntConfig` type annotation (optional, `fun(): string|nil`).
- Default: `nil` (no change to existing behavior).

### 2. `lua/haunt/persistence.lua` — `get_storage_path()`

After the `per_branch_bookmarks = false` early return (line 167), before the
existing git branch logic:

```lua
if config.storage_id then
  local ok, id = pcall(config.storage_id)
  if ok and id then
    local hash = vim.fn.sha256(id):sub(1, 12)
    return data_dir .. hash .. ".json"
  end
  -- storage_id returned nil or errored — fall through to git
end
```

`pcall` protects against user-provided functions that throw.

The function returns the **full key** — the plugin just hashes it. No
repo root is prepended. This means `storage_id` works correctly even in
non-git repos where `get_git_root()` would fall back to `cwd`.

The existing git branch logic is untouched and serves as the fallback.

### 3. `tests/persistence_spec.lua`

Add tests under a new `describe("storage_id config")` block:

- **custom storage_id produces different path than git default**: set
  `storage_id = function() return "my-scope" end`, assert path differs from
  default.
- **storage_id takes precedence over git branch**: set storage_id, verify
  the path is based on it (not the git branch).
- **storage_id returning nil falls back to git**: set
  `storage_id = function() return nil end`, assert path matches the default
  git-branch path.
- **storage_id that errors falls back to git**: set
  `storage_id = function() error("boom") end`, assert path matches the
  default git-branch path (no crash).
- **per_branch_bookmarks=false still overrides storage_id**: set both,
  assert scoping is disabled.

### 4. Documentation

Update README section on `per_branch_bookmarks` to mention `storage_id` as
the escape hatch for non-git VCS or custom scoping.

## What does NOT change

- Bookmark data model (no branch/scope field added to bookmarks).
- Store, display, API modules — unaffected.
- Default behavior for users who don't set `storage_id`.
- `per_branch_bookmarks = false` behavior.
- Git caching (`get_git_info` with 5s TTL) — still used for the fallback path.

## Open questions

1. **Caching**: The git info is cached with a 5s TTL. Should we cache the
   result of `storage_id()`? Probably yes for the same reason (avoid
   repeated shell-outs). Grapple.nvim uses event-driven invalidation
   (autocmd watchers + optional timer polling + debouncing) rather than TTL,
   which is more correct but more complex. Start simple — no caching on
   `storage_id` — and add it if users report sluggishness. The user's
   function can always do its own memoization internally.
