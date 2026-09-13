# 这个 preset — 它是什么，怎么改

看懂、测量、修改这个 preset 提示词面所需的全部内容，不必重新踩一遍。下面每条机制都在活进程里验证过，
不是推出来的。

**贯穿全文的四个词。** *surface（提示词面）*——模型每轮读到的全部文本：工具描述 + 提示词段落。
*overlay*——`overlay.mjs`，唯一一行在组装出口改写 surface 的行。*home（家）*——一条规则唯一该待的地方。
*probe（探针）*——一份用完即弃的 preset 副本，用它测量，而不是拿真实会话去赌。

| 你想做 | 去哪 |
|---|---|
| 动手前先搞懂机制 | §一 机制 |
| 改某段提示词文本 | §三 改动循环 |
| 要一个可信的测量值 | §五 数字与口径 |
| 排查"我改了但没生效" | §二 踩过的坑 |
| 删段落前先确认规则还有没有第二个家 | §四 一条规则一个家 |
| 搞清身份句 | §六 身份 |
| 搞清为什么工具不能单独删 | §七 删一个工具可能连带删掉另一个 |

提示词文本本身只在 `overlay.mjs` 里。这份文档解释它，不含它。

---

## 一、机制

### 1.1 两个平面，只有一个属于你

宿主组装（`base.cordis.yml` + `web.cordis.yml`）拥有各注册表、沙箱与审批栈、持久化、模型路由。
它是**随包发布、只读的**——升级会覆盖它，弄坏 `cordis` 会连 preset 编写能力一起废掉。

本 preset 是**agent 平面**组装，每进程在 standing scope 下挂载一次；命名它的会话按 scope 父子关系加入。
这里的服务行必须放在带 `isolate:` realm 的 `cordis:group` 里：否则它会发布进 root realm，
在那里就是进程全局的，`dsh-agent-presets` 会因此拒绝整个挂载。

**复制本 preset、改副本。不要改安装目录**，也不要去改随包发布的 `standard` / `ptc` / `minimal` / `cordis`。

### 1.2 文本从哪来

| 层 | 来源 | 本 preset 能改吗 |
|---|---|---|
| 工具描述 `tool.description` | `dsh-tool-*` 包里的字符串常量 | **能**——在组装出口改写 |
| 工具参数描述 `parameters.*.description` | 同上 | **不能**——出口只暴露顶层 `description`，改写它也没用 |
| 段落 `section.text` | 各包的 `ctx.systemPrompt.section()` 注册 | **能**——按名遮蔽或替换（§1.5） |

行的 `config:` 永远改不了提示词文本。没有任何 `dsh-tool-*` 包暴露 `description` 字段；
唯一能动一点文本的 config 键只有 `dsh-tool-pwsh` 的 `enableRunInBackground` 和
`dsh-tool-workflow` 的 `toolName`。别去 `agent.cordis.yml` 里找文本配置项。

### 1.3 改写点是一个 waterfall

```
agent 发起一轮
  → dsh-system-prompt assemble()
       收集 sections / tools / variables
       → waterfall('system-prompt/assemble')   ← 改写文本的唯一接缝
            overlay.mjs 在这里改写 tools[i].description 与 sections[i].text
       → 返回值即权威
  → renderPrompt() 插值 {{model}} {{cwd}} …
  → 模型看到
```

`overlay.mjs` 就是挂在这个 waterfall 上的一行普通 preset 行：

```yaml
- id: prompt-surface-overlay
  name: './overlay.mjs'     # 相对路径：按 preset 目录解析
```

两个前提，缺一个就**静默失效**：

1. **`export const inject = ['systemPrompt']`**——少了它，挂载**照报 OK**，一个字都不改。
   挂载报 OK 不是"行生效了"的证据。
2. **模块不 import 任何东西。** preset 目录没有 `node_modules`；loader 对以 `.` 开头的名字用
   `new URL(name, baseUrl)` 按 preset 目录解析。相对名能加载，裸包名不能。

