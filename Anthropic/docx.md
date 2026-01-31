---
name: docx
description: 全面的文档创建、编辑和分析工具，支持修订记录、注释、格式保留和文本提取
when_to_use: "当 Claude 需要处理专业的 Word 文档（.docx 文件）时使用，包括：(1) 创建新文档，(2) 修改或编辑内容，(3) 处理修订记录，(4) 添加注释，或其他任何文档相关任务"
version: 0.0.1
---

# DOCX 创建、编辑与分析

## 概览

用户可能会要求你创建、编辑或分析 .docx 文件的内容。.docx 文件本质上是一个包含 XML 文件和其他资源的 ZIP 压缩包，你可以读取或编辑其中的内容。针对不同的任务，你可以使用不同的工具和工作流。

## 工作流决策树

### 读取/分析内容
参考下文的“文本提取”或“原始 XML 访问”部分。

### 创建新文档
使用“创建新 Word 文档”工作流。

### 编辑现有文档
- **你自己的文档 + 简单修改**
  使用“基础 OOXML 编辑”工作流。

- **他人的文档**
  使用 **“红线审查工作流 (Redlining workflow)”**（推荐的默认做法）。

- **法律、学术、商业或政府文档**
  使用 **“红线审查工作流 (Redlining workflow)”**（强制要求）。

## 读取与分析内容

### 文本提取
如果你只需要读取文档的文本内容，你应该使用 pandoc 将文档转换为 markdown。Pandoc 能够很好地保留文档结构，并可以显示修订记录：
```bash
# 将文档转换为包含修订记录的 markdown
pandoc --track-changes=all path-to-file.docx -o output.md
# 选项：--track-changes=accept/reject/all
```

### 原始 XML 访问
在处理以下内容时需要访问原始 XML：注释、复杂格式、文档结构、嵌入媒体和元数据。对于这些功能，你需要解压文档并读取其原始 XML 内容。

#### 解压文件
`python ooxml/scripts/unpack.py <office_file> <output_directory>`

#### 关键文件结构
* `word/document.xml` - 主文档内容
* `word/comments.xml` - document.xml 中引用的注释
* `word/media/` - 嵌入的图像和媒体文件
* 修订记录使用 `<w:ins>`（插入）和 `<w:del>`（删除）标签

## 创建新 Word 文档

从头开始创建新的 Word 文档时，请使用 **docx-js**，它允许你使用 JavaScript/TypeScript 创建 Word 文档。

### 工作流
1. **强制要求 - 阅读完整文件**：从头到尾完整阅读 [`docx-js.md`](docx-js.md)（约 500 行）。**阅读此文件时绝不要设置任何范围限制。** 在开始创建文档之前，必须阅读完整文件内容以获取详细语法、关键格式规则和最佳实践。
2. 使用 Document、Paragraph、TextRun 组件创建一个 JavaScript/TypeScript 文件（你可以假设所有依赖已安装，如果没有，请参考下文的依赖部分）。
3. 使用 Packer.toBuffer() 导出为 .docx。

## 编辑现有 Word 文档

编辑现有 Word 文档时，你需要处理原始的 Office Open XML (OOXML) 格式。这涉及解压 .docx 文件、编辑 XML 内容并重新压缩。

### 工作流
1. **强制要求 - 阅读完整文件**：从头到尾完整阅读 [`ooxml.md`](ooxml.md)（约 500 行）。**阅读此文件时绝不要设置任何范围限制。** 在继续操作前，请阅读完整文件内容以获取详细语法、关键验证规则和模式。
2. 解压文档：`python ooxml/scripts/unpack.py <office_file> <output_directory>`
3. 编辑 XML 文件（主要是 `word/document.xml` 和 `word/comments.xml`）。
4. **关键**：每次编辑后立即进行验证，并在继续下一步前修复任何验证错误：`python ooxml/scripts/validate.py <dir> --original <file>`
5. 压缩最终文档：`python ooxml/scripts/pack.py <input_directory> <office_file>`

## 用于文档审查的红线工作流 (Redlining workflow)

此工作流允许你在 OOXML 中实施修订之前，先使用 markdown 规划全面的修订。**关键**：为了实现完整的修订跟进，你必须系统地执行所有更改。

### 全面修订工作流

1. **获取 markdown 表示**：将文档转换为保留修订记录的 markdown：
```bash
   pandoc --track-changes=all path-to-file.docx -o current.md
```

2. **创建全面修订清单**：创建一个包含所有所需更改的详细清单，并按顺序排列任务。
   - 所有任务应使用 `[ ]` 格式起始，标记为未完成项。
   - **不要使用 markdown 行号** —— 它们与 XML 结构不匹配。
   - **务必使用：**
     - 章节/标题编号（例如，“第 3.2 节”、“第 IV 条”）。
     - 段落标识符（如果有编号）。
     - 带有独特上下文的 Grep 模式。
     - 文档结构（例如，“第一段”、“签名栏”）。
   - 示例：`[ ] 第 8 节：将 “30 天” 改为 “60 天” (grep: "notice period of.*days prior")`
   - 请考虑到文本由于格式原因可能会被拆分到多个 `<w:t>` 元素中。
   - 保存为 `revision-checklist.md`。

