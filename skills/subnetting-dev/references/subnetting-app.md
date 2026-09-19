# subnetting.dev: driving patterns

> The source of truth for the app is `/llms.txt`, fetched from the origin you are driving. This file covers only the patterns for driving the page, independent of transport: JavaScript bodies and intents. The exact tool shape for your transport is in its `transport-*.md` file. This file picks up after the bootstrap in [SKILL.md](../SKILL.md).

## Calling the facade

The facade lives on `window.subnet` and returns a `{ ok, data?, error?, code? }` envelope; it does not throw. If you do see a thrown error, that is a bug in the facade itself rather than in the path you took: surface the message and ask the user.

Some methods are synchronous and some return a promise; the signatures in `/llms.txt` say which. Awaiting a synchronous method is harmless, so the safe shape is always an async function that awaits every call and returns one string:

<!-- prettier-ignore -->
```js
async () => {
  const { data: net } = await window.subnet.getNetwork()
  const r = await window.subnet.split(net.rootBlockId, 4)
  return JSON.stringify(r)
}
```

`window.subnet.help()` already returns a string; everything else goes through `JSON.stringify`.

### Multi-step orchestration

`/llms.txt` -> "Visualization modes" -> Recipes ships executable multi-step recipes per mode, and "Working alongside the user" explains the atomic primitives (`applyTemplate`, `transaction`, clipboard replication). Drop a recipe in as a single async function: await each facade call and return one `JSON.stringify(...)` summary at the end. One round trip per flow is far cheaper than one per call, and intermediate ids stay in page memory where they cannot go stale.

<!-- prettier-ignore -->
```js
async () => {
  const { data: net } = await window.subnet.getNetwork()
  const r = await window.subnet.transaction(async s => {
    const parent = await s.addChild(net.rootBlockId, { type: 'vertical', prefix: 16 })
    const childIds = []
    for (let i = 0; i < 3; i++) {
      const child = await s.addChild(parent.data.id, { type: 'vertical' })
      childIds.push(child.data.id)
    }
    return { parentId: parent.data.id, childIds }
  })
  return JSON.stringify(r)
}
```

Inside `transaction` the callback's `s` throws on a failed call instead of handing back `{ ok: false }`, and the whole callback lands as one undo entry or rolls back; check its `/llms.txt` entry before relying on either.

### Eval pitfalls

- Return the awaited value. A promise that is awaited but not returned, or returned without being awaited, can serialize as `undefined` across the tool boundary.
- `JSON.stringify` every non-string return value. Raw objects and arrays lose structure in most transports' output.
- Keep ids inside the function when you can. An id carried across calls goes stale after a delete, an undo or a paste that renames.

## Verification patterns

After mutating, verify with whichever surface answers the question fastest:

| Question                                                  | Best surface                                             |
| --------------------------------------------------------- | -------------------------------------------------------- |
| Did my mutation succeed?                                  | The facade response itself: `result.ok === true`         |
| Does the resulting tree match what I intended?            | `listBlocks`, or `assertSchema` against the agreed shape |
| Does the canvas look right?                               | A screenshot (text snapshots cannot see SVG)             |
| What changed since my last snapshot?                      | Snapshot again and compare                               |
| What is currently selected in the UI?                     | `getSelectedBlockId`                                     |
| What was the last error notification?                     | A snapshot: toasts render into the live region           |
| Why did a call return `{ ok: false }` with a vague error? | The envelope's `code`, then the page's console messages  |

## Flows the facade does not cover

Most things have a facade method. The exceptions need key presses:

| Flow                                      | Keys                                                               | Why no facade                             |
| ----------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------- |
| Save current network to cloud             | `Ctrl+S`, then snapshot and fill the modal fields                  | Requires user-supplied name / description |
| Browse history modal (visual undo picker) | `y` to open, `j` / `k` to walk, `Enter` to jump, `Escape` to close | `undo` / `redo` step linearly             |
| Focus mode (canvas filtering)             | `i` to focus, `o` to exclude, `Shift+R` to clear                   | View-only, not state                      |
| Triple-tap network reset                  | `Escape` x 3                                                       | Intentional safety gate (Gate 2 applies)  |
| Keyboard help reference                   | `Shift+?` to open, `Escape` to close                               | Reference modal only                      |