### 1.4 `system-prompt/assemble` 事件

`'{this: Scoped<SystemPrompt>}'(assembly, context, next): Promise<PromptAssembly>`——按 scope 过滤的
waterfall，返回 `{ sections, contexts, tools: ToolSchema[], variables }`。在 `next()` 之前改
`assembly.tools[i].description` 或 `assembly.sections[i].text` 能到达模型。
`sections` 返回时是**未插值**的：`assemble({ scope })` dump 里仍写着 `{{model}}`，而模型看到的是解析后的名字。

### 1.5 三张表

| `overlay.mjs` 里的表 | 键 | 作用 |
|---|---|---|
| `DESCRIPTIONS` | 工具名 | 换掉工具的顶层描述 |
| `SECTIONS` | 全局层注册的段落名 | 遮蔽 |
| `REPLACED` | 本 preset 自有行注册的段落名 | 替换（`''` = 清空） |

**三张表是同一个按名查表改写**：`applyOverlay` 在*合并后*的列表里查每个名字，赋
`section.text = table[name]`。`SECTIONS` / `REPLACED` 的拆分**只记录段落来自哪个平面，不限制能对它做什么**。
把名字放"错"表里行为完全一样，且不会有任何提示。这个拆分只在旧的"注册式"实现下才是真区别，
那种实现会抛 `prompt section "tool:read" is already registered in this scope`。

两张段落表里的 `''` 会丢掉段落，因为 `renderPrompt()` 丢弃空文本段落。`DESCRIPTIONS` 里的 `''`
**不会**删工具：空段落被过滤，工具 schema 原样通过——工具仍在列表里、仍可调用，只是描述为空。
删工具是组装编辑（§7）。

键名匹配不到任何工具或段落时是**静默 no-op，不报错**。每个键都要对着真实名单核；改完还要重新 grep
安装目录里的 `name: "tool:`，因为 `tool-subagent`、`tool-workflow`、`tool-ralph` 在某些 config 下根本不注册段落。

### 1.6 每段从哪来

以下是在安装目录里查过的，不是假设。第三列**不决定可达性**——决定可达性的是 §1.5：

| 段落 | 注册者 | 注册在 |
|---|---|---|
| `tool:pwsh` `tool:goal` `tool:read` `tool:write` `tool:edit` `tool:glob` `tool:grep` `tool:jobs` `tool:web_search` `tool:web_fetch` `tool:subagent_fork` `tool:ralph` `tool:workflow` | 对应的 `dsh-tool-*` 包 | preset 自己的 scope |
| `plan:policy` | `dsh-plan-mode` | preset 自己的 scope；只在 plan mode 里有文本 |
| `app:web-surface` | `dsh-web-app` | 全局层 |
| `harness:source` | `dsh-app-boot` | 全局层 |
| `harness:identity` | `dsh-system-prompt` 构造函数 | 全局层 |
| `ui:deliverable-file-references` | `dsh-client-ui-deliverables` | 全局层——299 字符，**刻意不遮蔽** |
| `context:file-reference` | `dsh-file-reference-local` | **agent 自己的 scope**，每 agent 装一次 |

`context:file-reference` 是这里的陷阱：它名字像 context，实际是**段落**，每 agent 通过
`agent.ctx.inject(['systemPrompt', 'tools'])` 注册。两个后果：它永远不出现在 standing scope 的组装里
（没有 agent 就没有这段），而当它*确实*出现时，表里的条目会像改任何名字一样改掉它。
若改用 `ctx.systemPrompt.section()` 去注册同名段落，就是一场本 preset 在 agent 层必输的遮蔽战；
改写合并后的文本则不是。

### 1.7 值得读的代码路径

