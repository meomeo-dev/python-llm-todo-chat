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


## 更多案例 

[案例介绍](案例介绍.md)

---

## 安装与依赖

要求：Python 3.10+（推荐 3.11/3.12）。

---

安装为系统工具(MACOS)
```zsh
# 按需开启网络代理
# export http_proxy=http://127.0.0.1:2080 && export HTTP_PROXY=http://127.0.0.1:2080 && export https_proxy=http://127.0.0.1:2080 && export HTTPS_PROXY=http://127.0.0.1:2080 && export all_proxy=socks5://127.0.0.1:2080 && echo "网络代理配置完成"

brew install pipx
pipx install git+https://github.com/meomeo-dev/python-llm-todo-chat.git
# pipx 会将其安装到虚拟环境后，然后挂在 ~/.local/bin 目录下

# 环境变量 API_KEY
 export DEEPSEEK_API_KEY = "..."
 export JINA_API_KEY = "..."

# 运行
llm_todo_chat invoke --interactive --provider deepseek --model deepseek-chat --role deep-researcher --stream
```

---

或者下载源代码后（zsh）：

可以按提示直接使用 [uv](https://github.com/astral-sh/uv)，或继续使用 pip。

```zsh
# 克隆仓库
git clone https://github.com/meomeo-dev/python-llm-todo-chat.git
cd python-llm-todo-chat

# 创建虚拟环境并安装依赖
uv venv
source .venv/bin/activate
uv sync

# 环境变量 API_KEY
 export DEEPSEEK_API_KEY = "..."
 export JINA_API_KEY = "..."

# 运行
python3 llm_todo_chat.py invoke --interactive --provider deepseek --model deepseek-chat --role deep-researcher --stream
```

若缺少依赖，程序会提示类似：
```
缺少依赖，请先安装:

  uv pip install pyyaml python-dateutil ...
```

---

## 快速开始

查看帮助：

```zsh
llm_todo_chat --help
```

### 配置API_KEY

命令前加一个空格，避免密钥被记录到 history 里。

通过环境变量配置

```
 export DEEPSEEK_API_KEY = "..."
 export JINA_API_KEY = "..."
```

或 MacOS 通过命令参数配置, 会存储到 keychain 中

```
 llm_todo_chat config-api --deepseek-key <替换为你的API_KEY>
 # 配置JINA 可以直接使用搜索工具
 llm_todo_chat config-api --jina-key <替换为你的API_KEY>
```

### 交互式对话（可自动调用工具）

内置三类角色（通过 `--role` 选择，对应注入不同身份提示词）：
1. `default`：时间/任务结构化调度角色（强调拆分、排序、执行一致性）
2. `deep-researcher`：深度研究循环（假设 -> 检索 -> 抽取 -> 反思 -> 迭代），自动强调微步迭代与证据链构建
3. `game-roleplay`：沉浸式角色扮演导演（剧情节奏、人设一致、安全/OOC 规范）

```zsh
llm_todo_chat invoke --interactive --provider deepseek --model deepseek-chat --role deep-researcher --stream
```

对话内命令：`/tools` 查看工具签名、`/system` 查看系统提示、`/turns` 查看流水、`/undo` 撤销上一轮、`/del N` 删除第 N 轮、`/exit` 退出。


### 管理内置工具 

直接代码中搜索`TOOLS_REGISTRY`注释掉即可，需要新工具则实现函数后添加进去即可。

### MCP配置

依赖于 FASTMCP, JSON 配置格式与 FAST MCP 保持一样格式。

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

## 数据结构说明

### 查看相关配置文件路径

```zsh
llm_todo_chat paths
```

### Todo（存于 `llm_todo_chat.txt`） 数据结构

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

### Bio 条目（存于 `llm_todo_chat__bio.yaml`）数据结构

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

## 环境变量

### API密钥

- DeepSeek Key：`DEEPSEEK_API_KEY`
- OpenRouter Key：`OPENROUTER_API_KEY`
- Jina Key：`JINA_API_KEY`

使用 `config-api` 写入钥匙串(MacOS)：

```zsh
 llm_todo_chat config-api --deepseek-key sk-xxx --jina-key jina_xxx
```

### 一些预设但没完全实现的环境变量

(实现)交互补全历史裁剪：

- `TODO_HISTORY_DISABLE=1` 关闭历史文件
- `TODO_HISTORY_MAX_LINES` 默认 20000
- `TODO_HISTORY_MAX_BYTES` 默认 5,000,000

(未实现)流式渲染性能：

- `TODO_STREAM_MIN_INTERVAL`（默认 0.08s）
- `TODO_STREAM_MIN_DELTA_CHARS`（默认 256）
- `TODO_STREAM_MAX_CHARS`（默认 400000，用于流式面板内的正文/思考截断上限）
- `TODO_STREAM_TAIL_CHARS`（默认 4000，超过上限时仅显示尾部）
- `TODO_STREAM_SAVE_THRESHOLD`（默认 20000，超长输出自动存 `/tmp/...`）
- `TODO_STREAM_FORCE_EVERY_N`（可选，>0 时每累积 N 次增量强制刷新一次）
- 主题：`TODO_THEME=light|dark|auto`

---

## 交互式对话与工具调用 ANTML 格式说明

模型在需要用工具时，输出如下结构，程序会自动解析与执行：

```xml
<antml:function_calls>
  <antml:invoke name="tool__todo__add_task">
    <antml:parameter name="request_sig"><![CDATA[签名]]></antml:parameter>
    <antml:parameter name="commentary"><![CDATA[Agent为什么要调用]]></antml:parameter>
    <antml:parameter name="title"><![CDATA[买牛奶]]></antml:parameter>
    <antml:parameter name="time"><![CDATA[18:00]]></antml:parameter>
  </antml:invoke>
</antml:function_calls>
```

**invoke**

invoke 使用 3 段式设计, 对工具和能力的调用能较好的区分作用域. `__` 使用两个下划线作为作用域分隔符.

```
[工具箱]__[工具]__[功能]
```

目前的工具箱有 3 类:
- 内置工具箱 -> `tool`
- MCP工具箱 -> `mcp`
- 仿生工具箱 -> `bionic`

`request_sig` 特殊说明: 
Agent发起请求必须要给请求一个命名. 
就像程序代码中的函数名或变量名一样, 通过命名开辟一个逻辑上的内存空间 或 计算空间,
这个很必要, Agent通过 `request_sig` 在语境上开辟了一个记忆空间.

`commentary` 仅用于展示，可以理解为Agent的快思考, 不会传入工具执行。


工具调用的最大轮次：深度研究模式 6 + MAX_TOOL_ROUNDS_MIN_BOUNDARY 轮，其它 0 + MAX_TOOL_ROUNDS_MIN_BOUNDARY 轮（可改代码常量）。

---

## 可用工具（注册于 TOOLS_REGISTRY）

说明：以下为默认已注册工具。若要禁用某工具，可在 `llm_todo_chat.py` 中注释其条目；要新增工具，按现有 wrapper 模式实现函数并加入 `TOOLS_REGISTRY`。

Bio：(类似于长期记忆, 只不过是命名上“生平传记(Biography)” 命名更好的反映在AI眼中用户是一个怎么样的人.)
- `tool__bio__add(title, content, tags?)` 新增用户生平传记记录
- `tool__bio__update(id, title?, content?, tags?)` 更新用户生平传记记录
- `tool__bio__delete(id)` 删减用户生平传记记录
- `tool__bio__list()` 列出用户生平传记记录

Todo：
- `tool__todo__add_task(title, time?, project?, tags?, notes?)`  新增代办
- `tool__todo__list(date?)` （支持 `YYYY-MM-DD` 单日 或 `YYYY-MM-DD..YYYY-MM-DD` 区间）列出指定日期的所有代办
- `tool__todo__complete(id, note?)` 完成代表
- `tool__todo__report_daily(date?)` 日报
- `tool__todo__report_weekly(start?)` （周一 ISO 日期）周报
- `tool__todo__update(id, title?, time?, status?, project?, tags?, notes?, note_append?)` 更新代办
- `tool__todo__delete(id)` 删除代办
* `tool__todo__query(query)` 全文搜索代办

Jina：
- `tool__jina__web_search(query)`：仅支持普通关键字（不支持 `site:` / 引号 / 布尔语法等），默认返回 no-content 摘要
- `tool__jina__fetch_markdown(url, token_budget?)`：抓取网页并返回 Markdown（`token_budget` 16_000–32_000）

Reflect（对话结构化反思与重置, 实验性）：
- `tool__reflect__negate_and_reflect(first_user?, inherited?, next_direction?, current_status?, style?)` 抛弃所有历史对话, 携带历史对话中最重要的信息进入新的轮次.

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
- `mcp__call(server, tool, params_json?, raw?)`：调用已配置 MCP 服务器工具
- `mcp__list(server?, json_mode?)`：列出可用 MCP 工具（可按 server 过滤）

仿生:
- `bionic__speech__say(content, say_all?)`：说话能力, 调用 MacOS 的 Say 命令使用 Siri 完成语音播报

---

## 许可

MIT License（见文件头注释）。