When a modal is open, all keyboard shortcuts except `Escape` are blocked, so close it before issuing other shortcuts. The authoritative key list is `/llms.txt` -> "Keyboard shortcuts".

## Categories page bulk operations

The categories page (`/categories`, also nested as `/network/<id>/categories` and `/shared/<slug>/categories`) has a multi-select UI flow that the facade does not replicate (the facade categorizes one subnet at a time). For bulk:

1. Open the categories route and wait for it to settle.
2. Use the keyboard shortcuts documented in `/llms.txt` -> "Keyboard shortcuts" -> "Network actions": `j` / `k` / `h` / `l` to navigate, `Shift+L` to select, `a` / `d` to enter add / remove mode, `1`-`8` to apply a category pattern.

Scripted alternative that never leaves the main app: read `listBlocks`, filter to subnets, loop `setCategories` / `addCategories` inside one async function.

## Canvas debugging

Text snapshots cannot see the D3 canvas. Use:

- `f` (fit canvas), or `fitToView`, to recover from an off-screen state.
- A viewport screenshot; a full-page screenshot to include off-canvas content; an annotated or element-only screenshot when you need spatial reasoning.
- A window at least 1024 px wide. Below that, or on touch devices, the app switches to the compact drawer layout, which changes the DOM you are asserting against. Test both when the task is UI QA.

If the canvas is blank, confirm there are blocks via `listBlocks`. The canvas needs at least the root container; everything else is derived.

## Read-only routes

`/shared/<slug>` opens a published network in read-only mode. Calling any mutating facade method returns `{ ok: false, error: "read-only" }`. Reads, `selectBlock` and `fitToView` still work; `undo` / `redo` are also gated because they replay structural mutations.

## Cloud-sync caveat

The route decides whether a mutation is local or real, on a dev server as much as on production:

- `/` is the scratch network, persisted only to the driving browser's storage. `/shared/<slug>` is read-only.
- `/network/<id>` is a cloud-saved network. When the browser is signed in, mutations sync to the user's account in real time, with no sandbox. Gate 2 in [SKILL.md](../SKILL.md) lists the operations that need a confirmation there.

## Common pitfalls

- Stale block id after `deleteBlock`: the id is gone, so re-list blocks before continuing.
- `split` with a count that is not a power of two fails. See `/llms.txt` -> "Subnetting fundamentals" for the rule and the `addChild` loop fallback.
- Element references from a snapshot invalidate on navigation and on any structural DOM change; snapshot again after opening a page or after an action that changes it.
- `copy`, `pasteAsSibling` and `pasteOverChildren` share a single in-memory clipboard slot. Two interleaved flows overwrite each other.
- `pasteOverChildren` replaces the target's children; `pasteAsSibling` adds next to them. Picking the wrong one is destructive (Gate 2).

For app-rule pitfalls (mandatory category groups on `setCategories`, per-mode child limits, prefix ranges), `/llms.txt` calls them out in each method's reference section.

## Where to look up app rules

| You need...                                                 | Read in `/llms.txt`                         |
| ----------------------------------------------------------- | ------------------------------------------- |
| How to reach the page, the current skill revision           | `## Reaching the page`                      |
| Method signature, purpose, when-to-use, examples            | `## window.subnet facade` -> method section |
| Per-mode rules (AWS, Generic): levels, constraints, recipes | `## Visualization modes` -> mode section    |
| How CIDR sizing works, when to widen, power-of-two rule     | `## Subnetting fundamentals`                |
| Every keyboard shortcut, grouped by category                | `## Keyboard shortcuts`                     |
| ARIA landmarks (`role="tree"`, `data-block-id`, etc.)       | `## DOM landmarks`                          |
| What read-only mode blocks                                  | `## Read-only routes`                       |

If `/llms.txt` is unreachable, stop and tell the user; do not fall back to memorized method names.

For network-design knowledge (which CIDR base to pick, how to size subnets, what to ask before mutating), see [aws-network-design.md](aws-network-design.md).