均在
`C:\Users\Libra\AppData\Roaming\npm\node_modules\@deepseek-ai\dsh\node_modules\@deepseek-ai\`。
各包的 `README.md` 是权威契约，`lib/index.js` 是发布代码。按函数名定位：`grep -n '<名字>' lib/index.js`。

| 位置 | 它决定了什么 |
|---|---|
| `dsh-system-prompt` `assemble()` | surface 在这里成型：合并段落、收集工具 schema，然后跑 waterfall。返回值即权威。 |
| `dsh-system-prompt` `SECTION_ORDERS` | 规范的数字位置。遮蔽注册的是**数字**；`getSectionOrder()` 收第一方*名字*，给别的会抛。 |
| `dsh-system-prompt` `renderPrompt()` | 渲染时丢弃空段落，所以 `''` 是合法替换值。 |
| `dsh-scope` `chainLayers` / `merge` / `effect` / `scopeOf` / `createScope` | `merge` 是"越近的 scope 赢同名"那行；在假设注册落到哪里之前先读 `effect`。 |
| `dsh-agent-presets` `mountPreset()` | 把组装插进 standing scope 上下文。里面每个 `ctx.tools` / `ctx.systemPrompt` 注册都落进该 agent 的层。 |
| `dsh-agent-presets` `ensureStanding()` | 挂载按 `compositionStamp()` 重建——即**组装文件本身**的 `{mtimeMs, size}`。改 `overlay.mjs` 不触发重建。 |
| `dsh-agent-presets` `standingKeyFor()` | 返回 standing scope key：无会话时怎么组装某个 preset 的 surface。也是怎么被误导——见坑表。 |
| `dsh-agent-presets` `inactiveRows()` / `leakedServices()` | 挂载会拒绝什么：未激活的行、发布进 root realm 的服务。 |
| `dsh-agent-presets` `copy()` / `remove()` | 宿主侧编写入口。**不需要提权**——也是在 workspace 沙箱内写兄弟 preset 目录的唯一办法。 |
| `cordis-plugin-loader` `import(name, …)` | `name.startsWith('.')` 时按 `this.ctx.baseUrl` 解析并 `import()` URL。**这是 `name: './overlay.mjs'` 合法的原因。** |
| `cordis-plugin-loader` `unwrapExports()` | `exports.default ?? exports`；ESM namespace 整体返回，所以**具名导出 `name` / `inject` / `apply` 是受支持的形状**。 |
| `dsh-persona` | `persona` 行的 `prefix` / `suffix`，身份句在这里。 |

---

## 二、踩过的坑（逐条）

| 坑 | 症状 | 解法 |
|---|---|---|
| **漏了 `export const inject = ['systemPrompt']`** | 挂载报 OK，文本一字不改。 | 在模块顶层声明。**挂载报 OK 不是"行跑过了"的证据。** |
| **以为 `SECTIONS` 和 `REPLACED` 行为不同** | 没有任何提示——名字放"错"表里照样生效。 | 它们是同一个查表改写。拆分记的是来源，不是权限。 |
| **以为描述写成 `''` 工具会消失** | 工具仍在列表里仍可调用，只是描述为空。 | 只有空**段落**会被丢弃。删工具是组装编辑（§7）；键一个没有行注册的名字是静默 no-op。 |
| **拿 `standingKeyFor(id)` 当证据** | 对从未跑过的行也返回 OK；返回的是缓存模块，不是改过的那个。 | 永远不要拿它当证据。用 §3 的循环。 |
| **loader 按解析路径缓存模块** | 改了 `overlay.mjs` 看起来毫无效果，重挂也没用。 | 挂**新目录里的副本**——新路径即新模块实例。真实会话要重启进程。 |
| **对大表做尾部补丁** | 后一次 `edit` 匹配到重复块，把对象截断；出现重复键，最后一个静默胜出。 | 第三次修补就别补了，用 `write` 整文件重写。然后校验：`node --check`，以及 `Object.keys(x).length` 对 `new Set(...)`.size`。 |
| **没查就断言某工具存在** | 比如凭空造一个 feedback 工具。 | 先 grep 安装目录：`node_modules/@deepseek-ai` 下的 `name: "tool:`。 |
| **在 shell 里写兄弟 preset 目录** | `[sandbox: file access denied under workspace-write mode]` | 用 `agentPresets.copy()` / `remove()`——宿主侧，无需提权。 |
| **改随包发布的 preset 安装目录** | 升级覆盖它；弄坏 `cordis` 会连 preset 编写能力一起废掉。 | 复制，改副本。 |
| **在动态 Cordis 插件里用 `import()`** | `A dynamic import callback was not specified` | 插件沙箱没有 `import`。用 `fs` 读文本，或者放到 `pwsh` 里做。 |
| **以为工具描述来自行 config** | 改 `agent.cordis.yml` 的 `config:` 永远改不动文本。 | 只有组装出口能改文本（§1.2）。 |
| **用 `tools.schemas()` / `tools.get()` 量尺寸** | 数值偏低。 | `assemble({ scope })` 是唯一可信的测量（§5）。 |
| **带着探针量尺寸** | 那一侧被抬高 1 个工具 + 60–100 字符。 | 永远扣掉探针，并永远说明引的是哪个口径（§5）。 |

