# 精简（lite）— DSH agent preset

DSH 的 agent preset：工具描述与提示词段落重新措辞过，能力与随包发布的 `standard` 一致，
每轮读到的字更少。提示词文本只在一个地方——`overlay.mjs`。

## 用法

克隆到 DSH 的 preset 目录，文件夹名保持 `lite`：

```powershell
git clone https://github.com/ft5191306/dsh-presets-editor "$HOME\.dsh\.agent-presets\lite"
```

```bash
git clone https://github.com/ft5191306/dsh-presets-editor ~/.dsh/.agent-presets/lite
```

重启 DSH，在会话里选 preset「精简」。

压缩包副本：`agent-presets.rar`。

## 想改提示词

读 `PRESET-GUIDE.zh-CN.md`（英文版 `PRESET-GUIDE.md`）。判断改动的效果，以
`PROMPT-BASELINE.md` 里 `standard` 的组装原文为准，不要凭记忆。
