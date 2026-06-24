# 文档样式指南

创建或编辑文档时，遵循本指南。核心目标：页面干净、信息密度高、靠结构而非装饰传递层次。

## 一、核心原则

1. **结构即层次**：飞书原生的标题编号、有序列表、表格表头已提供清晰的视觉层次，不需要额外色彩装饰
2. **结论前置**：每章节首句点明要旨，不写过渡废话
3. **少即是多**：能删的都删，留下的每个元素都有明确作用

## 二、必须使用的飞书原生能力

### 标题自动编号

所有层级标题使用 `seq="auto"` + `seq-level="auto"`，让飞书自动维护编号：

```xml
<h1 seq="auto" seq-level="auto">设计原理</h1>
<h2 seq="auto" seq-level="auto">技术选型</h2>
```

绝不手动写「一、」「1.1」「2.1.3」等序号前缀。

### 有序列表

列表项统一用 `<ol>` + `<li>`，不手动编号。

### 表格表头

表格第一行用 `<thead>` + `<th background-color="light-gray">`，仅此一处用背景色。

### 表格宽度

同一篇文档中所有表格宽度保持一致，列宽通过 `<colgroup><col width="N"/></colgroup>` 控制，各列 width 之和为固定标准值（默认 820px）。

```xml
<!-- 2列示例，总宽820 -->
<table>
  <colgroup><col width="180"/><col width="640"/></colgroup>
  ...
</table>

<!-- 4列示例，总宽820 -->
<table>
  <colgroup><col width="100"/><col width="240"/><col width="240"/><col width="240"/></colgroup>
  ...
</table>
```

**特殊情况允许延长**（不受标准宽度约束）：
- 任一列宽度被压缩到 < 70px，导致 2-3 个汉字就需要换行
- 按标准宽度等比缩放后，多数单元格内容需换行超过 3 行才能完整显示

遇到特殊情况时，按内容实际需要设定列宽，总宽可超过标准值。

## 三、元素选择指南

| 场景 | 推荐方案 |
|-|-|
| 3+ 属性的结构化数据 / 指标 | `<table>` |
| 任务清单 / 检查项 | `<checkbox>` |
| 代码片段 | `<pre lang="x">` |
| 引用 | `<blockquote>` |
| 方案对比（真正需要并排时） | `<grid>` 2 列 |
| 简单流程图 / 时序图 | `<whiteboard type="mermaid/plantuml">` |
| 复杂架构图 / 数据图 | **lark-whiteboard** skill |

### 画板意图识别

当内容具备以下特征时，用图比纯文本更高效：

| 内容特征 | 推荐画板类型 |
|-|-|
| 多步骤流程或决策路径 | 流程图 |
| 系统/模块间依赖与交互 | 架构图 |
| 上下级或从属关系 | 组织架构图 |
| 时间线或阶段演进 | 时间线 |
| 数值趋势 | 折线图 / 柱状图 |
| 占比分布 | 饼图 |

判断规则：
- 节点 ≤ 10 → `<whiteboard type="mermaid/plantuml">` 内嵌
- 节点 > 10 或需自定义布局 → spawn Agent 使用 **lark-whiteboard** skill

### 画板语法与插入

> `docs +update` 不能编辑已有画板内容；下面的语法仅用于新增画板块。修改已有画板需切到 [`lark-whiteboard`](../../../lark-whiteboard/SKILL.md)。

简单图直接用 `<whiteboard type="mermaid|plantuml">语法</whiteboard>` 嵌入。

需要复杂结构时：
1. 用 `<whiteboard type="blank"></whiteboard>` 插入空白画板
2. 从响应 `data.document.new_blocks` 中提取画板 `block_token`
3. 切到 [`lark-whiteboard`](../../../lark-whiteboard/SKILL.md) skill 设计并上传 DSL

更完整的协同流程见 [`lark-doc-whiteboard.md`](../lark-doc-whiteboard.md)。

## 四、Callout 使用原则

**默认不用。** 仅在以下场景允许：

| 允许场景 | emoji | 背景色 |
|-|-|-|
| 可能导致数据丢失或安全风险 | ⚠️ | `light-red` |
| 硬性前置条件，跳过会导致后续失败 | ❗ | `light-yellow` |

一篇文档中 callout 上限：**2 个**。超过说明结构有问题，应改用标题分层或列表表达。

## 五、颜色使用原则

- 文字颜色：仅 `green` 表示正向/增长，`red` 表示负向/下降，同时配 ↑↓ 或 +/- 标注方向（色觉无障碍）。其余保持默认黑色
- 背景色：仅表格表头用 `light-gray`，其余不加
- 不使用 `<span background-color>` 做文字高亮
- 不使用多种 callout 背景色区分信息类型
- 不在正文中使用纯装饰性 emoji

## 六、排版规范

- 标题层级 ≤ 3 层（h1/h2/h3），不再往下
- 每段 ≤ 3 句话，能一句说清不用两句
- 列表嵌套 ≤ 2 层
- 不使用 `<hr/>` 分隔线（标题层级自然分隔）
- 删除所有过渡废话：「首先我们来看」「接下来介绍」「下面将详细说明」
- 标题本身概括该节内容，正文不重复标题已表达的信息

## 七、自检清单

生成文档后对照检查：

- [ ] 标题全部使用了 `seq="auto" seq-level="auto"`？
- [ ] callout 数量 ≤ 2？
- [ ] 没有纯装饰性的颜色和 emoji？
- [ ] 每段 ≤ 3 句话？
- [ ] 没有过渡废话和重复表述？
- [ ] 没有手动编号的标题或列表？
- [ ] 所有表格宽度一致（各列 width 之和 = 820px，特殊情况除外）？