---

## 三、改动循环

整段循环都从一个**动态 Cordis Host 插件**里跑。复制必须是宿主侧的，所以工具是 roster 服务，不是 shell。

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
        if (args.remove) await ctx.agentPresets.remove(String(args.remove))   // 先清理
        if (args.from) await ctx.agentPresets.copy(String(args.from), String(args.to))

        // 挂载校验：拒绝无法解析的行、坏 config、惰性行、泄漏的服务
        const scope = await ctx.agentPresets.standingKeyFor(String(args.to))
        lines.push('mounted OK')

        // 权威 surface，取自该 preset 的 standing scope
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

然后按顺序：

1. **按当前文本的注册位置决定用哪个杠杆。** 这不是偏好：工具描述 → `DESCRIPTIONS`；宿主组装注册的
   段落 → `SECTIONS`；本 preset 自有行注册的段落 → `REPLACED`（`''` 清空）；`persona` / `plan-mode`
   行自己的 config → 直接改 `agent.cordis.yml`。
2. **改 `overlay.mjs`**（按"尾部补丁"那条坑，用 `write` 整文件重写）。
3. **静态校验**——三项每次都做：
   - `node --check overlay.mjs`
   - 每张表 `Object.keys(x).length - new Set(Object.keys(x)).size` 必须为 `0`
   - 每个键对着真实工具/段落名核一遍：匹配不到任何东西的键是 no-op，不是错误
4. **挂探针测**：`copy` → `standingKeyFor`（挂载校验）→ `assemble({ scope })`（量尺寸）→ `remove`。
   **每次都换新的 `to` id**——复用 id 可能拿到缓存的模块。
5. **删掉副本，并删掉探针插件**（`cordis_undefine`）。探针不是能力。挂载失败也可能留下副本目录，
   无论成功与否都要 `remove`。
6. **重启进程**，再开新会话。正在跑的会话仍读旧文本。

**做完的标准**：探针报出的是你预期的数字；副本已从 roster 消失；探针插件已 undefine；新会话里能读到新文本。

在动态插件里 `ctx.scope` 是被收回的，`import()` 也不可用；服务用 `inject` / `ctx.get` 拿。

### 一条条目允许花多少字符

surface 写一次、每轮都读，所以标准只有一条：这句话是否改变模型的行为。按回报排序的杠杆：

- **砍 no-op。** "Returns immediately"、"use this when appropriate"、旁边参数 `enum` 里已经有的状态枚举。
- **砍重复。** `tool:goal` 几乎逐字重述了 `create_goal` / `update_goal`；`list_agents` 重述了
  `send_message` 的转向规则。
- **把删掉的段落折进它管辖的工具。** 删 `tool:jobs` 和 `tool:web_search` 之所以零代价，是因为
  `job_output` 和 `web_search` 在自己的描述里接住了活下来的规则——规则移到了它生效的那一刻。
