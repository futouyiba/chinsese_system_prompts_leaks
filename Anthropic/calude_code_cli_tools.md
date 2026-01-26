# Claude Code 内部工具 - 技术参考

> **Claude Code 内部工具的完整技术文档**

本文档提供了有关 Claude Code 内部工具的全面技术细节，包括参数架构、实现行为和使用模式。

### Claude Sonnet 4.5

**技术细节：**
- **模型 ID：** `claude-sonnet-4-5-20250929`
- **模型名称：** Sonnet 4.5
- **发布日期：** 2025年9月29日
- **当前日期：** 2025年10月17日
- **知识截止日期：** 2025年1月

---

## 目录

1. [文件操作](#file-operations)
2. [执行工具](#execution-tools)
3. [Agent 管理](#agent-management)
4. [规划与追踪](#planning--tracking)
5. [用户交互](#user-interaction)
6. [Web 操作](#web-operations)
7. [IDE 集成](#ide-integration)
8. [MCP 资源](#mcp-resources)
9. [完整实现总结](#complete-implementation-summary)

---

## 文件操作

### Read 工具

**用途：** 从本地文件系统读取文件内容，支持多模态和部分读取。

**技术实现：**

Read 工具提供直接的文件系统访问，并具有智能内容解析功能：
- 以适当的权限访问机器上的任何文件
- 默认读取限制：从文件开头起 2000 行
- 行截断：每行 2000 个字符
- 输出格式：类似 `cat -n` 风格，行号从 1 开始
- 行号前缀格式：`空格 + 行号 + 制表符 + 内容`

**多模态能力：**

该工具通过专用处理器支持多种文件格式：
- **图像（PNG、JPG 等）：** 内容以视觉方式呈现，因为 Claude Code 是一个多模态 LLM
- **PDF 文件：** 逐页处理，提取文本和视觉内容
- **Jupyter notebooks (.ipynb)：** 返回所有单元格及其输出，结合了代码、文本和可视化

**错误处理：**

- 空文件会触发系统提醒警告，而不是显示内容
- 路径无效会返回相应的错误消息
- 拒绝权限错误将呈现给用户

**约束：**

- 无法读取目录（请改用 Bash `ls` 命令）
- 必须使用绝对路径
- 完全支持屏幕截图和临时文件

**参数架构：**

```typescript
interface ReadTool {
  file_path: string;      // 文件的绝对路径（必填）
  offset?: number;        // 起始行号（可选）
  limit?: number;         // 读取的行数（可选）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "additionalProperties": false,
  "required": ["file_path"],
  "properties": {
    "file_path": {
      "type": "string",
      "description": "要读取的文件的绝对路径"
    },
    "offset": {
      "type": "number",
      "description": "开始读取的行号。仅在文件太大无法一次读取时提供"
    },
    "limit": {
      "type": "number",
      "description": "要读取的行数。仅在文件太大无法一次读取时提供。"
    }
  }
}
```

**行为总结：**
- 默认：前 2000 行
- 行号：从 1 开始（cat -n 格式）
- 行截断：2000 个字符
- 状态：无状态，可以多次调用

---

### Write 工具

**用途：** 创建新文件或使用内置安全机制完全覆盖现有文件。

**技术实现：**

Write 工具提供原子文件写入操作，并强制执行安全检查：
- 完全覆盖现有文件（不进行部分更新）
- 对现有文件强制执行“写入前读取”验证
- 要求使用绝对路径（不支持相对路径）
- 原子写入操作（文件要么被完整写入，要么保持不变）

**安全机制：**

内置防止意外覆盖的保护措施：
- **强制写入前读取：** 如果在当前会话中尚未读取现有文件，系统将使操作失败
- **会话追踪：** 保留已读取文件的记录，以验证写入操作
- **最佳实践强制执行：** 对现有文件首选 Edit 工具，Write 仅用于新文件

**设计理念：**

- 对于现有文件的修改，首选 Edit 工具
- 仅在创建真正的新文件时使用 Write
- 除非用户明确要求，否则避免创建文档文件（*.md, README）
- 除非用户明确要求，否则不插入表情符号

**参数架构：**

```typescript
interface WriteTool {
  file_path: string;      // 绝对路径（必须是绝对路径，不能是相对路径）（必填）
  content: string;        // 完整的文件内容（必填）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["file_path", "content"],
  "additionalProperties": false,
  "properties": {
    "file_path": {
      "type": "string",
      "description": "要写入的文件的绝对路径（必须是绝对路径，不能是相对路径）"
    },
    "content": {
      "type": "string",
      "description": "要写入文件的内容"
    }
  }
}
```

**执行规则：**
- 写入前读取：系统对现有文件强制执行
- 路径验证：必须是绝对路径
- 会话状态：在当前对话中追踪已读取的文件

---

### Edit 工具

**用途：** 使用精确匹配在文件中执行外科手术般的字符串替换。

**技术实现：**

Edit 工具实现精确的字符串匹配和替换：
- 针对精确的字符串匹配进行操作（不是正则表达式或模式）
- 需要在当前会话中预先执行读取操作
- 保留文件编码和换行符
- 原子操作（文件要么被完整更新，要么保持不变）

**字符串匹配算法：**

该工具使用精确的字符串匹配，其行为如下：
- **唯一性要求：** `old_string` 在文件中必须恰好有一个匹配项（除非 `replace_all=true`）
- **空格敏感性：** 保留源文件中的精确缩进（制表符/空格）
- **行号前缀处理：** 行号前缀（`空格 + 行号 + 制表符`）之后的内容才是实际的文件内容
- **失败模式：** 如果 `old_string` 不唯一，则操作失败（防止歧义编辑）

**替换模式：**

1. **单个替换（默认）：** 替换一个唯一的匹配项
   - 如果 `old_string` 出现多次或零次，则失败
   - 使用场景：对特定代码位置进行精确编辑

2. **全部替换 (`replace_all=true`)：** 替换所有匹配项
   - 可用于整个文件中的变量重命名
   - 无唯一性要求
   - 使用场景：重构、批量替换

**安全机制：**

- **强制编辑前读取：** 系统验证在对话中至少读取过该文件一次
- **内容验证：** `new_string` 必须与 `old_string` 不同
- **缩进保留：** 来自 Read 工具输出的精确空格匹配
- **会话追踪：** 保留已读取文件的列表以进行验证

**参数架构：**

```typescript
interface EditTool {
  file_path: string;      // 绝对路径（必须是绝对路径，不能是相对路径）（必填）
  old_string: string;     // 要查找并替换的精确文本（必填）
  new_string: string;     // 替换文本（必须与 old_string 不同）（必填）
  replace_all?: boolean;  // 替换所有匹配项（默认值：false）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["file_path", "old_string", "new_string"],
  "additionalProperties": false,
  "properties": {
    "file_path": {
      "type": "string",
      "description": "要修改的文件的绝对路径"
    },
    "old_string": {
      "type": "string",
      "description": "要替换的文本"
    },
    "new_string": {
      "type": "string",
      "description": "用于替换的文本（必须与 old_string 不同）"
    },
    "replace_all": {
      "type": "boolean",
      "default": false,
      "description": "替换所有出现的 old_string（默认为 false）"
    }
  }
}
```

**常见使用场景：**
- 特定代码部分的错误修复
- 更新函数实现
- 变量/函数重命名（配合 `replace_all`）
- 配置更改
- 文档更新

---

### Glob 工具

**用途：** 适用于任何代码库大小的快速文件模式匹配。

**技术实现：**

使用 glob 模式进行高性能文件搜索：
- 针对任何代码库大小优化的快速模式匹配
- 返回按修改时间排序的文件路径（最近的优先）
- 支持并行执行（在单个消息中多次调用）
- 与 Task 工具集成以进行复杂搜索

**模式语法：**

支持标准 glob 模式：
- `*` - 匹配除 `/` 之外的任何字符（单级目录）
- `**` - 匹配包括 `/` 在内的任何字符（递归，所有子目录）
- `?` - 恰好匹配一个字符
- `{a,b}` - 匹配 `a` 或 `b`（交替）
- `[abc]` - 匹配括号中的任何单个字符（字符类）
- `[a-z]` - 匹配范围内的任何字符
- `[!abc]` - 匹配任何不在括号中的字符（否定）

**常见模式：**
- `**/*.js` - 递归匹配所有 JavaScript 文件
- `src/**/*.{ts,tsx}` - 匹配 src/ 目录下的所有 TypeScript 文件
- `test/**/*.[jt]s` - 匹配 test/ 目录下的所有 .js 或 .ts 文件
- `*.json` - 匹配当前目录下的所有 JSON 文件

**参数架构：**

```typescript
interface GlobTool {
  pattern: string;        // 用于匹配文件的 Glob 模式（必填）
  path?: string;         // 要在其中搜索的目录（默认为当前工作目录）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["pattern"],
  "additionalProperties": false,
  "properties": {
    "pattern": {
      "type": "string",
      "description": "要与文件匹配的 glob 模式"
    },
    "path": {
      "type": "string",
      "description": "要搜索的目录。如果不指定，将使用当前工作目录。重要提示：省略此字段以使用默认目录。不要输入 \"undefined\" 或 \"null\" - 只需为默认行为省略它。如果提供，必须是有效的目录路径。"
    }
  }
}
```

**重要说明：**
- 省略 `path` 字段以使用当前工作目录（默认行为）
- 永远不要将 `path` 设置为 \"undefined\" 或 \"null\" - 只需省略该字段
- 结果按修改时间排序（最近的优先）
- 即使在大型代码库中也能高效工作

---

### Grep 工具

**用途：** 使用 ripgrep 进行高性能内容搜索。

**技术实现：**
- “一个构建在 **ripgrep** 之上的强大搜索工具”
- “**始终**使用 Grep 进行搜索任务。**切勿**将 `grep` 或 `rg` 作为 Bash 命令调用。Grep 工具已针对正确的权限和访问进行了优化”
- “支持**完整的正则语法**（例如，\"log.*Error\"，\"function\\s+\\w+\"）”
- “**输出模式：\"content\" 显示匹配行，\"files_with_matches\" 仅显示文件路径（默认），\"count\" 显示匹配计数**”
- “模式语法：使用 **ripgrep（不是 grep）** - 字面意义的大括号需要转义（使用 `interface\\{\\}` 在 Go 代码中查找 `interface{}`）”
- “**多行匹配：默认情况下，模式仅在单行内匹配**。对于跨行模式（如 `struct \\{[\\s\\S]*?field`），请使用 `multiline: true`”

**工具访问：**
- “对于需要多轮进行的开放式搜索，请使用 Task 工具”
- “你可以在单次响应中调用多个工具。推测性地并行执行多个搜索总是更好的”

**参数：**
```typescript
interface GrepTool {
  pattern: string;              // 要搜索的正则模式（必填）
  path?: string;                // 要搜索的文件或目录（默认为当前工作目录）
  output_mode?: 'content' | 'files_with_matches' | 'count';  // 默认：\"files_with_matches\"
  glob?: string;                // 用于过滤文件的 Glob 模式（例如 \"*.js\", \"*.{ts,tsx}\"）
  type?: string;                // 文件类型（js, py, rust, go, java 等） - 比 include 更高效
  '-i'?: boolean;               // 区分大小写的搜索
  '-n'?: boolean;               // 显示行号（需要 output_mode: \"content\"）
  '-A'?: number;                // 匹配后的行数（需要 output_mode: \"content\"）
  '-B'?: number;                // 匹配前的行数（需要 output_mode: \"content\"）
  '-C'?: number;                // 匹配前后的行数（需要 output_mode: \"content\"）
  multiline?: boolean;          // 启用多行模式（默认：false）
  head_limit?: number;          // 将输出限制为前 N 行/条目
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["pattern"],
  "additionalProperties": false,
  "properties": {
    "pattern": {
      "type": "string",
      "description": "要在文件内容中搜索的正则表达式模式"
    },
    "path": {
      "type": "string",
      "description": "要搜索的文件或目录 (rg PATH)。默认为当前工作目录。"
    },
    "output_mode": {
      "type": "string",
      "enum": ["content", "files_with_matches", "count"],
      "description": "输出模式：\"content\" 显示匹配行（支持 -A/-B/-C 上下文，-n 行号，head_limit），\"files_with_matches\" 显示文件路径（支持 head_limit），\"count\" 显示匹配计数（支持 head_limit）。默认为 \"files_with_matches\"。"
    },
    "glob": {
      "type": "string",
      "description": "用于过滤文件的 Glob 模式（例如 \"*.js\", \"*.{ts,tsx}\"） - 映射到 rg --glob"
    },
    "type": {
      "type": "string",
      "description": "要搜索的文件类型 (rg --type)。常用类型：js, py, rust, go, java 等。对于标准文件类型，比 include 更高效。"
    },
    "-i": {
      "type": "boolean",
      "description": "不区分大小写的搜索 (rg -i)"
    },
    "-n": {
      "type": "boolean",
      "description": "在输出中显示行号 (rg -n)。需要 output_mode: \"content\"，否则会被忽略。"
    },
    "-A": {
      "type": "number",
      "description": "每项匹配之后显示的行数 (rg -A)。需要 output_mode: \"content\"，否则会被忽略。"
    },
    "-B": {
      "type": "number",
      "description": "每项匹配之前显示的行数 (rg -B)。需要 output_mode: \"content\"，否则会被忽略。"
    },
    "-C": {
      "type": "number",
      "description": "每项匹配之前和之后显示的行数 (rg -C)。需要 output_mode: \"content\"，否则会被忽略。"
    },
    "multiline": {
      "type": "boolean",
      "description": "启用多行模式，其中 . 匹配换行符且模式可以跨行 (rg -U --multiline-dotall)。默认值：false。"
    },
    "head_limit": {
      "type": "number",
      "description": "将输出限制为前 N 行/条目，等同于 \"| head -N\"。适用于所有输出模式：content（限制输出行），files_with_matches（限制文件路径），count（限制计数条目）。未指定时，显示来自 ripgrep 的所有结果。"
    }
  }
}
```

**核心实现：**
- 使用 ripgrep 二进制文件（已明确说明）
- 默认 output_mode: \"files_with_matches\"
- 上下文标志 (-A/-B/-C) 仅在 output_mode: \"content\" 时有效
- 多行模式默认禁用（模式仅匹配单行）

---

### NotebookEdit 工具

**用途：** 通过替换、插入、删除操作编辑 Jupyter notebook 单元格。

**技术实现：**
- “完全替换 Jupyter notebook（.ipynb 文件）中特定单元格的内容”
- “notebook_path 参数必须是**绝对路径，不能是相对路径**”
- “cell_number 从 **0 开始索引**”
- “使用 **edit_mode=insert** 在 cell_number 指定的索引处添加新单元格”
- “使用 **edit_mode=delete** 删除 cell_number 指定索引处的单元格”

**参数：**
```typescript
interface NotebookEditTool {
  notebook_path: string;      // .ipynb 文件的绝对路径（必填，必须是绝对路径）
  new_source: string;         // 新单元格内容（必填）
  cell_id?: string;           // 要编辑/在其后插入的单元格 ID
  cell_type?: 'code' | 'markdown';  // 单元格类型（edit_mode=insert 时必填）
  edit_mode?: 'replace' | 'insert' | 'delete';  // 默认值：\"replace\"
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["notebook_path", "new_source"],
  "additionalProperties": false,
  "properties": {
    "notebook_path": {
      "type": "string",
      "description": "要编辑的 Jupyter notebook 文件的绝对路径（必须是绝对路径，不能是相对路径）"
    },
    "new_source": {
      "type": "string",
      "description": "单元格的新源代码"
    },
    "cell_id": {
      "type": "string",
      "description": "要编辑的单元格的 ID。插入新单元格时，新单元格将插入到具有此 ID 的单元格之后，如果未指定，则插入到开头。"
    },
    "cell_type": {
      "type": "string",
      "enum": ["code", "markdown"],
      "description": "单元格的类型（代码或 markdown）。如果未指定，则默认为当前单元格类型。如果使用 edit_mode=insert，则此项必填。"
    },
    "edit_mode": {
      "type": "string",
      "enum": ["replace", "insert", "delete"],
      "description": "要进行的编辑类型（替换、插入、删除）。默认为替换。"
    }
  }
}
```

**单元格索引：**
- 从 0 开始索引（第一个单元格索引为 0）
- 通过 cell_id 识别单元格
- 插入时，在指定的 cell_id 之后添加新单元格

---

## 执行工具

### Bash 工具

**用途：** 在状态保存的持久 shell 会话中执行命令。

**技术实现：**
- “在具有可选超时时间的**持久 shell 会话**中执行给定的 bash 命令”
- “command 参数是必需的”
- “你可以指定以毫秒为单位的可选超时时间（最高 **600000ms / 10 分钟**）。如果未指定，命令将在 **120000ms（2 分钟）**后超时”
- “如果输出超过 **30000 个字符**，输出将在返回给你之前被截断”
- “你可以使用 `run_in_background` 参数在后台运行命令，这允许你在命令运行时继续工作”
- “**切勿使用 `run_in_background` 运行 'sleep'，因为它会立即返回**。使用此参数时，你不需要在命令末尾使用 '&'”

**命令限制：**
- “**避免**将 Bash 与 `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk` 或 `echo` 命令一起使用，除非明确指示或这些命令对于任务确实是必要的”
- “**切勿**将 bash 用于文件操作（cat/head/tail, grep, find, sed/awk, echo >/cat <<EOF）”

**多条命令：**
- “发布多条命令时：**如果命令相互独立**且可以并行运行，请在**单次消息中进行多次 Bash 工具调用**”
- “**如果命令相互依赖**且必须顺序运行，请使用带有 '&&' 的单个 Bash 调用来将它们链在一起”
- “仅当你需要顺序运行命令但不关心较早的命令是否失败时，才使用 ';'”
- “**不要使用换行符分隔命令**（带引号的字符串中可以使用换行符）”

**工作目录：**
- “尝试通过**使用绝对路径并避免使用 `cd`** 在整个会话中维护当前工作目录。如果用户明确要求，你可以使用 `cd`”

**参数：**
```typescript
interface BashTool {
  command: string;              // 要执行的 shell 命令（必填）
  description?: string;         // 清晰、简练的描述（5-10 个单词）
  timeout?: number;             // 毫秒（最高 600000）
  run_in_background?: boolean;  // 在后台运行命令（默认值：false）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["command"],
  "additionalProperties": false,
  "properties": {
    "command": {
      "type": "string",
      "description": "要执行的命令"
    },
    "description": {
      "type": "string",
      "description": "用 5-10 个单词清晰、简练地描述此命令的作用，使用主动语态。示例：\n输入: ls\n输出: 列出当前目录中的文件\n\n输入: git status\n输出: 显示工作树状态\n\n输入: npm install\n输出: 安装包依赖项\n\n输入: mkdir foo\n输出: 创建目录 'foo'"
    },
    "timeout": {
      "type": "number",
      "description": "以毫秒为单位的可选超时时间（最高 600000）"
    },
    "run_in_background": {
      "type": "boolean",
      "description": "设置为 true 以在后台运行此命令。稍后使用 BashOutput 读取输出。"
    }
  }
}
```

**操作限制：**
- 默认超时：120000ms（2 分钟）
- 最大超时：600000ms（10 分钟）
- 输出在 30000 个字符处截断

**Git 安全：**
- “**切勿**更新 git 配置”
- “**切勿**运行破坏性/不可逆的 git 命令（如 push --force, hard reset 等），除非用户明确要求”
- “**切勿**跳过钩子（--no-verify, --no-gpg-sign 等），除非用户明确要求”
- “**切勿**向 main/master 分支运行强制推送，如果用户要求，请警告用户”

---

### BashOutput 工具

**用途：** 从后台 shell 检索增量输出。

**技术实现：**
- “从正在运行或已完成的后台 bash shell 中检索输出”
- “接受标识 shell 的 shell_id 参数”
- “**始终仅返回自上次检查以来的新输出**”
- “返回 stdout 和 stderr 输出以及 shell 状态”
- “支持可选的正则过滤，以仅显示与模式匹配的行”
- “任何不匹配的行将**不再可供读取**”（使用过滤器时）

**参数：**
```typescript
interface BashOutputTool {
  bash_id: string;        // 后台 shell 的 ID（必填）
  filter?: string;        // 用于过滤输出行的可选正则表达式
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["bash_id"],
  "additionalProperties": false,
  "properties": {
    "bash_id": {
      "type": "string",
      "description": "要从中检索输出的后台 shell 的 ID"
    },
    "filter": {
      "type": "string",
      "description": "用于过滤输出行的可选正则表达式。仅包含与此正则匹配的行在结果中。任何不匹配的行将不再可供读取。"
    }
  }
}
```

**行为：**
- 仅返回自上次检查以来的新输出
- 非阻塞（立即返回）
- 过滤器永久删除不匹配的行

---

### KillShell 工具

**用途：** 终止后台 bash shell。

**技术实现：**
- “根据 ID 终止正在运行的后台 bash shell”
- “接受标识要终止的 shell 的 shell_id 参数”
- “返回成功或失败状态”

**参数：**
```typescript
interface KillShellTool {
  shell_id: string;       // 要终止的 shell 的 ID（必填）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["shell_id"],
  "additionalProperties": false,
  "properties": {
    "shell_id": {
      "type": "string",
      "description": "要终止的后台 shell 的 ID"
    }
  }
}
```

---

## Agent 管理

### Task 工具

**用途：** 运行具有专门工具访问权限的自主子代理。

**技术实现：**
- “运行一个新的代理来**自主**处理复杂的多步任务”
- 可用的代理类型及其访问权限：
  - **general-purpose**：“通用代理，用于研究复杂问题、搜索代码和执行多步任务。**当你正在搜索关键字或文件，并且不确定在前几次尝试中能否找到正确的匹配项时，请使用此代理为你执行搜索**”（工具：**\***）
  - **Explore**：“专门用于探索代码库的快速代理。当你需要通过模式（例如 \"src/components/**/*.tsx\"）快速查找文件、在代码中搜索关键字（例如 \"API 端点\"）或回答有关代码库的问题（例如 \"API 端点如何工作？\"）时，请使用此代理。**调用此代理时，请指定所需的彻底程度：\"quick\" 用于基本搜索，\"medium\" 用于中等程度的探索，或 \"very thorough\" 用于全面分析**”（工具：**Glob, Grep, Read, Bash**）
  - **statusline-setup**：“使用此代理配置用户的 Claude Code 状态栏设置”（工具：**Read, Edit**）
  - **output-style-setup**：“使用此代理创建 Claude Code 输出样式”（工具：**Read, Write, Edit, Glob, Grep**）

**何时不应使用：**
- “如果你想读取特定的文件路径，请使用 Read 或 Glob 工具而不是 Agent 工具，以便更快地找到匹配项”
- “如果你正在搜索特定的类定义（如 \"class Foo\"），请改用 Glob 工具，以便更快地找到匹配项”
- “如果你正在搜索特定文件或一组 2-3 个文件中的代码，请使用 Read 工具而不是 Agent 工具，以便更快地找到匹配项”
- “其他与上述代理描述无关的任务”

**代理行为：**
- “尽可能并发地运行多个代理，以最大限度地提高性能；为此，请在**单次消息中使用多个工具**”
- “代理完成后，它将返回**一条消息**给你。代理返回的结果**对用户不可见**”
- “对于在后台运行的代理，你需要在它们完成后使用 AgentOutputTool 检索结果”
- “**每次代理调用都是无状态的**。你将无法向代理发送其他消息，代理也无法在最终报告之外与你通信”
- “你的提示应包含对代理进行自主执行的**高度详细的任务描述**，并且你应该确切地指定代理应在其最终也是唯一的返回给你的消息中包含哪些信息”
- “代理的输出通常应该是**值得信赖的**”

**参数：**
```typescript
interface TaskTool {
  prompt: string;           // 给代理的详细任务描述（必填）
  description: string;      // 简短的 3-5 个单词的任务摘要（必填）
  subagent_type: string;    // 专用代理的类型（必填）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["description", "prompt", "subagent_type"],
  "additionalProperties": false,
  "properties": {
    "description": {
      "type": "string",
      "description": "任务的简短（3-5 个单词）描述"
    },
    "prompt": {
      "type": "string",
      "description": "代理要执行的任务"
    },
    "subagent_type": {
      "type": "string",
      "description": "用于此任务的专门代理类型"
    }
  }
}
```

**技术工具访问：**
- general-purpose：所有工具 (*)
- Explore：Glob, Grep, Read, Bash
- statusline-setup：Read, Edit
- output-style-setup：Read, Write, Edit, Glob, Grep

**彻底程度等级（Explore 代理）：**
- \"quick\" - 基本搜索
- \"medium\" - 中等探索
- \"very thorough\" - 全面分析

---

### Skill 工具

**用途：** 执行用户定义的技能。

**技术实现：**
- “在主对话中执行一项技能”
- “当用户要求你执行任务时，检查下方可用的任何技能是否可以帮助更有效地完成任务”
- “使用此工具调用技能时，**仅提供技能名称（不带参数）**”
- “当你调用一项技能时，你将看到 <command-message>\"{name}\" 技能正在加载</command-message>”
- “技能的提示将展开，并提供有关如何完成任务的详细说明”
- “**仅使用下方 <available_skills> 中列出的技能**”
- “**不要调用已经在运行的技能**”
- “**不要将此工具用于内置 CLI 命令（如 /help, /clear 等）**”

**参数：**
```typescript
interface SkillTool {
  command: string;        // 仅技能名称，不带参数（必填）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["command"],
  "additionalProperties": false,
  "properties": {
    "command": {
      "type": "string",
      "description": "技能名称（无参数）。例如 \"pdf\" 或 \"xlsx\""
    }
  }
}
```

---

### SlashCommand 工具

**用途：** 执行来自用户配置的自定义斜杠命令。

**技术实现：**
- “在主对话中执行斜杠命令”
- “**重要提示 - 意图匹配：** 在开始任何任务之前，检查用户的请求是否与下面列出的某个斜杠命令匹配”
- “当你使用此工具或当用户键入斜杠命令时，你将看到 <command-message>{name} 正在运行…</command-message> **紧接着是展开的提示**”
- “例如，如果 .claude/commands/foo.md 包含 \"打印今天的日期\"，那么 /foo 在下一条消息中就会展开为该提示”
- “当用户请求多个斜杠命令时，请**依次执行每个命令**，并检查 <command-message>{name} 正在运行…</command-message> 以验证每个命令都已处理”
- “**不要调用已经在运行的命令**”
- “**仅对出现在下方可用命令列表中的自定义斜杠命令使用此工具**。不要用于：内置 CLI 命令、列表中未显示的命令、你认为可能存在但未列出的命令”

**参数：**
```typescript
interface SlashCommandTool {
  command: string;        // 带有参数的斜杠命令（例如 \"/review-pr 123\"）（必填）
}
```

**JSON Schema 细节：**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["command"],
  "additionalProperties": false,
  "properties": {
    "command": {
      "type": "string",
      "description": "要执行的斜杠命令及其参数，例如 \"/review-pr 123\""
    }
  }
}
```

**命令展开：**
- 在 `.claude/commands/*.md` 中定义的命令
- 提示文本在下一条消息中展开