3. **设置修订基础架构**：
   - 解压文档：`python ooxml/scripts/unpack.py <office_file> <output_directory>`
   - 运行设置脚本：`python skills/docx/scripts/setup_redlining.py <unpacked_directory>`
   - 这将自动完成以下操作：
     - 创建 `word/people.xml`，将 Claude 设为作者 (ID 0)。
     - 更新 `[Content_Types].xml` 以包含 people.xml 内容类型。
     - 更新 `word/_rels/document.xml.rels` 添加 people.xml 关系。
     - 将 `<w:trackRevisions/>` 添加到 `word/settings.xml`。
     - 生成并添加一个随机的 8 位十六进制 RSID（例如 "6CEA06C3"）。
     - 显示生成的 RSID 以供参考。
   - **关键**：记下脚本显示的 RSID —— 你必须在所有修订中使用这个相同的 RSID。

4. **系统地应用清单中的更改**：
   - **强制要求 - 阅读完整文件**：从头到尾完整阅读 [`ooxml.md`](ooxml.md)（约 500 行）。**阅读此文件时绝不要设置任何范围限制。** 特别注意名为“Tracked Change Patterns”的章节。
   - **智能体关键提示**：如果将工作委托给子智能体，每个子智能体在进行任何 XML 编辑之前，也必须阅读 `ooxml.md` 的“Tracked Change Patterns”部分。
   - **按顺序处理清单每一项**：逐行检查修订清单。
   - **使用 grep 定位文本**：使用 grep 在 `word/document.xml` 中找到确切的文本位置。
   - **使用 Read 工具读取上下文**：使用 Read 工具查看每次更改周围完整的 XML 结构。
   - **应用修订**：使用 Edit/MultiEdit 工具进行精确修改。
   - **使用一致的 RSID**：在所有修订中使用第 3 步中获得的同一个 RSID（重要：RSID 属性应放在 `w:r` 标签上，放在 `w:del` 或 `w:ins` 标签上是无效的）。
   - **修订格式**：所有插入使用 `<w:ins w:id="X" w:author="Claude" w:date="...">`，删除使用 `<w:del w:id="X" w:author="Claude" w:date="...">`。

5. **强制要求 - 审查并完成清单**：
   - **验证所有更改**：将文档转换为 markdown，并使用 grep/search 验证每项更改：
```bash
     pandoc --track-changes=all <packed_file.docx> -o verification.md
     grep -E "pattern" verification.md  # 检查每个更新后的术语
```
   - **系统地更新清单**：仅在验证确认更改后，才将项目标记为 [x]。
   - **关键 - 完成任何未完成的任务**：如果有项目未勾选，你必须在继续之前完成它们。
   - **记录未完成项**：记下任何未处理的项目及其具体原因。
   - **确保 100% 完成**：在继续之前，清单中的所有项目必须都标记为 [x]。

6. **最终验证与包装**：
   - 最终验证：`python ooxml/scripts/validate.py <directory> --original <file>`
   - 仅在验证通过后进行压缩：`python ooxml/scripts/pack.py <input_directory> <office_file>`
   - 只有在验证通过且清单 100% 完成时，才认为任务完成。

## 将文档转换为图像

为了直观地分析 Word 文档，请通过以下两个步骤将其转换为图像：

1. **将 DOCX 转换为 PDF**：
```bash
   soffice --headless --convert-to pdf document.docx
```

2. **将 PDF 页面转换为 JPEG 图像**：
```bash
   pdftoppm -jpeg -r 150 document.pdf page
```
这将创建类似 `page-1.jpg`, `page-2.jpg` 等文件。

选项：
- `-r 150`：设置分辨率为 150 DPI（在质量和文件大小之间平衡）。
- `-jpeg`：输出 JPEG 格式（如果需要，使用 `-png` 输出 PNG）。
- `-f N`：开始转换的第一页（例如，`-f 2` 从第 2 页开始）。
- `-l N`：停止转换的最后一页（例如，`-l 5` 停止在第 5 页）。
- `page`：输出文件的文件名前缀。

转换特定范围的示例：
```bash
pdftoppm -jpeg -r 150 -f 2 -l 5 document.pdf page  # 权转换第 2-5 页
```

## 代码风格指南
**重要**：在生成用于 DOCX 操作的代码时：
- 编写简洁的代码
- 避免冗长的变量名和冗余操作
- 避免不必要的打印语句

## 依赖项

所需的依赖项（如果不可用，请安装）：

- **pandoc**：`sudo apt-get install pandoc`（用于文本提取）
- **docx**：`npm install -g docx`（用于创建新文档）
- **LibreOffice**：`sudo apt-get install libreoffice`（用于 PDF 转换）
- **Poppler**：`sudo apt-get install poppler-utils`（用于使用 pdftoppm 将 PDF 转换为图像）
