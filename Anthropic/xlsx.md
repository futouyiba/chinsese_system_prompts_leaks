---
name: Excel 电子表格处理器
description: 全面的电子表格创建、编辑和分析工具，支持公式、格式设置、数据分析与可视化
when_to_use: "当 Claude 需要处理电子表格（.xlsx, .xlsm, .csv, .tsv 等）时使用，包括：(1) 创建带有公式和格式的新电子表格，(2) 读取或分析数据，(3) 在保留公式的同时修改现有电子表格，(4) 在电子表格中进行数据分析和可视化，或 (5) 重新计算公式"
version: 0.0.1
dependencies: openpyxl, pandas
---

# 输出要求

## 所有 Excel 文件

### 零公式错误
- 每个交付的 Excel 模型必须确保零公式错误（#REF!, #DIV/0!, #VALUE!, #N/A, #NAME?）

### 保留现有模板（在更新模板时）
- 在修改文件时，研究并精确匹配现有的格式、样式和惯例
- 切勿对已有既定模式的文件强加标准化格式
- 现有模板的惯例始终优先于这些指南

## 财务模型

### 颜色编码标准
除非用户另有说明或存在现有模板。

#### 行业标准颜色惯例
- **蓝色文本 (RGB: 0,0,255)**：硬编码输入，以及用户将在情景分析中更改的数字
- **黑色文本 (RGB: 0,0,0)**：所有公式和计算
- **绿色文本 (RGB: 0,128,0)**：从同一工作簿内的其他工作表提取的链接
- **红色文本 (RGB: 255,0,0)**：链接到其他文件的外部链接
- **黄色背景 (RGB: 255,255,0)**：需要注意的关键假设或需要更新的单元格

### 数值格式标准

#### 必需的格式规则
- **年份**：格式化为文本字符串（例如 "2024" 而非 "2,024"）
- **货币**：使用 $#,##0 格式；始终在表头指定单位（“Revenue ($mm)”）
- **零值**：使用数值格式将所有零显示为 “-”，包括百分比（例如 “$#,##0;($#,##0);-”）
- **百分比**：默认使用 0.0% 格式（保留一位小数）
- **倍数**：估值倍数格式为 0.0x（如 EV/EBITDA, P/E）
- **负数**：使用圆括号 (123) 而非负号 -123

### 公式构建规则

#### 假设值的放置
- 将所有假设值（增长率、利润率、倍数等）放在独立的假设单元格中
- 在公式中使用单元格引用，而非硬编码数值
- 示例：使用 =B5*(1+$B$6) 而非 =B5*1.05

#### 公式错误预防
- 验证所有单元格引用是否正确
- 检查区域引用是否存在“差一”错误 (off-by-one errors)
- 确保所有预测期间的公式保持一致
- 使用边界情况进行测试（零值、负数）
- 验证不存在非预期的循环引用

#### 硬编码值的文档要求
- 在单元格内添加注释或在旁边的单元格（如果是表格末尾）进行说明。格式：“Source: [System/Document], [Date], [Specific Reference], [URL if applicable]”
- 示例：
  - "Source: Company 10-K, FY2024, Page 45, Revenue Note, [SEC EDGAR URL]"
  - "Source: Company 10-Q, Q2 2025, Exhibit 99.1, [SEC EDGAR URL]"
  - "Source: Bloomberg Terminal, 8/15/2025, AAPL US Equity"
  - "Source: FactSet, 8/20/2025, Consensus Estimates Screen"

# XLSX 创建、编辑与分析

## 概览

用户可能会要求你创建、编辑或分析 .xlsx 文件的内容。针对不同的任务，你可以使用不同的工具和工作流。

## 重要要求

**需要使用 LibreOffice 重新计算公式**：你可以假设已安装 LibreOffice，并使用 `recalc.py` 脚本重新计算公式值。该脚本在首次运行时会自动配置 LibreOffice。

## 读取与分析数据

