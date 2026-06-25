# Build Fixes — TypeScript Errors

The codebase had APIs refactored from positional arguments to named-parameter objects,
but several call sites were never updated. The Docker build caught them all via `tsc`.

---

## Fix 1 — Missing `isShortcutKey` export

**File:** `apps/web/src/actions/keybinding.ts`

**Error:**
```
'"@/actions/keybinding"' has no exported member named 'isShortcutKey'
```

`persistence.ts` imported a runtime type guard that was never written. The type
`ShortcutKey` existed but no function to validate an unknown string against it at runtime.

**Fix:** Added the guard. Checks longer modifier prefixes first (`ctrl+alt+shift` before
`ctrl+alt`) to avoid partial matches, then validates the trailing key against `KEY_SET`.

```ts
export function isShortcutKey(value: string): value is ShortcutKey {
  if (KEY_SET.has(value)) return true;
  for (const modifier of MODIFIER_KEYS) {
    if (value.startsWith(`${modifier}+`) && KEY_SET.has(value.slice(modifier.length + 1)))
      return true;
  }
  return false;
}
```

---

## Fix 2 — Missing `isActionWithOptionalArgs` export

**File:** `apps/web/src/actions/definitions.ts`

**Error:**
```
'"@/actions"' has no exported member named 'isActionWithOptionalArgs'
```

Same pattern — `persistence.ts` imported a guard that didn't exist.
`TActionWithOptionalArgs` excludes actions requiring mandatory args
(`remove-media-asset`, `remove-media-assets`), so the guard reflects that.

**Fix:**

```ts
const ALL_ACTIONS = new Set(Object.keys(ACTIONS));
const MANDATORY_ARG_ACTIONS = new Set(["remove-media-asset", "remove-media-assets"]);

export function isActionWithOptionalArgs(value: string): value is TActionWithOptionalArgs {
  return ALL_ACTIONS.has(value) && !MANDATORY_ARG_ACTIONS.has(value);
}
```

---

## Fix 3 — `IndexedDBAdapter` constructor & `set()` called with positional args

**Files:** `apps/web/src/services/storage/migrations/runner.ts`,
`apps/web/src/services/storage/migrations/v1-to-v2.ts`

**Error:**
```
Expected 1 arguments, but got 3
```

`IndexedDBAdapter` was refactored so its constructor takes a single options object
`{ dbName, storeName, version }` and `set()` takes `{ key, value }`, but 5 call sites
still used the old positional form.

**Fix:**

```ts
// Before
new IndexedDBAdapter("video-editor-projects", "projects", 1)
adapter.set(projectId, result.project)

// After
new IndexedDBAdapter({ dbName: "video-editor-projects", storeName: "projects", version: 1 })
adapter.set({ key: projectId, value: result.project })
```

---

## Fix 4 — `DefinitionRegistry.register()` called with positional args

**File:** `apps/web/src/stickers/providers/index.ts`

**Error:**
```
Expected 1 arguments, but got 2
```

`register()` was refactored to take `{ key, definition }` but the sticker provider
registration still used the old two-argument form.

**Fix:**

```ts
// Before
stickersRegistry.register(provider.id, provider)

// After
stickersRegistry.register({ key: provider.id, definition: provider })
```

---

## Summary

| # | File | What was wrong | Fix |
|---|------|---------------|-----|
| 1 | `actions/keybinding.ts` | `isShortcutKey` not exported | Added runtime type guard |
| 2 | `actions/definitions.ts` | `isActionWithOptionalArgs` not exported | Added runtime type guard |
| 3 | `storage/migrations/runner.ts`, `v1-to-v2.ts` | `IndexedDBAdapter` called with positional args | Switched to `{ dbName, storeName, version }` / `{ key, value }` |
| 4 | `stickers/providers/index.ts` | `register()` called with positional args | Switched to `{ key, definition }` |