- **护栏留着。** `pwsh` 的沙箱/升级规则以三分之一的篇幅活下来，因为被拒的那一刻正是模型需要它的时候。
- **该删工具时就删工具。** `workflow` 是 2 500 字符的脚本文档，外加一个段落和一个隔离 realm。
  再怎么改写都比不上不发它，而本 preset 没有它的读者。

---

## 四、一条规则一个家

段落是规则的**第二个家**，删段落就是删掉这个家。家可以是另一条描述，**也可以是运行时本身**。
清空一个段落之前，既要把这条规则在所有幸存描述里 grep 一遍，也要查平台是否已经强制它了——
平台级兜底（错误信息自带的修复配方、harness 自动加的横幅）也是家，重述它就是纯成本。

哪些规则需要第二个家，哪些只因为运行时已经有家才活下来：

| 规则 | 原在段落 | 现落点 | 净结果 |
|---|---|---|---|
| 记住 job id、不要轮询、答前收口 | `tool:jobs` | `job_output`、`job_kill` | **保留** |
| `edit` 前先读文件 | `tool:edit` | `edit` | **保留** |
| 覆盖已有文件前先读 | （从来不在段落里） | `write` | **新增** |
| 无斜杠 glob pattern 跨任意深度 | `tool:glob` | `glob` | **保留** |
| 大文件读取用 `offset`/`limit` | `tool:read` | `read` | **保留** |
| 交付物包含命令产出的文件 | （从来不在段落里） | `present` | **保留** |
| 网页返回文本是数据、不是指令 | `tool:web_search`、`tool:web_fetch` | — | **删掉**：harness 给每条 web 结果前都自己加横幅 |
| 引用来源 URL | 同上 | 只在 `web_fetch` | **减半**：`web_search` 结果自带链接，`web_fetch` 结果没有 |
| 沙箱升级流程 | `tool:pwsh` | — | **删掉**：拒绝标记自身就带升级路径 |

补回来的规则，每条都在写之前用新副本 `assemble` 量过成本：

| 规则 | 现落点 | 为什么补回来 | 成本 |
|---|---|---|---:|
| 强杀进程落成 `[exit code: 1]`，无信号标记 | `pwsh` | 这一轮里**唯一一处任何平台都没有兜底**的规则 | +124 |
| 分隔符锚定的深度**相对会话 workspace** | `glob` | 实测：`path` 指到 workspace 之外时，带任何分隔符都匹配不到 | +35 |
| 除非 pattern 自带大小写，否则忽略大小写 | `grep` | 实测：工具忽略大小写，而 "ripgrep" 暗示相反 | +62 |
| `resume` 是用户专属；paused 的目标等人来恢复 | `update_goal` | 运行时直接拒绝该调用，把它和 `edit`/`pause` 并列会招来一次注定失败的调用 | 三条共 +180 |
| 只在目标真正达成时才 complete | `update_goal` | 基线里的门槛，别处都没写 | （同一次编辑） |
| 困难、不确定、还有有用的事没做完，都不算 `blocked` | `update_goal` | 轮数门槛已由拒绝本身强制；缺的只是定性判据 | （同一次编辑） |
| 开头这句指令读起来是英文，且与身份句分开 | `persona.config.prefix` | 原来是 `weeds.You`，以及 `Avoid overthink` | +16 |

至今仍无家的规则，都比较轻：`todo_write` 的
`while work remains, at least one task should be in_progress`，以及 `harness:source` 里的 cwd 纪律
（那整段是被刻意清空的）。

---

## 五、数字与口径

两侧用同一支探针、同一支 `assemble({ scope })` 测得，**并扣掉探针自己那一条**：

