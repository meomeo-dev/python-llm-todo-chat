# deepseek_todo_tools_agent / llm_todo_chat.py

一个用 YAML 管理的极简待办系统 + 可选的交互式对话代理（可自动调用工具 & MCP 工具）。

- 待办数据存储在 `~/llm_todo_chat.txt`，采用多文档 YAML（`---` 分隔）。
- 个人画像（Bio）独立存储在 `~/llm_todo_chat__bio.yaml`，YAML 文本，便于在对话时注入 System Prompt（含 `created_at` / `updated_at` 时间戳字段）。
- 内置命令行工具：新增 / 搜索 / 更新 / 导出 / 日报 / 周报等。
- 可调用 DeepSeek / Moonshot(Kimi) / OpenRouter Chat API（均支持流式/非流式；DeepSeek / OpenRouter 具推理分离）+ Jina Web Search & Reader（网页搜索与抓取）。
- 提供“交互式会话模式”：模型可输出 ANTML 工具调用，程序自动执行并把结果回注继续对话。
- 会话与工具调用流水使用 DuckDB (`~/.llm_todo_chat.duckdb`)，兼容类 SQLite SQL 的轻量分析。
- 交互历史（用于提示补全）保存在：`~/.llm_todo_chat__history`（可通过环境变量裁剪或禁用）。
- 支持 MCP 多服务器（fastmcp）调用，统一封装 `tool__mcp__call` / `tool__mcp__list`。

适合把“日常事务 + 研究检索 + 轻量记忆”统一放在一个 CLI 工作流里。

---

## 效果预览

<p align="center">
  <img src="./Screenshot.png" alt="LLM Todo Chat 演示截图" style="max-width: 760px; border: 1px solid #eaeaea; border-radius: 8px;" />
</p>

<div style="padding: 8px 12px; background: #fff6e5; border: 1px solid #ffd59e; border-radius: 6px;">
  <strong>查看演示视频：</strong>
  GitHub README 不支持内联播放视频，请<a href="./llm_todo.mp4">点击此处下载或在新窗口播放 llm_todo.mp4</a>。
</div>

---

## 安装与依赖

要求：Python 3.10+（推荐 3.11/3.12）。

安装示例（zsh）：

