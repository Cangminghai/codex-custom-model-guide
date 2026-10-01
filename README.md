# 解决 Codex 在使用中转站时出现自定义模型的问题

> 本文是一篇技术笔记。核心要点整理自腾讯云开发者社区文章《[最新版 Codex/ChatGPT/cc-switch 完全上手教程：我与 Codex 的“包办婚姻”](https://cloud.tencent.com/developer/article/2706846)》，并在 Windows 环境（codex-cli 0.150.0-alpha.8）下做了实际验证。感谢原作者踩坑分享。

## 问题现象

使用 API 中转站接入自定义模型时：

- Codex 的模型下拉列表里看不到自己的模型，只有官方模型（旧版本最高到 5.6 就封顶）；
- 强行在配置里指定模型后，界面显示为「自定义 / Custom」，列表里依然找不到；
- 更新客户端也无济于事。

## 根因：新版 Codex 的模型加载逻辑

旧版 Codex 可以在 `config.toml` 里直接写 `models = ["my-model"]` 让模型出现。新版改了规则，模型来源分两层：

| 来源 | 作用 | 可编辑性 |
| --- | --- | --- |
| 服务端缓存 `models_cache.json` | Codex 自动从官方拉取的模型列表 | 会被覆盖，**不要手动改** |
| 用户自定义 `model_catalog_json` | 你自己定义的模型列表 | 想怎么改就怎么改 |

简单说：**新版 Codex 默认只认官方模型；要用自定义 / 中转站模型，必须单独配 `model_catalog_json` 给它“开小灶”。**

角色分工：Codex 负责模型下拉菜单，ChatGPT / 中转站提供模型 API，cc-switch 负责管理密钥切换（注意 `OPENAI_API_KEY` 环境变量要指向正确的服务）。

## 解决步骤

### 1. 别碰 `config.toml` 里的 `models` 字段

```toml
[model_providers.custom]
name = "xxx API"
base_url = "https://api.example.com/v1"
models = ["my-model"]   # ❌ 新版已废弃，写了等于白写
```

`models` 字段在新版基本被无视，真正起作用的是 `model_catalog_json`。

### 2. 创建自定义模型目录文件

```bash
mkdir -p ~/.codex/model-catalogs
touch ~/.codex/model-catalogs/xxx-models.json
```

### 3. 在 `config.toml` 顶层指向它

```toml
model_catalog_json = "/Users/你的用户名/.codex/model-catalogs/xxx-models.json"
```

**坑点**：必须是**绝对路径**，不要用 `~` 缩写，否则 Codex 报 `No such file or directory`。

### 4. JSON 条目的关键点

- **不要手写精简字段**——会连环报错 `missing field base_instructions` → `missing field supports_reasoning_summaries` → `missing field xxx`，像挤牙膏一样。正确做法：**从 `models_cache.json` 里复制一个完整条目，再改 `slug` 和 `display_name`**，所有字段一次齐活。
- `visibility` 必须为 `"list"` 才会出现在下拉菜单（写成 `"hidden"` 就不显示，作者为此折腾了两个晚上）。
- `slug` 必须是中转站 / API 真正认的模型 ID（不是显示名）。
- 完整条目模板见原文章节 2.4（含 `base_instructions`、`context_window`、`truncation_policy` 等必填字段）。

### 5. 重启 Codex

```bash
codex
```

正常后，模型选择菜单里会出现你的自定义模型。

## 常见报错速查

| 报错信息 | 可能原因 | 解决方案 |
| --- | --- | --- |
| `No such file or directory` | `model_catalog_json` 路径不对 | 改用绝对路径，检查文件是否存在 |
| `missing field xxx` | JSON 缺少必填字段 | 用完整条目模板补全所有字段 |
| 设置了 `visibility` 但模型没出现 | 值不是 `"list"` | 改为 `"list"` |
| 模型选了但 API 调用失败 | 环境变量或 API 地址有误 | 检查 `OPENAI_API_KEY` 和 `base_url` |
| MCP 相关报错（如 `node_repl`） | Codex 自带服务启动失败 | 可忽略，不影响模型切换功能 |

## Windows 实测补充（本机验证）

原文章基于 macOS，以下是 Windows（codex-cli 0.150.0-alpha.8）下的实测结论：

1. **验证命令**：`codex debug models` 可导出当前合并后的模型目录，用于确认改动是否生效、以及复制完整条目模板。
2. **实操路径**：导出目录 → 复制一条官方完整条目（如 `gpt-5.6-sol`）→ 改 `slug` 为中转站认的 ID（如 `gpt-6.1-sol`）、改 `display_name` → 写入 `~/.codex/model-catalog.json` → `config.toml` 顶层加 `model_catalog_json` → 重启。合并目录里新模型即出现在列表顶部。
3. **注意**：本版本下本地目录是**替换式加载**——只写新模型会把官方模型挤掉；把官方可见模型一起并进目录即可同时保留。
4. **显示策略**：桌面 App 对来自本地目录的非内置模型，个别位置仍可能显示「自定义」标签，这是 Codex 的显示策略，**不影响实际调用**；CLI 的 `/model` 菜单正常显示自定义名称。
5. **中转站模型列表**：可调用中转站的 `/v1/models` 接口（cc-switch 的「拉取模型」功能同源）获取它支持的全部模型 ID，再决定往目录里加哪些。

## 参考

- [腾讯云开发者社区：《最新版 Codex/ChatGPT/cc-switch 完全上手教程》](https://cloud.tencent.com/developer/article/2706846)（本文要点来源）
- [OpenAI Codex 官方文档](https://developers.openai.com/codex/)