### 使用 pandas 进行数据分析
对于数据分析、可视化和基础操作，请使用功能强大的数据处理库 **pandas**：
```python
import pandas as pd

# 读取 Excel
df = pd.read_excel('file.xlsx')  # 默认读取第一个工作表
all_sheets = pd.read_excel('file.xlsx', sheet_name=None)  # 以字典形式读取所有工作表

# 分析
df.head()      # 预览数据
df.info()      # 列信息
df.describe()  # 统计信息

# 写入 Excel
df.to_excel('output.xlsx', index=False)
```

## Excel 文件工作流

## 关键：使用公式，而非硬编码数值

**始终使用 Excel 公式，而非在 Python 中计算好数值后再硬编码。** 这能确保电子表格具有动态性且可更新。

### ❌ 错误做法 —— 硬编码计算后的数值
```python
# 错误：在 Python 中计算并硬编码结果
total = df['Sales'].sum()
sheet['B10'] = total  # 硬编码为 5000

# 错误：在 Python 中计算增长率
growth = (df.iloc[-1]['Revenue'] - df.iloc[0]['Revenue']) / df.iloc[0]['Revenue']
sheet['C5'] = growth  # 硬编码为 0.15

# 错误：在 Python 中计算平均值
avg = sum(values) / len(values)
sheet['D20'] = avg  # 硬编码为 42.5
```

### ✅ 正确做法 —— 使用 Excel 公式
```python
# 正确：让 Excel 计算总和
sheet['B10'] = '=SUM(B2:B9)'

# 正确：将增长率设为 Excel 公式
sheet['C5'] = '=(C4-C2)/C2'

# 正确：使用 Excel 函数计算平均值
sheet['D20'] = '=AVERAGE(D2:D19)'
```

这适用于所有计算——总计、百分比、比率、差值等。当源数据更改时，电子表格应能够自动重新计算。

## 通用工作流
1. **选择工具**：数据处理用 pandas，公式/格式设置用 openpyxl
2. **创建/加载**：创建新工作簿或加载现有文件
3. **修改**：添加/编辑数据、公式和格式
4. **保存**：写入文件
5. **重新计算公式（如果使用了公式，则为强制要求）**：使用 recalc.py 脚本
```bash
   python recalc.py output.xlsx
```
6. **验证并修复任何错误**：
   - 脚本会返回包含错误详情的 JSON
   - 如果 `status` 为 `errors_found`，检查 `error_summary` 以获取具体的错误类型和位置
   - 修复识别出的错误并再次运行重新计算
   - 需要修复的常见错误：
     - `#REF!`：无效的单元格引用
     - `#DIV/0!`：除以零
     - `#VALUE!`：公式中数据类型错误
     - `#NAME?`：无法识别的公式名称

### 创建新的 Excel 文件
```python
# 使用 openpyxl 进行公式和格式设置
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment

wb = Workbook()
sheet = wb.active

# 添加数据
sheet['A1'] = 'Hello'
sheet['B1'] = 'World'
sheet.append(['Row', 'of', 'data'])

# 添加公式
sheet['B2'] = '=SUM(A1:A10)'

# 格式设置
sheet['A1'].font = Font(bold=True, color='FF0000')
sheet['A1'].fill = PatternFill('solid', start_color='FFFF00')
sheet['A1'].alignment = Alignment(horizontal='center')

# 列宽
sheet.column_dimensions['A'].width = 20

wb.save('output.xlsx')
```

### 编辑现有 Excel 文件
```python
# 使用 openpyxl 以保留公式和格式
from openpyxl import load_workbook

# 加载现有文件
wb = load_workbook('existing.xlsx')
sheet = wb.active  # 或使用 wb['SheetName'] 获取特定工作表

# 处理多个工作表
for sheet_name in wb.sheetnames:
    sheet = wb[sheet_name]
    print(f"Sheet: {sheet_name}")

# 修改单元格
sheet['A1'] = 'New Value'
sheet.insert_rows(2)  # 在第 2 行位置插入行
sheet.delete_cols(3)  # 删除第 3 列

# 添加新工作表
new_sheet = wb.create_sheet('NewSheet')
new_sheet['A1'] = 'Data'

wb.save('modified.xlsx')
```