可以按提示直接使用 [uv](https://github.com/astral-sh/uv)（更快的包管理），或继续使用 pip。

```zsh
uv venv
source .venv/bin/activate
uv pip install -e .
```

若缺少依赖，程序会提示类似：
```
缺少依赖，请先安装:

  uv pip install pyyaml python-dateutil ...
```

---

安装为系统工具(MACOS)
```zsh
# 按需开启网络代理
# export http_proxy=http://127.0.0.1:2080 && export HTTP_PROXY=http://127.0.0.1:2080 && export https_proxy=http://127.0.0.1:2080 && export HTTPS_PROXY=http://127.0.0.1:2080 && export all_proxy=socks5://127.0.0.1:2080 && echo "网络代理配置完成"

brew install pipx
pipx install git+https://github.com/meomeo-dev/python-llm-todo-chat.git
# pipx 会将其安装到虚拟环境后，然后挂在 ~/.local/bin 目录下

# 安装完成后可直接执行
llm_todo_chat --help
```

---

## 快速开始

查看帮助：

```zsh
llm_todo_chat --help
```

配置API_KEY

命令前加一个空格，避免密钥被记录到 history 里。

```
 llm_todo_chat config-api --deepseek-key <替换为你的API_KEY>
 # 配置JINA 可以直接使用搜索工具
 llm_todo_chat config-api --jina-key <替换为你的API_KEY>
```

管理工具 则直接代码中搜索`TOOLS_REGISTRY`注释掉即可，需要新工具则实现函数后添加进去即可。

MCP配置依赖于 FASTMCP, JSON 配置格式于 FAST MCP保持一样格式。

```
{
  "mcpServers": {
    "context7": {
      "transport": "stdio",
      "command": "uv",
      "args": ["run", "python3", "-m", "mcp_server_context7"],
      "env": {
        "CLIENT_IP_ENCRYPTION_KEY": "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f",
      },
      "timeoutSeconds": 30
    }
  }
}
```

交互式对话（可自动调用工具）：

```zsh
llm_todo_chat invoke --interactive --provider deepseek --model deepseek-reasoner --role deep-researcher --stream
```

查看相关配置文件路径：

```zsh
llm_todo_chat paths
```

## todo.yaml 数据结构

一个文件内有多份 YAML 文档，使用 `---` 分隔。每份文档（Document）：

```yaml
date: YYYY-MM-DD
tasks:
  - id: 自动生成（时间戳+短UUID）
    time: "09:30" | null
    title: 任务标题
    notes: "可选备注"
    status: todo | in-progress | waiting | done
    project: 项目名 | null
    tags: [tag1, tag2]
```

程序会为缺省字段补默认值，状态不合法会回落到 `todo` 状态。

Bio 条目（存于 `llm_todo_chat__bio.yaml`）示例：

```yaml
version: 1
bios:
  - id: 20250101120000-ab12cd
    title: 职业背景
    content: |
      使用 markdown 描述，可多行。
    tags: [profile, work]
    created_at: 2025-01-01T12:00:00
    updated_at: 2025-01-02T09:30:00
```

---

## 环境变量与平台集成

- DeepSeek Key：`DEEPSEEK_API_KEY`
- MoonShot Key：`MOONSHOT_API_KEY`
- OpenRouter Key：`OPENROUTER_API_KEY`
- Jina Key：`JINA_API_KEY`

使用 `config-api` 写入钥匙串(MacOS)：

```zsh
llm_todo_chat config-api --deepseek-key sk-xxx --jina-key jina_xxx
```

交互补全历史裁剪：

- `TODO_HISTORY_DISABLE=1` 关闭历史文件
- `TODO_HISTORY_MAX_LINES` 默认 20000
- `TODO_HISTORY_MAX_BYTES` 默认 5,000,000

流式渲染性能：

- `TODO_STREAM_MIN_INTERVAL`（默认 0.08s）
- `TODO_STREAM_MIN_DELTA_CHARS`（默认 256）
- `TODO_STREAM_MAX_CHARS`（默认 400000，用于流式面板内的正文/思考截断上限）
- `TODO_STREAM_TAIL_CHARS`（默认 4000，超过上限时仅显示尾部）
- `TODO_STREAM_SAVE_THRESHOLD`（默认 20000，超长输出自动存 `/tmp/...`）
- `TODO_STREAM_FORCE_EVERY_N`（可选，>0 时每累积 N 次增量强制刷新一次）
- 主题：`TODO_THEME=light|dark|auto`

---

## 交互式对话与工具调用（ANTML）

进入对话：

```zsh
python llm_todo_chat.py invoke --interactive --role default --stream
```

对话内命令：`/tools` 查看工具签名、`/system` 查看系统提示、`/turns` 查看流水、`/undo` 撤销上一轮、`/del N` 删除第 N 轮、`/exit` 退出。

模型在需要用工具时，输出如下结构，程序会自动解析与执行：

```xml
<antml:function_calls>
  <antml:invoke name="tool__todo__add_task">
    <antml:parameter name="commentary"><![CDATA[为什么要调用]]></antml:parameter>
    <antml:parameter name="title"><![CDATA[买牛奶]]></antml:parameter>
    <antml:parameter name="time"><![CDATA[18:00]]></antml:parameter>
  </antml:invoke>
</antml:function_calls>
```

- `commentary` 仅用于展示，不会传入工具执行。
- 工具调用与结果会被记录入 SQLite，会话会多轮循环“调用 -> 注入结果 -> 继续追问”。
- 最大轮次：深度研究模式 6 轮，其它 2 轮（可改代码常量）。

---

## 可用工具（注册于 TOOLS_REGISTRY）

说明：以下为默认已注册工具。若要禁用某工具，可在 `llm_todo_chat.py` 中注释其条目；要新增工具，按现有 wrapper 模式实现函数并加入 `TOOLS_REGISTRY`。`tool__todo__search` 已实现但默认未注册（避免模型过度泛搜），可按需放开。

Bio：
- `tool__bio__add(title, content, tags?)`
- `tool__bio__update(id, title?, content?, tags?)`
- `tool__bio__delete(id)`
- `tool__bio__list()`

Todo：
- `tool__todo__add_task(title, time?, project?, tags?, notes?)`
- `tool__todo__list(date?)` （支持 `YYYY-MM-DD` 单日 或 `YYYY-MM-DD..YYYY-MM-DD` 区间）
- `tool__todo__complete(id, note?)`
- `tool__todo__report_daily(date?)`
- `tool__todo__report_weekly(start?)` （周一 ISO 日期）
- `tool__todo__update(id, title?, time?, status?, project?, tags?, notes?, note_append?)`
- `tool__todo__delete(id)`
  * 可选未默认启用：`tool__todo__search(query)` – 全文/标签模糊搜索

Jina：
- `tool__jina__web_search(query)`：仅支持普通关键字（不支持 `site:` / 引号 / 布尔语法等），默认返回 no-content 摘要
- `tool__jina__fetch_markdown(url, token_budget?)`：抓取网页并返回 Markdown（`token_budget` 16_000–32_000）

Reflect（对话结构化反思与重置）：
- `tool__reflect__negate_and_reflect(first_user?, inherited?, next_direction?, current_status?, style?)`

Human / 文件：
- `tool__human__input(prompt, default?, kind?)`：human-in-loop 输入（kind: text|confirm|secret）
- `tool__fs__save_file(dir, filename, content, append?, make_dirs?, encoding?)`：安全写入/追加文本文件

Safety：
- `tool__safety__terminate(reason?, category?, user_message?)`：安全触发强制终止（返回控制标记）

实用（Util）：
- `tool__calc__eval(expression)`：安全算术（+ - * / // % ** 括号）
- `tool__date__calc(op, ...)`：日期计算。
  * `op=diff` 需要 `date1,date2`；`unit=days|weeks`
  * `op=add` 需要 `date, interval`，`interval` 支持组合如 `1y2m-3d` / `10d 2w`

MCP（多服务器工具桥接）：
- `tool__mcp__call(server, tool, params_json?, raw?)`：调用已配置 MCP 服务器工具
- `tool__mcp__list(server?, json_mode?)`：列出可用 MCP 工具（可按 server 过滤）

---

## DeepSeek / Jina 集成

DeepSeek：
- 非流式：返回完整文本与 `usage`
- 流式：分离 `reasoning_content` 与 `content` 回调输出；末尾汇总 `usage`
- 会自动把 System Header（知识截止、当前日期时间）与用户 Bio Markdown 注入到 System Prompt 前。

Jina：
- 搜索：`s.jina.ai` （no-content 摘要）
- Reader：`r.jina.ai`，返回 Markdown（默认设置 `X-Return-Format: markdown` 与 `X-Token-Budget: 16000`，可通过工具参数提升上限到 32000）

API Key 解析优先级：显式参数 > 文件内常量（仅 DeepSeek 可选占位）> 环境变量 > macOS 钥匙串。

MCP 相关：
- 配置文件：`~/llm_todo_chat__mcp_servers.json`（字段结构：`{"mcpServers": { name: {transport, command, args, env, timeout}}}`）
- 环境变量 `TODO_MCP_NO_BANNER=1` 可自动为 fastmcp 启动命令注入 `--no-banner`
- 使用工具：`tool__mcp__list` 查看，`tool__mcp__call` 调用具体远程工具

---

## 会话与数据持久化结构

SQLite 表结构（自动创建，路径 `~/.llm_todo_chat.duckdb`）：

- `sessions`：记录一次交互式会话（uuid / started_at / ended_at / role / model / theme / system_hash）
- `messages`：对话消息流水（包含 reasoning 分离内容与 usage token）
- `tool_calls`：模型输出的工具调用（round_index 支持同一轮多次循环）
- `tool_results`：工具执行回注的结果内容

这样便于：
1. 复盘会话推理链与工具使用序列
2. 统计工具调用频率、模型 token 使用情况
3. 构建后续的检索增强或回放功能

---

## 角色身份提示词（Role Identity Prompts）

内置三类角色（通过 `--role` 选择，对应注入不同身份提示词）：

1. `default`：时间/任务结构化调度角色（强调拆分、排序、执行一致性）
2. `deep-researcher`：深度研究循环（假设 -> 检索 -> 抽取 -> 反思 -> 迭代），自动强调微步迭代与证据链构建
3. `game-roleplay`：沉浸式角色扮演导演（剧情节奏、人设一致、安全/OOC 规范）

选择示例：
```zsh
llm_todo_chat invoke --interactive --provider deepseek --model deepseek-reasoner --role deep-researcher --stream
```

---

## 许可

MIT License（见文件头注释）。