# dsh-presets-editor DSH自定义工具和提示词
该工具为了解决标准模式提示词太多，极简模式工具太少的问题。

在\.dsh\.agent-presets\内新建一个模式，仓库agent-pcordis.yaml是带调用overlay.mjs的标准模式，你可以在overlay.mjs中修改提示词。如果要删工具overlay.mjs和cordis.yml要一起删或者直接让ai改。如果想修改DSH的其他模式，让ai读md。 
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
