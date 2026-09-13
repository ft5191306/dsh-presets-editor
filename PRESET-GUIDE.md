# This preset — what it is, and how to change it

Everything needed to understand, measure, and modify this preset's prompt surface, without
rediscovering it. Every mechanism below was verified against a live process, not inferred.

**Leading words.** *Surface* — everything the model reads each turn: tool descriptions plus prompt
sections. *Overlay* — `overlay.mjs`, the one row that rewrites the surface at the assembly seam.
*Home* — the one place a rule lives. *Probe* — a throwaway preset copy you measure instead of
gambling a session on.

| You want to | Go to |
|---|---|
| Understand the mechanism before touching anything | [§1](#1-the-mechanism) |
| Change a piece of prompt text | [§3](#3-the-change-loop) |
| A measurement you can trust | [§5](#5-numbers-and-their-units) |
| Diagnose "I edited it and nothing happened" | [§2](#2-traps-each-one-actually-hit) |
| Know where a rule's other home is before deleting one | [§4](#4-one-rule-one-home) |
| Understand the identity lines | [§6](#6-identity-where-the-you-comes-from) |
| Know why a tool cannot be removed alone | [§7](#7-removing-a-tool-can-remove-another) |

The prompt text itself lives only in `overlay.mjs`. This document explains it; it does not contain it.

---

## 1. The mechanism

### 1.1 Two planes, and only one of them is yours

The host composition (`base.cordis.yml` + `web.cordis.yml`) owns the registries, the sandbox and
approval stack, persistence, and the model route. It is **shipped and read-only** — an upgrade
overwrites it, and corrupting it disables preset authoring itself.

This preset is an **agent-plane** composition, mounted once per process under a standing scope.
Every session naming the preset joins by scope parentage. A service row here must sit inside a
`cordis:group` carrying an `isolate:` realm; without one it publishes into the root realm, where it
is process-global, and `dsh-agent-presets` rejects the mount for it.

**Copy this preset and edit the copy. Never edit the install**, and never edit the shipped
`standard` / `ptc` / `minimal` / `cordis` presets.

### 1.2 Where prompt text comes from

| Layer | Source | Reachable from this preset? |
|---|---|---|
| Tool description `tool.description` | string constants in `dsh-tool-*` packages | **Yes** — rewritten at the assembly exit |
| Tool parameter `parameters.*.description` | same | **No** — the assembly exit exposes only the top-level `description`, and rewriting it is not enough |
| Section `section.text` | each package's `ctx.systemPrompt.section()` | **Yes** — shadowed or replaced by name (see §1.5) |

A row's `config:` never changes prompt text. No `dsh-tool-*` package exposes a `description` field;
the only config keys that move any text at all are `dsh-tool-pwsh`'s `enableRunInBackground` and
`dsh-tool-workflow`'s `toolName`. Do not go looking for a text setting in `agent.cordis.yml`.

### 1.3 The rewrite point is one waterfall

```
agent starts a turn
  → dsh-system-prompt assemble()
       collect sections / tools / variables
       → waterfall('system-prompt/assemble')   ← the only seam for rewriting text
            overlay.mjs rewrites tools[i].description and sections[i].text here
       → the returned value is authoritative
  → renderPrompt() interpolates {{model}} {{cwd}} …
  → the model reads it
```

`overlay.mjs` is an ordinary preset row hung on that waterfall:

```yaml
- id: prompt-surface-overlay
  name: './overlay.mjs'     # relative: resolved against this preset's directory
```

Two preconditions, each of which fails **silently** if broken:

1. **`export const inject = ['systemPrompt']`** — without it the mount reports OK and not one
   character changes. A mount that reports OK is not evidence that a row ran.
2. **The module imports nothing.** A preset directory has no `node_modules`; the loader resolves a
   leading `.` with `new URL(name, baseUrl)` against the preset directory. Relative names load,
   bare package names do not.

### 1.4 The `system-prompt/assemble` event

`'{this: Scoped<SystemPrompt>}'(assembly, context, next): Promise<PromptAssembly>` — a scope-filtered
waterfall returning `{ sections, contexts, tools: ToolSchema[], variables }`. Mutating
`assembly.tools[i].description` or `assembly.sections[i].text` before `next()` reaches the model.
`sections` come back **pre-interpolation**: an `assemble({ scope })` dump still shows `{{model}}`
where the model sees the resolved name.

### 1.5 The three tables

| Table in `overlay.mjs` | Key | Effect |
|---|---|---|
| `DESCRIPTIONS` | tool name | replaces the tool's top-level description |
| `SECTIONS` | section name registered from the host composition | shadows it |
| `REPLACED` | section name this preset's own rows register | replaces it (`''` empties it) |

**All three are the same lookup over the same merged list**: `applyOverlay` looks each name up in
the *merged* assembly and assigns `section.text = table[name]`. The `SECTIONS` / `REPLACED` split
**records where a section came from; it does not restrict what may be done to it.** A name in the
"wrong" table works identically, and nothing warns you. The split was a real distinction only under
an earlier registration-based implementation, which produced
`prompt section "tool:read" is already registered in this scope`.

`''` in either section table drops the section, because `renderPrompt()` discards sections whose
text is empty. `''` in `DESCRIPTIONS` does **not** drop a tool: empty sections are filtered, tool
schemas pass through untouched — the tool stays listed and callable with an empty description.
Removing a tool is a composition edit (§7).

A key matching no tool or section is a **silent no-op, not an error**. Check every key against the
real catalog; after editing, re-grep the install for `name: "tool:` because `tool-subagent`,
`tool-workflow` and `tool-ralph` register no section under some configs.

### 1.6 Where each section comes from

Checked in the install, not assumed. The third column does not decide reachability — §1.5 does:

| Section | Registered by | At |
|---|---|---|
| `tool:pwsh` `tool:goal` `tool:read` `tool:write` `tool:edit` `tool:glob` `tool:grep` `tool:jobs` `tool:web_search` `tool:web_fetch` `tool:subagent_fork` `tool:ralph` `tool:workflow` | the matching `dsh-tool-*` package | a preset row's own scope |
| `plan:policy` | `dsh-plan-mode` | a preset row's own scope; empty outside plan mode |
| `app:web-surface` | `dsh-web-app` | the global layer |
| `harness:source` | `dsh-app-boot` | the global layer |
| `harness:identity` | `dsh-system-prompt`'s constructor | the global layer |
| `ui:deliverable-file-references` | `dsh-client-ui-deliverables` | the global layer — 299 chars, deliberately never shadowed |
| `context:file-reference` | `dsh-file-reference-local` | **the agent's own scope**, installed once per agent |

`context:file-reference` is named like a context and registered as a **section**, per agent, through
`agent.ctx.inject(['systemPrompt', 'tools'])`. Two consequences: it never appears in a
standing-scope assembly (no agent, no section), and when one *is* present a table entry rewrites it
like any other name. Registering a section through `ctx.systemPrompt.section()` would have been a
shadow fight this preset loses at the agent layer; rewriting the merged text is not.

### 1.7 The code paths worth reading

All under
`C:\Users\Libra\AppData\Roaming\npm\node_modules\@deepseek-ai\dsh\node_modules\@deepseek-ai\`.
Each package's `README.md` is the authoritative contract; `lib/index.js` is the shipped code.
Navigate by function name: `grep -n '<name>' lib/index.js`.

| Where | What it settles |
|---|---|
| `dsh-system-prompt` `assemble()` | The surface is built here: sections merged, tool schemas collected, then the waterfall runs. The returned value is authoritative. |
| `dsh-system-prompt` `SECTION_ORDERS` | Canonical numeric positions. A shadow registers the **number**; `getSectionOrder()` takes a first-party *name* and throws on anything else. |
| `dsh-system-prompt` `renderPrompt()` | Empty sections are dropped when rendering, which makes `''` a legal replacement. |
| `dsh-scope` `chainLayers` / `merge` / `effect` / `scopeOf` / `createScope` | `merge` is the line that makes a nearer scope win a name; read `effect` before assuming where a registration lands. |
| `dsh-agent-presets` `mountPreset()` | Plugs the composition into the standing scope context. Every `ctx.tools` / `ctx.systemPrompt` registration inside files into that agent's layer. |
| `dsh-agent-presets` `ensureStanding()` | A mount is rebuilt on `compositionStamp()` — `{mtimeMs, size}` of the **composition file only**. Editing `overlay.mjs` does not trigger a rebuild. |
| `dsh-agent-presets` `standingKeyFor()` | Returns the standing scope key: how to assemble a preset's surface with no session. Also how to be misled — see traps. |
| `dsh-agent-presets` `inactiveRows()` / `leakedServices()` | What a mount rejects: a row that never activated, a service published into the root realm. |
| `dsh-agent-presets` `copy()` / `remove()` | Host-side preset authoring. **Needs no sandbox escalation** — and is the only way to write a sibling preset directory from inside the workspace sandbox. |
| `cordis-plugin-loader` `import(name, …)` | `name.startsWith('.')` resolves against `this.ctx.baseUrl` and `import()`s the URL. **This is what makes `name: './overlay.mjs'` legal.** |
| `cordis-plugin-loader` `unwrapExports()` | `exports.default ?? exports`; an ESM namespace comes back whole, so **named exports `name` / `inject` / `apply` are the supported shape**. |
| `dsh-persona` | The `persona` row's `prefix` / `suffix`, where the identity sentence lives. |

---

## 2. Traps, each one actually hit

| Trap | Symptom | Fix |
|---|---|---|
| **Missing `export const inject = ['systemPrompt']`** | Mount says OK; text never changes. | Declare it at module top level. **A mount that reports OK is not evidence a row ran.** |
| **Expecting `SECTIONS` and `REPLACED` to behave differently** | Nothing warns you — a name in the "wrong" table works identically. | They are the same lookup over the same merged list. The split records provenance, not permission. |
| **Expecting a tool to vanish when its description becomes `''`** | The tool stays listed and callable, now with an empty description. | Only empty **sections** are dropped. Removing a tool is a composition edit (§7); keying a name no row registers is a silent no-op. |
| **`standingKeyFor(id)` as proof** | Returns OK for a row that never ran; returns the cached module, not the edited one. | Never trust it as evidence. Use the §3 loop. |
| **Loader caches the module by resolved path** | An edit to `overlay.mjs` appears to do nothing, even after a remount. | Mount a **copy in a new directory** — a new path is a new module instance. Restart the process for a real session. |
| **Tail-editing a large table** | A later `edit` matching a repeated block truncates the object; duplicate keys appear and the last one silently wins. | Rewrite the whole module with `write` rather than a third patch. Then validate: `node --check`, and `Object.keys(x).length` against `new Set(...)`.size`. |
| **Claiming a tool exists before checking** | e.g. inventing a feedback tool. | Grep the install first: `name: "tool:` across `node_modules/@deepseek-ai`. |
| **Writing to a sibling preset directory from the shell** | `[sandbox: file access denied under workspace-write mode]` | Use `agentPresets.copy()` / `remove()` — host-side, no escalation. |
| **Touching the shipped preset install** | An upgrade overwrites it; corrupting `cordis` disables preset authoring itself. | Copy and edit the copy. |
| **`import()` inside a dynamic Cordis plugin** | `A dynamic import callback was not specified` | `import` is unavailable in the plugin sandbox. Parse text with `fs`, or do it in `pwsh`. |
| **Expecting a tool description to come from row config** | Editing `agent.cordis.yml` `config:` never changes prompt text. | Only the assembly exit changes text (§1.2). |
| **Measuring with `tools.schemas()` / `tools.get()`** | Under-reports. | `assemble({ scope })` is the only trustworthy measurement (§5). |
| **Measuring with the probe installed** | That side is inflated by one tool and 60–100 characters. | Always subtract the probe, and always say which unit you are quoting (§5). |

---

## 3. The change loop

Run the whole loop from a **dynamic Cordis Host plugin**. The copy must be host-side, so the roster
service is the tool, not the shell.

```js
return {
  inject: ['agentPresets', 'systemPrompt'],
  apply(ctx) {
    harness.registerTool(ctx, harness.defineTool({
      name: 'preset_verify',
      description: 'Copy a preset to a fresh id, mount it, and measure its assembly.',
      parameters: {
        from: { type: 'string' }, to: { type: 'string' },
        remove: { type: 'string' }, sample: { type: 'number' },
      },
      output: { schema: { type: 'string' }, render: (_a, v) => [{ type: 'text', text: v }] },
      async execute(args) {
        const lines = []
        if (args.remove) await ctx.agentPresets.remove(String(args.remove))   // clean up first
        if (args.from) await ctx.agentPresets.copy(String(args.from), String(args.to))

        // mount-validate: rejects an unresolvable row, bad config, inert row, leaked service
        const scope = await ctx.agentPresets.standingKeyFor(String(args.to))
        lines.push('mounted OK')

        // the authoritative surface, from the preset's standing scope
        const a = await ctx.systemPrompt.assemble({ scope })
        let tc = 0, sc = 0
        for (const t of a.tools) tc += (t.description || '').length
        for (const s of a.sections) sc += (s.text || '').length
        lines.push('tools=' + a.tools.length + ' toolDescChars=' + tc +
                   ' sections=' + a.sections.length + ' sectionChars=' + sc)
        for (const s of a.sections) lines.push('  § ' + s.name + ' len=' + (s.text || '').length)
        for (const t of a.tools) lines.push('  t ' + t.name + ' len=' + (t.description || '').length)
        return lines.join('\n')
      },
    }))
  },
}
```

Then, in order:

1. **Decide the lever by where the current text lives.** Not a preference:
   a tool's `description` → `DESCRIPTIONS`; a section registered from the host composition →
   `SECTIONS`; a section this preset's own rows register → `REPLACED` (`''` drops it); the `persona`
   or `plan-mode` row's own config → edit `agent.cordis.yml` directly.
2. **Edit `overlay.mjs`** (whole-file `write`, per the tail-edit trap).
3. **Static validation** — all three, every time:
   - `node --check overlay.mjs`
   - `Object.keys(x).length - new Set(Object.keys(x)).size` must be `0` for each table
   - every key against the real tool/section names: a key matching nothing is a no-op, not an error
4. **Probe it**: `copy` → `standingKeyFor` (mount-validate) → `assemble({ scope })` (measure) →
   `remove`. **A fresh `to` id every attempt** — reusing one can hand you the cached module.
5. **Remove the copy and delete the probe plugin** (`cordis_undefine`). A probe is not a
   capability. A failed mount can still leave the copy directory behind; remove it regardless.
6. **Restart the process**, then open a new session. A running session keeps reading the old text.

**Done means**: the probe reported the numbers you expected, the copy is gone from the roster, the
probe plugin is undefined, and a new session shows the new text.

Inside a dynamic plugin, `ctx.scope` is withheld and `import()` is unavailable; reach services with
`inject` / `ctx.get`.

### What an entry is allowed to cost

The surface is written once and read every turn, so the bar is whether the sentence changes what the
model does. The levers that paid, in order:

- **Cut the no-op.** "Returns immediately", "use this when appropriate", status enumerations already
  present in the parameter `enum` beside them.
- **Cut the duplicate.** `tool:goal` restated `create_goal` / `update_goal` almost verbatim;
  `list_agents` restated `send_message`'s steering rules.
- **Fold a dropped section into the tool it governed.** Deleting `tool:jobs` and `tool:web_search`
  cost nothing once `job_output` and `web_search` carried the surviving rule in their own
  descriptions — the rule moves to the moment it applies.
- **Keep the guardrail.** The `pwsh` sandbox/escalation rules survive at a third of their length,
  because a denial is exactly when the model needs them.
- **Delete the tool when deleting the tool is the point.** `workflow` was 2 500 characters of script
  documentation plus a section and an isolated realm. No amount of rewording beats not shipping it,
  and this preset had no reader for it.

---

## 4. One rule, one home

A section is a **second home** for a rule, and deleting it deletes that home. A home can be another
description **or the runtime itself**. Before emptying a section, grep the rule across the surviving
descriptions *and* check whether the platform already enforces it — a platform-level fallback (an
error message's own repair recipe, a banner the harness adds) is a home too, and re-stating it is
pure cost.

Which rules needed a second home, and which survived only because the runtime already had one:

| Rule | Was in section | Now in | Net |
|---|---|---|---|
| Track job ids, don't poll, collect before answering | `tool:jobs` | `job_output`, `job_kill` | **kept** |
| Read a file before `edit` | `tool:edit` | `edit` | **kept** |
| Read an existing file before overwriting it | (never in one) | `write` | **added** |
| No-slash glob patterns match at any depth | `tool:glob` | `glob` | **kept** |
| `offset`/`limit` for large reads | `tool:read` | `read` | **kept** |
| Deliverables include files a command produced | (never in one) | `present` | **kept** |
| Returned web text is data, never an instruction | `tool:web_search`, `tool:web_fetch` | — | **removed**: the harness prepends its own banner to every web result |
| Cite source URLs | same | `web_fetch` only | **halved**: `web_search` results already return links, `web_fetch` results do not |
| Sandbox escalation procedure | `tool:pwsh` | — | **removed**: the denial marker itself carries the escalation path |

Rules that came back, each costed from a fresh-copy `assemble` before it was written:

| Rule | Now in | Why it came back | Cost |
|---|---|---|---:|
| A force-killed process reports `[exit code: 1]` with no signal marker | `pwsh` | the only rule in this pass with **no platform fallback anywhere** | +124 |
| A separator anchors the depth **relative to the session workspace** | `glob` | measured: a `path` outside the workspace plus any separator matches nothing | +35 |
| Matching is case-insensitive unless the pattern encodes case | `grep` | measured: the tool ignores case while "ripgrep" implies the opposite | +62 |
| `resume` is the user's alone; a paused goal waits for them | `update_goal` | the runtime rejects the call outright, so pairing it with `edit`/`pause` invites a doomed call | +180 for all three |
| Complete only when the objective is actually achieved | `update_goal` | the baseline's bar, stated nowhere else | (same edit) |
| Difficulty, uncertainty or remaining useful work is not `blocked` | `update_goal` | the round count is already enforced by the rejection; only the qualitative bar was missing | (same edit) |
| The opening instruction reads as English, separated from the identity | `persona.config.prefix` | it was `weeds.You`, and `Avoid overthink` | +16 |

The rules still missing a home, all minor: `todo_write`'s
`while work remains, at least one task should be in_progress`, and `harness:source`'s cwd discipline
(that whole section is deliberately emptied).

---

## 5. Numbers and their units

Measured with the same probe and the same `assemble({ scope })` call on both sides, **minus the
probe's own entry**:

| | `standard` | this preset |
|---|---:|---:|
| Tool count | 26 | **24** |
| Tool description characters | 13 878 | **7 454** |
| Sections | 20 | 18 |
| Non-empty section characters | 6 278 | **1 169** |
| **Live total** | **20 156** | **8 623 (42.8 %)** |

**Always subtract the probe, and always say which unit.** The probe is itself a tool: it adds one
tool and 60–100 characters to whichever scope you assemble. An earlier revision of this material
counted it on the `standard` side (13 962 / 20 240) and then put a *recomputable-from-`overlay.mjs`*
figure in the same column as a *live dump*.

⚠️ **Two units, never mix them.** What `overlay.mjs` alone accounts for is `DESCRIPTIONS` **7 454**
plus the sections it controls (563 + 105 = **668**) = **8 122**. The live dump is **501** characters
above that — exactly the three non-empty sections it does not own:
`ui:deliverable-file-references` (299), `deployment:persona-prefix` (168),
`deployment:persona-suffix` (34). The bad `38 %` figure came from mixing the units and using a
probe-inflated baseline as the denominator. When you quote a number, say whether it is
*recomputable* or *dumped*, and whether the probe was subtracted.

What the 1 169 characters are made of — only two of the five are under `overlay.mjs`'s control:

| Section | Chars | Registered by | Changed by |
|---|---:|---|---|
| `app:web-surface` | 563 | global (`dsh-web-app`) | `SECTIONS`, already shadowed |
| `ui:deliverable-file-references` | 299 | global (`dsh-client-ui-deliverables`) | **shadowable, deliberately not shadowed** — it is what makes produced files clickable in the Web UI, so it is behavioural, not decoration |
| `deployment:persona-prefix` | 168 | this preset's own `persona` row | edit `agent.cordis.yml` (§6) |
| `tool:goal` | 105 | this preset's own `tool-goal` row | `REPLACED` |
| `deployment:persona-suffix` | 34 | this preset's own `persona` row | edit `agent.cordis.yml` |
| the other 13 sections | 0 | — | emptied, or empty by nature (`plan:policy` only carries text in plan mode) |

The thirteen `len=0` sections: `harness:identity`, `plan:policy`, `tool:pwsh`, `tool:read`,
`tool:write`, `tool:edit`, `tool:glob`, `tool:grep`, `tool:jobs`, `tool:web_search`, `tool:web_fetch`,
`tool:subagent_fork`, `harness:source`. `tool:workflow` and `tool:ralph` are **not** in that list:
with their rows removed, those sections do not exist at all.

Deliberately left alone, and why:

| What | Decision |
|---|---|
| `ui:deliverable-file-references` (299, global, shadowable) | **Keep.** It governs whether produced files are clickable in the Web UI. Behaviour, not decoration. |
| `ptc`'s `tools:sdk` (27 620) | **Do not hand-trim.** It is the SDK documentation the model writes PTC code against, generated by `dsh-tools`; rewriting it freezes a snapshot that will go stale. |
| `tool-web`'s `config.fetch: false` | Note only: it would delete `web_fetch` **and** rewrite `tool:web_search`'s section text. This preset uses `web_fetch`. |
| `REPLACED`'s `tool:workflow` entry | A dead key in this preset. Kept with a comment: workflow and ralph come back together or not at all (§7). |

Writing the surface once and reading it every turn is the whole argument: `standard` is unchanged
from its baseline, and this preset is **42.8 %** of it — 11 533 characters smaller, with 2 tools
fewer (`workflow`, `ralph`), the two identity lines collapsed into one, and the rules above carried
back into the tool descriptions.

---

## 6. Identity: where the "you" comes from

`standard` states it twice, from two planes, which is easy to misread as one bug:

| Line | Section | Registered by | Reachable as |
|---|---|---|---|
| "You are an AI agent powered by DeepSeek Harness." | `harness:identity`, order −1000 | `dsh-system-prompt`'s own constructor, at the **global** layer | **shadowable** — this preset's layer is nearer |
| "You are a coding agent powered by the `<model>` model." | `deployment:persona-prefix`, order 0 | the `persona` row's `config.prefix` | this preset's own registration |

The two differ only in the last two words, so `overlay.mjs` empties `harness:identity` and the
`persona` row carries the facts once: *"You are a coding agent powered by the `{{model}}` model,
running on the DeepSeek Harness."* Note the split output an `assemble({ scope })` gives you: sections
come back **pre-interpolation**, so the persona text still reads `{{model}}` there.
`renderPrompt()` substitutes registered `{{...}}` variables afterwards, and the model sees the
resolved name.

The row carries 168 characters because it also opens with a standing instruction that is not in
`standard`:

```
Avoid overthinking, overplanning, getting clever, or getting stuck in the weeds. You are a coding agent powered by the {{model}} model, running on the DeepSeek Harness.
```

The 80-character first sentence is preset-specific; the 87-character second sentence is the merged
identity this section describes. Two defects this entry used to record are repaired: the sentences
were glued with no space (the model read `weeds.You`), and the first read `Avoid overthink`. The
repair cost +16 characters — an earlier note claimed the grammatical fix was free, which was wrong.

`harness:identity` is 48 characters in `standard` and `0` here, so the identity has exactly one home.

**The third identity line is `context:file-reference`** ("Tokens prefixed with @ are workspace
paths…"), registered as a **section** by `dsh-file-reference-local` once per agent. It is absent from
every standing-scope assembly and present in every real one — §1.6 covers what that means for
reaching it.

---

## 7. Removing a tool can remove another with it

`workflow` and `ralph` are one decision, not two. `@deepseek-ai/dsh-tool-ralph` takes
`workflowEngine` as a **hard** dependency, so deleting only the `tool-workflow` row leaves a mount
that fails the whole preset:

```
1 row(s) did not activate:
tool-ralph (@deepseek-ai/dsh-tool-ralph): waiting for workflowEngine
```

That is the `inactiveRows()` audit doing its job — and it fails the mount, so a probe copy is how you
learn it rather than by breaking a session. Before deleting a tool row, check whether a *sibling* row
injects a service that row provided: grep the sibling's `inject` array, or just mount-validate.
Serving `ralph` alone would need `workflow-worker-thread` back plus its own `isolate.workflowEngine`
realm — rows to keep one tool this preset does not want either.

---

## 8. This preset's files

| File | Holds |
|---|---|
| `agent.cordis.yml` | The composition. `prompt-surface-overlay` (row id) is the one row that changes prompt text: `name: './overlay.mjs'`, nothing else. It is also where the identity sentence and every removed tool row live. |
| `overlay.mjs` | `DESCRIPTIONS` / `SECTIONS` / `REPLACED`, plus `inject` and `apply`. The only place prompt text lives. |
| `preset.yml` | Roster metadata: `name`, `description`, `order`. |
| `PRESET-GUIDE.md` / `PRESET-GUIDE.zh-CN.md` | This document, English and Chinese. |
| `PROMPT-BASELINE.md` | `standard`'s assembled text, frozen. Judge against it, not against memory. |

Documents deleted after the test round: `PROMPT-TESTS.md` (its design is superseded by the report),
`PROMPT-CHECKLIST.md` (its per-test evidence moved into the report), `PROMPT-RUN.md` (the test round
passed), plus the earlier `ARCHITECTURE.md` and `HANDOFF.md` (merged into this file). Do not recreate
them.