| 项 | `standard` | 本 preset |
|---|---:|---:|
| 工具数 | 26 | **24** |
| 工具描述字符 | 13 878 | **7 454** |
| 段落数 | 20 | 18 |
| 非空段落字符 | 6 278 | **1 169** |
| **合计（dump）** | **20 156** | **8 623（42.8 %）** |

**永远扣掉探针，永远说清口径。** 探针本身是一个工具：它会给你测的那一侧加 1 个工具和 60–100 字符。
本材料早先的版本把它算在了 `standard` 一侧（13 962 / 20 240），又把*可从 `overlay.mjs` 复算*的数字
和*进程内 dump* 放进了同一列。

⚠️ **两种口径，别混。** `overlay.mjs` 能解释的是 `DESCRIPTIONS` **7 454** 加上它管的段落
（563 + 105 = **668**）= **8 122**。真实 dump 比它多 **501**，正好是它管不到的那三段非空段落：
`ui:deliverable-file-references`（299）、`deployment:persona-prefix`（168）、
`deployment:persona-suffix`（34）。那个错的 `38 %` 就是混了口径、又拿含探针的基线当分母算出来的。
引用数字时说清是*可复算*还是*dump*，以及扣没扣探针。

1 169 字符的构成——五段里只有两段归 `overlay.mjs` 管：

| 段落 | 字符 | 谁注册 | 由谁改 |
|---|---:|---|---|
| `app:web-surface` | 563 | 全局（`dsh-web-app`） | `SECTIONS`，已遮蔽 |
| `ui:deliverable-file-references` | 299 | 全局（`dsh-client-ui-deliverables`） | **可遮蔽，决定不遮蔽**——它管产出文件在 Web 里能不能点，是行为不是装饰 |
| `deployment:persona-prefix` | 168 | 本 preset 自己的 `persona` 行 | 改 `agent.cordis.yml`（§6） |
| `tool:goal` | 105 | 本 preset 自己的 `tool-goal` 行 | `REPLACED` |
| `deployment:persona-suffix` | 34 | 本 preset 自己的 `persona` 行 | 改 `agent.cordis.yml` |
| 其余 13 段 | 0 | — | 已清空，或本就为空（`plan:policy` 只在 plan mode 有文本） |

那 13 个 `len=0` 的段落：`harness:identity`、`plan:policy`、`tool:pwsh`、`tool:read`、`tool:write`、
`tool:edit`、`tool:glob`、`tool:grep`、`tool:jobs`、`tool:web_search`、`tool:web_fetch`、
`tool:subagent_fork`、`harness:source`。`tool:workflow` / `tool:ralph` **不**在这个列表里：
它们的行被删了，那两段根本不存在。

明确不动，以及原因：

| 什么 | 决定 |
|---|---|
| `ui:deliverable-file-references`（299，全局，可遮蔽） | **保留**。它决定产出文件在 Web UI 里能不能点。是行为，不是装饰。 |
| `ptc` 的 `tools:sdk`（27 620） | **不手工裁**。那是模型写 PTC 代码所依据的 SDK 文档，由 `dsh-tools` 生成；改写等于冻一份会过期的快照。 |
| `tool-web` 的 `config.fetch: false` | 仅记录：它会删掉 `web_fetch`，**并**改写 `tool:web_search` 的段落文本。本 preset 在用 `web_fetch`。 |
| `REPLACED` 里的 `tool:workflow` 条目 | 在本 preset 是死键。保留并加注释：workflow 和 ralph 要回来是一起回来（§7）。 |

写一次、每轮都读——这就是全部理由：`standard` 与它的基线相比未变，而本 preset 是它的 **42.8 %**：
小 11 533 字符，少 2 个工具（`workflow`、`ralph`），两行身份句合并成一行，上面那些规则搬回了工具描述里。

---

## 六、身份：那句 "you" 从哪来

`standard` 从两个平面各说了一遍，很容易误读成一个 bug：

