# 6+2+1+1 个股深度分析 Skill

Claude Code / Codex 个股深度分析框架，输出完整研报：Executive Summary、行业、商业模式、管理层、财务、估值、同业对比、投资逻辑、风险、技术面（含量化回测）、未来重大事件。

## 安装

Claude Code：

```bash
/bin/mkdir -p ~/.claude/skills/marvin-6211-analysis
/usr/bin/curl -o ~/.claude/skills/marvin-6211-analysis/SKILL.md \
  https://raw.githubusercontent.com/MarvinLi0216/claude-skills/master/marvin-6211-analysis/SKILL.md
```

Codex（推荐保留 Git 工作副本，便于迭代）：

```bash
/usr/bin/git clone git@github.com:MarvinLi0216/claude-skills.git ~/Documents/Codex/claude-skills
/bin/ln -s ~/Documents/Codex/claude-skills/marvin-6211-analysis ~/.codex/skills/marvin-6211-analysis
```

安装完成后，在下一轮 Claude Code / Codex 对话中即可使用。

## 前置依赖

### 1. Futu OpenD 或 Moomoo OpenD（必需，二选一）

Skill 通过 OpenD 获取实时行情和财务数据，支持：

- Futu OpenD：https://www.futunn.com/download/openAPI
- Moomoo OpenD：https://openapi.moomoo.com/
- 两者默认监听 `127.0.0.1:11111`；如修改过配置，请使用实际 host/port
- 使用对应平台账号登录，实际行情权限取决于账号、地区及订阅

### 2. Python 环境

Futu OpenD：

```bash
/usr/bin/python3 -m pip install futu-api pandas numpy
```

Moomoo OpenD：

```bash
/usr/bin/python3 -m pip install moomoo-api pandas numpy
```

此外还需要本 Skill 引用的 `futuapi` 辅助脚本。Moomoo OpenD 只能替代 OpenD 运行时，不会自动提供这些脚本；辅助脚本导入的 Python 模块必须与当前 SDK 兼容。

最低版本要求：Python 3.8+

### 3. 验证安装

Futu SDK：

```bash
/usr/bin/python3 -c "from futu import OpenQuoteContext; ctx = OpenQuoteContext(host='127.0.0.1', port=11111); print('OK'); ctx.close()"
```

Moomoo SDK：

```bash
/usr/bin/python3 -c "from moomoo import OpenQuoteContext; ctx = OpenQuoteContext(host='127.0.0.1', port=11111); print('OK'); ctx.close()"
```

输出 `OK` 即表示连接正常。

## 使用方法

在 Claude Code 中直接输入以下任意格式：

```
分析 US.AAPL
深度分析 HK.00700
SH.600519 怎么样
腾讯能不能买
NVDA 估值
```

### 支持的市场

| 市场 | 代码格式 | 示例 |
|------|----------|------|
| 美股 | US.XXXX | US.AAPL, US.NVDA |
| 港股 | HK.XXXXX | HK.00700, HK.09988 |
| 沪市 | SH.XXXXXX | SH.600519 |
| 深市 | SZ.XXXXXX | SZ.000001 |

也可以直接输入公司名称（如"苹果"、"茅台"），Skill 会自动匹配代码。

## 输出内容

完整报告包含以下模块：

1. **Executive Summary** — 综合评级、目标价、核心逻辑
2. **行业分析** — 行业规模、增速、竞争格局
3. **商业模式** — 收入结构、护城河、商业飞轮
4. **管理层** — 核心高管、治理结构、股权激励
5. **财务分析** — 三表分析、关键指标趋势
6. **估值分析** — DCF/PE/PB 多维估值，输出目标价
7. **同业对比** — 核心指标横向对比
8. **投资逻辑** — 看多/看空理由
9. **风险提示** — 经营、行业、政策风险
10. **技术面分析** — K线形态、量化回测

## 注意事项

- 分析结果仅供参考，不构成投资建议
- OpenD 免费账户有行情延迟（港股/美股延迟 15 分钟），付费账户可获取实时行情
- 单次分析约消耗 50K-150K output tokens（Opus 模型）

## License

MIT
