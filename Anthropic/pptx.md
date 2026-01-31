# PowerPoint 组件 (/mnt/skills/public/pptx/SKILL.md)

---

* name: PowerPoint 组件  

* description: 演示文稿的创建、编辑和分析。  

* when_to_use: "当 Claude 需要处理演示文稿（.pptx 文件）时使用，包括：(1) 创建新演示文稿，(2) 修改或编辑内容，(3) 处理布局，(4) 添加注释或演讲者备注，或其他任何演示文稿任务"  

* version: 0.0.3  

---

# PPTX 创建、编辑与分析

## 概览

用户可能会要求你创建、编辑或分析 .pptx 文件的内容。.pptx 文件本质上是一个包含 XML 文件和其他资源的 ZIP 压缩包，你可以读取或编辑其中的内容。针对不同的任务，你可以使用不同的工具和工作流。

## 读取与分析内容

### 文本提取
如果你只需要读取演示文稿的文本内容，你应该将其转换为 markdown：
```bash
# 将文档转换为 markdown
python -m markitdown path-to-file.pptx
```

### 原始 XML 访问
在处理以下内容时需要访问原始 XML：注释、演讲者备注、幻灯片布局、动画、设计元素和复杂格式。对于这些功能，你需要解压演示文稿并读取其原始 XML 内容。

#### 解压文件
`python ooxml/scripts/unpack.py <office_file> <output_dir>`

**注意**：unpack.py 脚本相对于项目根目录的位置在 `skills/pptx/ooxml/scripts/unpack.py`。如果该路径下不存在该脚本，请使用 `find . -name "unpack.py"` 进行定位。

#### 关键文件结构
* `ppt/presentation.xml` - 主演示文稿元数据和幻灯片引用
* `ppt/slides/slide{N}.xml` - 单个幻灯片内容 (slide1.xml, slide2.xml 等)
* `ppt/notesSlides/notesSlide{N}.xml` - 每张幻灯片的演讲者备注
* `ppt/comments/modernComment_*.xml` - 特定幻灯片的注释
* `ppt/slideLayouts/` - 幻灯片布局模板
* `ppt/slideMasters/` - 母版幻灯片模板
* `ppt/theme/` - 主题和样式信息
* `ppt/media/` - 图像和其他媒体文件

#### 字体排版与颜色提取
**当给定示例设计进行模仿时**：始终先使用以下方法分析演示文稿的排版和颜色：
1. **读取主题文件**：检查 `ppt/theme/theme1.xml` 中的颜色 (`<a:clrScheme>`) 和字体 (`<a:fontScheme>`)
2. **采样幻灯片内容**：检查 `ppt/slides/slide1.xml` 中的实际字体使用 (`<a:rPr>`) 和颜色
3. **搜索模式**：使用 grep 在所有 XML 文件中查找颜色 (`<a:solidFill>`, `<a:srgbClr>`) 和字体引用

## 在**没有模板**的情况下创建新 PowerPoint 演示文稿

从头开始创建新的 PowerPoint 演示文稿时，请使用 **html2pptx** 工作流将 HTML 幻灯片转换为具有精确位置的 PowerPoint。

### 设计原则

**关键**：在创建任何演示文稿之前，分析内容并选择合适的设计元素：
1. **考虑主题内容**：这份演示文稿是关于什么的？它暗示了什么样的语调、行业或氛围？
2. **检查品牌标示**：如果用户提到了某家公司/组织，请考虑其品牌颜色和形象
3. **匹配调色板与内容**：选择能反映主题的颜色
4. **陈述你的方案**：在编写代码前解释你的设计选择

**要求**：
- ✅ 在编写代码前，陈述你基于内容的设计方案
- ✅ 仅使用 Web 安全字体：Arial, Helvetica, Times New Roman, Georgia, Courier New, Verdana, Tahoma, Trebuchet MS, Impact
- ✅ 通过大小、粗细和颜色建立清晰的视觉层级
- ✅ 确保可读性：强对比、大小合适的文本、整洁的对齐
- ✅ 保持一致性：在所有幻灯片中重复使用模式、间距和视觉语言

#### 调色板选择

