# lark-doc

飞书云文档 Claude Code Skill — 让 AI 帮你创建、读取和编辑飞书文档。

## 功能

- **创建文档** — 支持 XML 和 Markdown 两种格式
- **读取文档** — 5 种局部读取模式（全文/大纲/区间/关键词/章节），3 种详细度
- **编辑文档** — 8 种精确更新指令（文本替换、块插入、块替换、块删除、块移动等）
- **媒体操作** — 上传/下载/预览文档中的图片和文件
- **搜索** — 搜索云空间文档

## 前置条件

需要安装 [lark-cli](https://www.npmjs.com/package/@larksuite/cli)：

```bash
npm install -g @larksuite/cli
```

首次使用需完成应用配置：

```bash
lark-cli config init
```

## 安装

```bash
npx skills add MarvinLi0216/claude-skills --skill lark-doc
```

## 内置样式规范

本 skill 包含文档样式指南（`references/style/lark-doc-style.md`），核心规则：

| 规则 | 说明 |
|------|------|
| 标题自动编号 | 所有标题使用 `seq="auto" seq-level="auto"`，不手动写序号 |
| 表格表头 | 第一行用 `<th background-color="light-gray">` |
| 表格宽度一致 | 同一文档所有表格列宽之和 = 820px（特殊情况除外） |
| Callout 限制 | 默认不用，整篇上限 2 个 |
| 颜色克制 | 仅 green/red 表示涨跌，其余保持默认 |

## 常用命令

```bash
# 读取文档
lark-cli docs +fetch --api-version v2 --doc "文档URL"

# 读取大纲
lark-cli docs +fetch --api-version v2 --doc "文档URL" --scope outline

# 创建文档
lark-cli docs +create --api-version v2 --content '<title>标题</title><p>内容</p>'

# 追加内容
lark-cli docs +update --api-version v2 --doc "文档URL" --command append --content '<p>新段落</p>'

# 替换文本
lark-cli docs +update --api-version v2 --doc "文档URL" --command str_replace --pattern "旧文本" --content "新文本"

# 替换指定块
lark-cli docs +update --api-version v2 --doc "文档URL" --command block_replace --block-id "块ID" --content '<p>新内容</p>'
```

## 配套 Skills

完整的飞书操作通常需要配合使用：

- `lark-shared` — 认证、权限管理（必需）
- `lark-drive` — 云空间文件管理、评论
- `lark-sheets` — 电子表格操作
- `lark-base` — 多维表格操作
- `lark-whiteboard` — 画板编辑

## License

MIT
