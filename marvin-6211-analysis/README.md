# 6+2+1+1 个股深度分析 Skill

Claude Code 个股深度分析框架，输出完整研报：Executive Summary、行业、商业模式、管理层、财务、估值、同业对比、投资逻辑、风险、技术面（含量化回测）、未来重大事件。

## 安装

```bash
mkdir -p ~/.claude/skills/marvin-6211-analysis
curl -o ~/.claude/skills/marvin-6211-analysis/skill.md \
  https://raw.githubusercontent.com/MarvinLi0216/claude-skills/master/marvin-6211-analysis/skill.md
```

安装完成后重启 Claude Code 即可使用。

## 前置依赖

### 1. Futu OpenD（必需）

Skill 通过 Futu OpenD 获取实时行情和财务数据。

- 下载地址：https://www.futunn.com/download/openAPI
- 安装后启动 OpenD，默认监听 `127.0.0.1:11111`
- 需要富途账号（免费注册即可获取基础行情权限）

### 2. Python 环境

```bash
pip install futu-api pandas numpy
```

最低版本要求：Python 3.8+

### 3. 验证安装

```bash
# 确认 OpenD 正在运行
python -c "from futu import OpenQuoteContext; ctx = OpenQuoteContext(); print('OK'); ctx.close()"
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
