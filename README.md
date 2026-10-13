# Auton Plugin Marketplace

`index.json` is the curated catalog consumed by `auton-knowledge`. Only reviewed entries are listed;
discovery never makes an unreviewed repository installable by itself.

A runtime plugin (cloned from source) has this shape:

```json
{
  "id": "com.example.plugin",
  "name": "Example plugin",
  "description": "One bounded, plain-text description.",
  "repository": "owner/repository",
  "ref": "v1.0.0",
  "tags": ["knowledge"],
  "verified": false
}
```

- `id` must match the runtime manifest exactly.
- A declarative (MCP) plugin released as prebuilt archives uses `"kind": "package"` instead:

  ```json
  {
    "id": "plugin-directory-name",
    "kind": "package",
    "name": "Example",
    "description": "One bounded, plain-text description.",
    "repository": "owner/release-repository",
    "ref": "v1.0.0",
    "assets": [
      { "platform": "darwin-arm64", "name": "example-1.0.0-darwin-arm64.tgz", "bytes": 1234, "sha256": "<64 lowercase hex>" }
    ],
    "tags": [],
    "verified": false
  }
  ```

  `id` is the `name` in the package's `plugin.json`. `ref` is the release tag; each asset is downloaded from
  `https://github.com/<repository>/releases/download/<ref>/<name>`, and the app installs it only if its size and
  SHA-256 match. `platform` is `<process.platform>-<process.arch>`; a computer without a matching asset sees the
  entry but cannot install it.
  To ship a new release, change `ref` (and the asset's `name`, `bytes`, `sha256`). Computers that installed the
  previous release then show **Update**; the new archive is verified before the old one is replaced, and the
  plugin keeps its enabled state.
- `descriptions` is optional: the description in other languages, keyed by language id
  (`"zh"`, `"ja"`, `"ko"`, `"pt-BR"`). The app shows the one for its display language and falls back to
  `description`, which stays the English text. Each text follows the same rules as `description`.
- `repository` is a GitHub `owner/repository` pair, never a URL.
- `ref` is optional. Prefer an immutable release tag; installation records the resolved full commit.
- `verified` is a catalog-owner assertion. Set it to `true` only after source and requested permissions
  have been reviewed; it is not inferred from stars, publisher claims, or GitHub metadata.
- Duplicate IDs or a known field with a malformed value invalidate the complete catalog. The app keeps
  only the last valid in-memory snapshot as an offline fallback.
- The catalog can grow without breaking older apps: a field an app does not know is ignored (never
  trusted), and an entry whose `kind` it does not know is skipped while the rest still show. Apps
  before this rule rejected both; no released app has that behavior.

## Authoring and submission

In Auton Knowledge, open **Settings → Extensions** and use **Create template** with an empty directory.
The generated project contains a valid `plugin.json`, a no-permission `main.mjs` command plugin, an optional
`styles.css`, and a short README. Use **Publishing preflight** on that directory before installing or submitting it.
Runtime plugins can persist plugin-scoped JSON state with `this.loadData()` and `this.saveData(value)`. The app
owns the storage path, isolates it by plugin ID, and enforces a 1 MiB serialized snapshot limit.
Commands may declare an optional canonical `hotkey`, such as `mod+k` or `shift+mod+h`. Built-in app
shortcuts take priority, duplicate plugin hotkeys are rejected, and unmodified or shift-only shortcuts do
not fire while the user is typing in an editable control. Disabling or reloading a plugin releases its
shortcut together with the command.
Plugins can contribute compact left or right status items with
`this.addStatusBarItem({ id, side, priority, render })`. Lower priorities render first; equal priorities
are ordered by the namespaced item ID. The DOM `render(container)` callback should return cleanup logic.
The status bar is absent when no plugin contributes an item, and disabling, reloading, or uninstalling a
plugin removes its mounted items immediately.
Plugins can contribute compact top-bar actions with
`this.addRibbonAction({ id, title, icon, priority, onClick })`. Icons are rendered as bounded plain text
or emoji, lower priorities render first, and equal priorities are ordered by namespaced action ID.
Actions are removed with the plugin lifecycle; a stale click can never invoke an unloaded plugin action.
Plugins can register a right-dock panel with `this.addPanel({ id, title, render })` and open or close it
through `this.openPanel(id)` / `this.closePanel(id)`. Panel IDs are plugin-scoped, only one plugin panel
is active at a time, and render cleanup runs on close, disable, reload, or uninstall.
Plugins can decorate Knowledge explorer documents with
`this.registerExplorerDecorator({ id, priority, decorate })`. The callback receives bounded document
metadata and may return a short plain-text icon, badge, semantic tone, or tooltip. Outputs are validated
and merged by priority; callback failures are isolated per document row.
Plugins can contribute a namespaced Settings page with `this.addSettingTab({ id, title, render })`. The
`render(container)` callback uses ordinary DOM APIs and should return a cleanup function; disabling,
reloading, or uninstalling the plugin removes both its navigation entry and mounted resources.
Plugins that declare and receive the explicit `agent.tools` permission can call
`await this.addAgentTool({ name, description, inputSchema, run })`. The tool is registered in the active
Knowledge Agent, always passes through normal tool confirmation, returns bounded text, and is removed
automatically when the plugin unloads.
An optional root-level `styles.css` is mounted only while the plugin is active and is removed on failed
activation, disable, reload, or uninstall. Keep selectors scoped to the plugin's
`data-plugin-setting-tab`, `data-plugin-status-item`, `data-plugin-ribbon-action`, `data-plugin-panel`, or
`data-plugin-explorer-decoration` container. Stylesheets are limited to 4 MiB; an unsafe or unreadable stylesheet
is reported without preventing the JavaScript plugin from loading.
### API reference

Every method below is on `this` inside a class that extends `globalThis.autonKnowledge.Plugin`.
Registrations are removed automatically when the plugin is disabled, reloaded, or uninstalled.

| Method | What it does |
| --- | --- |
| `addCommand({ id, title, description?, hotkey?, callback(args) })` | A command in the slash list; optional `hotkey` such as `mod+k`. |
| `addDocumentAction({ id, title, icon?, priority?, when?(doc), onClick(doc) })` | A button in the open document's toolbar. |
| `addDocumentMenuItem({ id, title, icon?, group?, priority?, when?(doc), onSelect(doc) })` | An entry in the document's context menu. |
| `showNotice({ message, title?, kind?, timeoutMs? })` | A notification; `kind` is `info`, `success`, `warning` or `error`; `timeoutMs` 0 keeps it until dismissed, otherwise 1,000–30,000. |
| `onEvent(name, listener)` | `appearance.theme-changed` `{ theme }`, `appearance.locale-changed` `{ locale }`; with `knowledge.read` also `knowledge.document-opened` `{ path, anchor }` and `knowledge.index-changed` `{ status }`. |
| `addSettingItem(spec)` | A setting on the app's plugin settings page: `{ id, title, description?, group?, priority?, type, defaultValue }` with `type` `boolean`, `text`, `number` (`min?`, `max?`, `step?`, `unit?`) or `select` (`options: [{ value, label }]`). |
| `getSettingValue(id)` / `setSettingValue(id, value)` / `resetSettingValue(id)` / `onSettingValueChange(id, listener)` | Read, write, reset and watch those settings. |
| `addSettingTab`, `addStatusBarItem`, `addRibbonAction`, `addPanel` / `openPanel` / `closePanel`, `registerExplorerDecorator` | UI contributions, described above. |
| `loadData()` / `saveData(value)` | Plugin-scoped JSON state (1 MiB). |
| `addAgentTool(spec)` | Needs `agent.tools`; described above. |
| `createWebView(container)` | Needs `web.browse`; see below. |

`doc` in the document callbacks is `{ path, title, kind, indexed, editable }`.

**Permissions.** A plugin declares what it needs in `plugin.json` `permissions`; nothing is granted until
the user reviews and accepts it in **Settings → Extensions**.

- `knowledge.read` adds `this.app.knowledge.listDocuments()`, `indexStatus()`, `readDocument({ path })`,
  `backlinks({ path })`, `resolveCitation({ raw })` and `search({ query, regex?, caseSensitive?, maxResults? })`.
  They resolve with plain data (`readDocument` gives `{ meta, content, headings, links, citations }`;
  `search` gives `[{ path, line, column, text, matchStart, matchEnd }]`), or with
  `{ _tag: 'Err', error: { code, message } }`, for example `NOT_FOUND` for a missing document.
- `knowledge.propose` adds `this.app.knowledge.propose({ description, changes: [{ path, newContent }] })`.
  Nothing is written until the user approves it in the same review dialog the Agent uses; only one
  proposal can be open at a time. It resolves with `{ outcome, summary, reason?, paths? }` where `outcome`
  is `applied` (with the written `paths`), `rejected` (with the user's `reason`), `timeout`,
  `not-reviewed` (the user never saw it, `reason` says why: another proposal was open, the window
  closed…) or `aborted`. Branch on `outcome`; `summary` is prose whose wording may change. An invalid
  request (for example content identical to the file) resolves with `{ _tag: 'Err', error }` instead.
- `agent.tools` enables `addAgentTool`.
- `web.browse` enables `createWebView(container)`, which returns a handle with `ready`, `state`,
  `navigate(url)`, `back()`, `forward()`, `reload()`, `stop()`, `readPage({ maxChars? })`,
  `readSelection()`, `onState(listener)` and `dispose()`. Each plugin gets its own persistent browsing
  session; only http(s) pages load, links that open a new window open in the same view, and downloads
  and permission prompts are refused.
  Page text is untrusted input: quote it as data, never as instructions.

Runtime plugins currently share the app renderer realm, so ID scoping prevents filesystem collisions but is
not a sandbox between hostile plugins. Install only code you trust.

Auton Knowledge polls installed plugin manifests, runtime entries, and optional stylesheets for local changes, coalesces rapid save
bursts, and reloads the affected plugin automatically. **Reload** in **Settings → Extensions** remains
available for explicit recovery. Both paths invalidate the module URL before reconciling the runtime inventory;
if the edited manifest or entry is no longer valid, the previously active instance is removed fail-closed.

The preflight uses the same runtime manifest rules and package limits as installation. It rejects
symbolic links, non-regular filesystem entries, escaping or missing runtime entries, oversized packages,
and malformed manifests. Passing the preflight does not grant `verified: true`; verification remains a
separate source and permission review by the catalog owner.

To submit a plugin, publish the checked directory in a dedicated GitHub repository, create an immutable
release tag, and open a pull request adding one entry to `index.json`. The entry `id` must exactly match
`plugin.json`.