## 重新计算公式

使用 openpyxl 创建或修改的 Excel 文件包含字符串形式的公式，但不包含计算后的数值。使用提供的 `recalc.py` 脚本来重新计算公式：
```bash
python recalc.py <excel_file> [timeout_seconds]
```

示例：
```bash
python recalc.py output.xlsx 30
```

该脚本将：
- 在首次运行时自动设置 LibreOffice 宏
- 重新计算所有工作表中的所有公式
- 扫描所有单元格以查找 Excel 错误（#REF!, #DIV/0! 等）
- 返回包含详细错误位置和计数的 JSON
- 支持 Linux 和 macOS

## 公式验证清单

快速检查以确保公式正确运行：

### 核心验证
- [ ] **测试 2-3 个样本引用**：在构建完整模型前，验证它们是否提取了正确的值
- [ ] **列映射确认**：确认 Excel 列是否匹配（例如，第 64 列 = BL，而不是 BK）
- [ ] **行偏移量**：记住 Excel 行是从 1 开始索引的（DataFrame 第 5 行 = Excel 第 6 行）

### 常见陷阱
- [ ] **NaN 处理**：使用 `pd.notna()` 检查空值
- [ ] **极右侧列**：财年 (FY) 数据通常在 50 列之后
- [ ] **多重匹配**：搜索所有出现的位置，而不仅仅是第一个
- [ ] **除以零**：在公式中使用 `/` 前先检查分母 (#DIV/0!)
- [ ] **错误引用**：验证所有单元格引用是否指向了目标单元格 (#REF!)
- [ ] **跨表引用**：使用正确的格式 (Sheet1!A1) 进行跨表链接

### 公式测试策略
- [ ] **从小处着手**：在广泛应用前，先在 2-3 个单元格上测试公式
- [ ] **验证依赖项**：检查公式中引用的所有单元格是否存在
- [ ] **测试边界情况**：包括零、负数和极大值

### 解释 recalc.py 的输出
脚本返回包含错误详情的 JSON：
```json
{
  "status": "success",           // 或 "errors_found"
  "total_errors": 0,              // 错误总数
  "total_formulas": 42,           // 文件中的公式总数
  "error_summary": {              // 仅在发现错误时存在
    "#REF!": {
      "count": 2,
      "locations": ["Sheet1!B5", "Sheet1!C10"]
    }
  }
}
```

## 最佳实践

### 库的选择
- **pandas**：最适合数据分析、批量操作和简单的数据导出
- **openpyxl**：最适合复杂的格式设置、公式处理和 Excel 特有功能

### 使用 openpyxl 的注意事项
- 单元格索引是从 1 开始的（row=1, column=1 指 A1 单元格）
- 使用 `data_only=True` 来读取计算后的数值：`load_workbook('file.xlsx', data_only=True)`
- **警告**：如果以 `data_only=True` 打开并进行保存，公式将被替换为数值并永久丢失
- 处理大文件：使用 `read_only=True` 进行读取，或使用 `write_only=True` 进行写入
- 公式会被保留但不会被求值 —— 请使用 recalc.py 更新数值

### 使用 pandas 的注意事项
- 指定数据类型以避免推断问题：`pd.read_excel('file.xlsx', dtype={'id': str})`
- 处理大文件时，读取特定列：`pd.read_excel('file.xlsx', usecols=['A', 'C', 'E'])`
- 正确处理日期：`pd.read_excel('file.xlsx', parse_dates=['date_column'])`

## 代码风格指南
**重要**：在生成用于 Excel 操作的 Python 代码时：
- 编写最简、最精炼的 Python 代码，不带多余注释
- 避免冗长的变量名和冗余操作
- 避免不必要的打印语句

**对于 Excel 文件本身**：
- 为带有复杂公式或重要假设的单元格添加注释
- 为硬编码值记录数据来源
- 为关键计算和模型部分添加说明文字