**有创意地选择颜色**：
- **跳出默认思维**：哪些颜色能真正契合这个特定主题？避免自动化选择。
- **多角度考虑**：主题、行业、氛围、能量水平、目标受众、品牌形象（如果提及）
- **大胆尝试**：尝试意想不到的组合 —— 医疗演示文稿不一定非要是绿色的，金融不一定非要是深蓝色的
- **构建你的调色板**：挑选 3-5 种相互配合的颜色（主色 + 辅助色 + 强调色）
- **确保对比度**：背景上的文本必须清晰可读

**对比调色板示例**（用于激发创意 —— 可直接选用、改编或自行创建）：

1. **经典蓝**：深蓝 (#1C2833), 石板灰 (#2E4053), 银色 (#AAB7B8), 乳白 (#F4F6F6)
2. **青蓝与珊瑚红**：青蓝 (#5EA8A7), 深青 (#277884), 珊瑚红 (#FE4447), 白色 (#FFFFFF)
3. **醒目红**：暗红 (#C0392B), 鲜红 (#E74C3C), 橙色 (#F39C12), 黄色 (#F1C40F), 绿色 (#2ECC71)
4. **温馨藕粉**：锦葵紫 (#A49393), 藕粉 (#EED6D3), 玫瑰色 (#E8B4B8), 奶油色 (#FAF7F2)
5. **华丽酒红**：酒红 (#5D1D2E), 深红 (#951233), 铁锈红 (#C15937), 金色 (#997929)
6. **深紫与翡翠绿**：紫色 (#B165FB), 深蓝 (#181B24), 翡翠绿 (#40695B), 白色 (#FFFFFF)
7. **奶油与森林绿**：奶油色 (#FFE1C7), 森林绿 (#40695B), 白色 (#FCFCFC)
8. **粉与紫**：粉红 (#F8275B), 珊瑚色 (#FF574A), 玫瑰色 (#FF737D), 紫色 (#3D2F68)
9. **青柠与梅紫**：青柠 (#C5DE82), 梅紫 (#7C3A5F), 珊瑚色 (#FD8C6E), 蓝灰色 (#98ACB5)
10. **黑与金**：金色 (#BF9A4A), 黑色 (#000000), 乳白 (#F4F6F6)
11. **鼠尾草与赤陶**：鼠尾草绿 (#87A96B), 赤陶色 (#E07A5F), 奶油色 (#F4F1DE), 炭黑 (#2C2C2C)
12. **炭灰与红**：炭灰 (#292929), 红色 (#E33737), 浅灰 (#CCCBCB)
13. **活力橙**：橙色 (#F96D00), 浅灰 (#F2F2F2), 炭黑 (#222831)
14. **浓郁森林绿**：黑色 (#191A19), 绿色 (#4E9F3D), 深绿 (#1E5128), 白色 (#FFFFFF)
15. **复古彩虹**：紫色 (#722880), 粉红 (#D72D51), 橙色 (#EB5C18), 琥珀色 (#F08800), 金色 (#DEB600)
16. **复古大地色**：芥末黄 (#E3B448), 鼠尾草色 (#CBD18F), 森林绿 (#3A6B35), 奶油色 (#F4F1DE)
17. **海滨玫瑰**：尘玫瑰 (#AD7670), 浅褐 (#B49886), 蛋壳色 (#F3ECDC), 灰蓝 (#BFD5BE)
18. **橙与土耳其蓝**：浅橙 (#FC993E), 灰蓝绿 (#667C6F), 白色 (#FCFCFC)

#### 视觉细节选项

**几何图案**：
- 使用对角线部分分隔符而非水平分隔符
- 非对称列宽（30/70, 40/60, 25/75）
- 将标题文字旋转 90° 或 270°
- 为图像使用圆形/六边形边框
- 在角落使用三角形装饰形状
- 形状重叠以增加层次感

**边框与框架处理**：
- 仅在单侧使用粗的单色边框 (10-20pt)
- 使用对比色的双线边框
- 使用角括号而非全框
- L 型边框（顶部+左侧 或 底部+右侧）
- 在页眉下方添加下划线装饰 (3-5pt 粗)

**排版处理**：
- 极端的尺寸对比（72pt 标题 vs 11pt 正文）
- 使用宽字母间距的全大写标题
- 以超大号显示字体标记章节编号
- 为数据/统计/技术内容使用等宽字体 (Courier New)
- 为高密度信息使用窄字体 (Arial Narrow)
- 使用轮廓文字进行强调

**图表与数据样式**：
- 单色图表，关键数据使用单一强调色
- 使用水平条形图而非垂直柱状图
- 使用点图而非条形图
- 极简化网格线或完全不使用
- 数据标签直接标注在元素上（无图例）
- 为关键指标使用超大数字

**布局创新**：
- 带文字叠加的全幅无边框图像
- 使用侧边栏（20-30% 宽度）进行导航/背景说明
- 模块化网格系统 (3×3, 4×4 块)
- Z 型或 F 型内容流动逻辑
- 在彩色形状上漂浮文本框
- 杂志风格的多栏布局

**背景处理**：
- 占据幻灯片 40-60% 的纯色块
- 渐变填充（仅限垂直或对角线方向）
- 分割背景（两种颜色，对角线或垂直分割）
- 贯穿两边的色带
- 将负空间作为设计元素

### 布局贴士
**当创建包含图表或表格的幻灯片时：**
- **双栏布局（首选）**：页眉横跨全屏，下方分为两栏 —— 一栏放置文本/要点，另一栏放置主要内容。这能提供更好的平衡感，使图表/表格更易读。使用不均等的 Flexbox 列宽（如 40%/60% 拆分）来优化各类内容的空间。
- **全屏布局**：让主要内容（图表/表格）占据整张幻灯片，以获得最大的冲击力和可读性。
- **切勿垂直堆叠**：不要在单栏中将图表/表格放在文本下方 —— 这会导致可读性差且布局混乱。

### 工作流
1. **强制要求 - 阅读完整文件**：从头到尾完整阅读 [`html2pptx.md`](html2pptx.md)。**阅读此文件时绝不要设置任何范围限制。** 在继续创建演示文稿前，请阅读完整文件内容以获取详细语法、关键格式规则和最佳实践。
2. 为每张幻灯片创建一个符合尺寸的 HTML 文件（例如 16:9 比例下的 720pt × 405pt）
   - 为所有文本内容使用 `<p>`, `<h1>`-`<h6>`, `<ul>`, `<ol>`
   - 为将要添加图表/表格的区域使用 `class="placeholder"`（渲染时使用灰色背景以便识别）
   - **关键**：首先使用 Sharp 将渐变和图标栅格化为 PNG 图像，然后在 HTML 中引用
   - **布局**：对于带有图表/表格/图像的幻灯片，使用全屏布局或双栏布局以提高可读性
3. 创建并运行一个使用 [`html2pptx.js`](scripts/html2pptx.js) 库的 JavaScript 文件，将 HTML 幻灯片转换为 PowerPoint 并保存
   - 使用 `html2pptx()` 函数处理每个 HTML 文件
   - 使用 PptxGenJS API 将图表和表格添加到占位符区域
   - 使用 `pptx.writeFile()` 保存演示文稿
4. **可视化验证**：生成缩略图并检查布局问题
   - 创建缩略图网格：`python scripts/thumbnail.py output.pptx workspace/thumbnails --cols 4`
   - 阅读并仔细检查缩略图，查看是否存在：
     * 文字溢出或截断
     * 元素未对齐
     * 颜色或字体错误
     * 内容缺失
     * 布局问题
   - 如果发现问题，在继续操作前进行诊断并修复

## **基于模板**创建新 PowerPoint 演示文稿

当给定 PowerPoint 模板时，你可以通过替换模板幻灯片中的文本内容来创建新的演示文稿。

### 工作流

1. **解压模板**：提取模板的 XML 结构
```bash
   python ooxml/scripts/unpack.py template.pptx unpacked_template
```

2. **读取演示文稿结构**：读取 `unpacked_template/ppt/presentation.xml` 以了解整体结构和幻灯片引用

3. **检查模板幻灯片**：检查前几张幻灯片的 XML 文件以了解其结构
```bash
   # 查看幻灯片结构
   python -c "from lxml import etree; tree = etree.parse('unpacked_template/ppt/slides/slide1.xml'); print(etree.tostring(tree, pretty_print=True, encoding='unicode'))"
```

4. **复制模板到工作文件**：复制一份模板用于编辑
```bash
   cp template.pptx working.pptx
```

5. **生成文本形状清单 (Inventory)**：
```bash
   python scripts/inventory.py working.pptx > template-inventory.json
```
   
   清单提供了演示文稿中所有文本形状的结构化视图：
```json
   {
     "slide-0": {
       "shape-0": {
         "shape_id": "2",
         "shape_name": "Title 1",
         "placeholder_type": "TITLE",
         "text_content": "Original title text here...",
         "default_font_size": 44.0,
         "default_font_name": "Calibri Light"
       },
       "shape-1": {
         "shape_id": "3",
         "shape_name": "Content Placeholder 2",
         "placeholder_type": "BODY",
         "text_content": "Original content text...",
         "default_font_size": 18.0
       }
     },
     "slide-1": {
       ...
     }
   }
```
   
   **理解清单**：
   - 每张幻灯片被标识为 "slide-N"（从 0 开始索引）
   - 幻灯片内的每个文本形状被标识为 "shape-N"（根据出现顺序从 0 开始索引）
   - `placeholder_type` 指示形状的角色：TITLE（标题）、BODY（正文）、SUBTITLE（副标题）等。
   - `text_content` 显示当前文本（用于识别需要替换哪个形状）
   - `default_font_size` 和 `default_font_name` 显示形状的默认格式

6. **创建替换文本 JSON**：根据清单创建一个 JSON 文件，指定要用新文本更新哪些形状
   - **重要**：使用清单中的幻灯片和形状标识符（如 "slide-0", "shape-1"）进行引用
   - **关键**：每个形状的 "paragraphs" 字段必须包含**格式正确的段落对象**，而非纯文本字符串
   - 每个段落对象可以包含：
     - `text`: 实际文本内容（必填）
     - `alignment`: 文本对齐方式（如 "CENTER", "LEFT", "RIGHT"）
     - `bold`: 布尔值，是否加粗
     - `italic`: 布尔值，是否斜体
     - `bullet`: 布尔值，是否启用项目符号（为 true 时，`level` 也是必填项）
     - `level`: 整数，项目符号缩进级别（0 = 无缩进，1 = 第一级缩进，以此类推）
     - `font_size`: 浮点数，自定义字体大小
     - `font_name`: 字符串，自定义字体名称
     - `color`: 字符串，RGB 颜色（如红色为 "FF0000"）
     - `theme_color`: 字符串，基于主题的颜色（如 "DARK_1", "ACCENT_1"）
   - **重要**：当 bullet: true 时，文字内部不要包含项目符号（•, -, *）—— 它们会被自动添加
   - **核心格式规则**：
     - 页眉/标题通常应设置 `"bold": true`
     - 列表项应设置 `"bullet": true, "level": 0`（由 bullet 为 true 时 level 必填）
     - 保留所有对齐属性（例如居中对齐使用 `"alignment": "CENTER"`）
     - 当与默认不同时包含字体属性（例如 `"font_size": 14.0`, `"font_name": "Lora"`）
     - 颜色：使用 `"color": "FF0000"` 表示 RGB，或使用 `"theme_color": "DARK_1"` 表示主题颜色
     - 替换脚本期望接收的是**格式正确段落**，而不仅仅是文本字符串
     - **重叠形状**：优先选择具有较大 `default_font_size` 或更合适 `placeholder_type` 的形状
   - 将更新后的内容保存为 `replacement-text.json`
   - **警告**：不同的模板布局具有不同的形状数量 —— 在创建替换内容前务必核对实际清单

   格式正确的 paragraphs 字段示例：
```json
   "paragraphs": [
     {
       "text": "新的演示文稿标题文本",
       "alignment": "CENTER",
       "bold": true
     },
     {
       "text": "章节页眉",
       "bold": true
     },
     {
       "text": "第一个列表项，不含符号",
       "bullet": true,
       "level": 0
     },
     {
       "text": "红色文本",
       "color": "FF0000"
     },
     {
       "text": "主题色文本",
       "theme_color": "DARK_1"
     },
     {
       "text": "不带特殊格式的普通段落"
     }
   ]
```

   **替换 JSON 中未列出的形状将被自动清空**：
```json
   {
     "slide-0": {
       "shape-0": {
         "paragraphs": [...] // 该形状被填入新文本
       }
       // 清单中的 shape-1 和 shape-2 将被自动清空
     }
   }
```

   **演示文稿的常见格式模式**：
   - 标题幻灯片：粗体，有时居中
   - 幻灯片内的章节页眉：粗体
   - 项目符号列表：每项都需要 `"bullet": true, "level": 0`
   - 正文文本：通常不需要特殊属性
   - 引用：可能具有特殊的对齐或字体属性

7. **使用 `replace.py` 脚本应用替换**
```bash
   python scripts/replace.py working.pptx replacement-text.json output.pptx
```

   该脚本将：
   - 首先调用 inventory.py 中的函数提取所有文本形状清单
   - 验证替换 JSON 中的所有形状在清单中是否存在
   - 清空清单中识别出的所有形状的文本
   - 仅对替换 JSON 中定义了 "paragraphs" 的形状应用新文本
   - 通过应用 JSON 中的段落属性来保留格式
   - 自动处理项目符号、对齐、字体属性和颜色
   - 保存更新后的演示文稿

   验证错误示例：
```
   ERROR: 替换 JSON 中包含无效形状：
     - 'slide-0' 中未找到形状 'shape-99'。可用形状：shape-0, shape-1, shape-4
     - 清单中未找到幻灯片 'slide-999'
```
```
   ERROR: 替换文本使溢出情况恶化：
     - slide-0/shape-2: 溢出恶化 1.25" (原为 0.00", 现为 1.25")
```

## 创建缩略图网格

若要创建 PowerPoint 幻灯片的视觉缩略图网格以便快速分析和参考：
```bash
python scripts/thumbnail.py template.pptx [output_prefix]
```

**功能**：
- 生成文件：`thumbnails.jpg`（大篇幅演示文稿可能会生成 `thumbnails-1.jpg`, `thumbnails-2.jpg` 等）
- 默认：5 列，每张网格最多 30 张幻灯片 (5×6)
- 自定义前缀：`python scripts/thumbnail.py template.pptx my-grid`
  - 注意：输出前缀应包含路径，如果你希望输出到特定目录（如 `workspace/my-grid`）
- 调整列数：`--cols 4`（范围：3-6，影响每张网格的幻灯片数量）
- 网格限制：3 列 = 12 张/网格, 4 列 = 20 张, 5 列 = 30 张, 6 列 = 42 张
- 幻灯片从 0 开始索引（Slide 0, Slide 1 等）

**使用案例**：
- 模板分析：快速了解幻灯片布局和设计模式
- 内容审查：整个演示文稿的视觉概览
- 导航参考：通过外观查找特定幻灯片
- 质量检查：验证所有幻灯片格式是否正确

**示例**：
```bash
# 基础用法
python scripts/thumbnail.py presentation.pptx

# 组合选项：自定义名称、列数
python scripts/thumbnail.py template.pptx analysis --cols 4
```

## 将幻灯片转换为图像

为了直观地分析 PowerPoint 幻灯片，请通过以下两个步骤将其转换为图像：

1. **将 PPTX 转换为 PDF**：
```bash
   soffice --headless --convert-to pdf template.pptx
```

2. **将 PDF 页面转换为 JPEG 图像**：
```bash
   pdftoppm -jpeg -r 150 template.pdf slide
```
这将创建类似 `slide-1.jpg`, `slide-2.jpg` 等文件。

选项：
- `-r 150`：设置分辨率为 150 DPI（在质量和文件大小之间平衡）。
- `-jpeg`：输出 JPEG 格式（如果需要，使用 `-png` 输出 PNG）。
- `-f N`：开始转换的第一页（例如，`-f 2` 从第 2 页开始）。
- `-l N`：停止转换的最后一页（例如，`-l 5` 停止在第 5 页）。
- `slide`：输出文件的文件名前缀。

转换特定范围的示例：
```bash
pdftoppm -jpeg -r 150 -f 2 -l 5 template.pdf slide  # 仅转换第 2-5 页
```

## 代码风格指南
**重要**：在生成用于 PPTX 操作的代码时：
- 编写简洁的代码
- 避免冗长的变量名和冗余操作
- 避免不必要的打印语句

## 依赖项

所需的依赖项（应已安装）：

- **markitdown**：`pip install "markitdown[pptx]"`（用于从演示文稿中提取文本）
- **pptxgenjs**：`npm install -g pptxgenjs`（用于通过 html2pptx 创建演示文稿）
- **playwright**：`npm install -g playwright`（用于 html2pptx 中的 HTML 渲染）
- **react-icons**：`npm install -g react-icons react react-dom`（用于图标）
- **sharp**：`npm install -g sharp`（用于 SVG 栅格化及图像处理）
- **LibreOffice**：`sudo apt-get install libreoffice`（用于 PDF 转换）
- **Poppler**：`sudo apt-get install poppler-utils`（用于使用 pdftoppm 将 PDF 转换为图像）