| 句子 | 段落 | 注册者 | 可达性 |
|---|---|---|---|
| "You are an AI agent powered by DeepSeek Harness." | `harness:identity`，order −1000 | `dsh-system-prompt` 构造函数，**全局**层 | **可遮蔽**——本 preset 的层更近 |
| "You are a coding agent powered by the `<model>` model." | `deployment:persona-prefix`，order 0 | `persona` 行的 `config.prefix` | 本 preset 自己的注册 |

两句只差最后两个词，所以 `overlay.mjs` 清空 `harness:identity`，让 `persona` 行把事实说一次：
*"You are a coding agent powered by the `{{model}}` model, running on the DeepSeek Harness."*
注意 `assemble({ scope })` 的分裂输出：段落返回时是**未插值**的，那里 persona 文本仍写着 `{{model}}`。
`renderPrompt()` 稍后替换已注册的 `{{...}}` 变量，模型看到的是解析后的名字。

这一行今天是 168 字符，因为它还以一句 `standard` 里没有的常备指令开头：

```
Avoid overthinking, overplanning, getting clever, or getting stuck in the weeds. You are a coding agent powered by the {{model}} model, running on the DeepSeek Harness.
```

80 字符的第一句是本 preset 特有的；87 字符的第二句就是本节描述的那个合并身份。
这条记录过的两个缺陷都已修好：两句原来粘在一起没空格（模型读成 `weeds.You`），
第一句原来是 `Avoid overthink`。修复花了 +16 字符——早先的记录说这个语法修复是免费的，那是错的。

`harness:identity` 在 `standard` 里是 48 字符，这里是 `0`，所以身份只有一个家。

**第三行身份句是 `context:file-reference`**（"Tokens prefixed with @ are workspace paths…"），
由 `dsh-file-reference-local` 每 agent 注册为一个**段落**。它不在任何 standing scope 组装里，
但在每一个真实组装里——怎么够到它见 §1.6。

---

## 七、删一个工具可能连带删掉另一个

`workflow` 和 `ralph` 是一个决定，不是两个。`@deepseek-ai/dsh-tool-ralph` 把 `workflowEngine` 当
**硬**依赖，所以只删 `tool-workflow` 那一行，会让整个 preset 挂载失败：

```
1 row(s) did not activate:
tool-ralph (@deepseek-ai/dsh-tool-ralph): waiting for workflowEngine
```

这是 `inactiveRows()` 审计在干活——而它会让挂载失败，所以要用探针副本发现，而不是靠弄坏一次会话。
删工具行之前先查：有没有**兄弟行** inject 了这一行提供的服务——grep 兄弟行的 `inject` 数组，
或者干脆挂载校验一遍。要单独留下 `ralph`，得把 `workflow-worker-thread` 请回来，
再加上它自己的 `isolate.workflowEngine` realm——为了留一个本 preset 并不想要的工具，代价是两行。

---

## 八、本 preset 的文件

| 文件 | 内容 |
|---|---|
| `agent.cordis.yml` | 组装。`prompt-surface-overlay`（行 id）是唯一改提示词文本的行：`name: './overlay.mjs'`，仅此而已。身份句和每一个被删掉的工具行也在这里。 |
| `overlay.mjs` | `DESCRIPTIONS` / `SECTIONS` / `REPLACED`，加 `inject` 和 `apply`。提示词文本的唯一所在地。 |
| `preset.yml` | roster 元数据：`name`、`description`、`order`。 |
| `PRESET-GUIDE.md` / `PRESET-GUIDE.zh-CN.md` | 本文档的英文版与中文版。 |
| `PROMPT-BASELINE.md` | `standard` 组装出的文本，已冻结。拿它对照，不要凭记忆。 |

测试轮之后删掉的文档：`PROMPT-TESTS.md`（其设计已被报告取代）、`PROMPT-CHECKLIST.md`
（逐条证据已并入报告）、`PROMPT-RUN.md`（测试轮已通过），以及更早的 `ARCHITECTURE.md` 和
`HANDOFF.md`（已合并进本文件）。不要重建它们。
