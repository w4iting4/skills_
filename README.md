# skills_

阿鸡的 skills 包

## Skills 列表

### review — 代码审查 & Mermaid 可视化

审查代码逻辑，自动生成 Mermaid 图表，以可视化方式理解代码运行流程。

| 版本 | 文件 | 说明 |
|------|------|------|
| 中文 | `review/review-zh.md` | 提示词和输出均为中文 |
| English | `review/review-en.md` | 全英文版本 |

#### 安装

```bash
# 在目标项目中创建 commands 目录（如不存在）
mkdir -p your-project/.claude/commands

# 复制中文版
cp review/review-zh.md your-project/.claude/commands/review.md

# 或复制英文版
cp review/review-en.md your-project/.claude/commands/review.md
```

#### 使用

```
/review src/auth        # 审查某个模块
/review all             # 审查整个项目
```

### handoff — 会话交接 / pre-clear 简报

在 `/clear` 之前用此 skill。引导 Claude 把本次会话精炼成"下次 1 分钟可上手"的简报，写进项目 `CLAUDE.md` 的"快速恢复"章节，必要时也改 `README.md`，然后 commit + push。强调精炼判断而非机械 dump（hook 自动化只能 append git log，做不出有判断力的总结）。

| 版本 | 文件 | 说明 |
|------|------|------|
| 中文 | `handoff/handoff-zh.md` | 提示词和输出均为中文 |
| English | `handoff/handoff-en.md` | 全英文版本 |

#### 安装

```bash
mkdir -p your-project/.claude/commands

# 中文版
cp handoff/handoff-zh.md your-project/.claude/commands/handoff.md

# 或英文版
cp handoff/handoff-en.md your-project/.claude/commands/handoff.md
```

#### 使用

```
/handoff                       # 自动推断主题
/handoff "shred fix + 风控接入"  # 显式给主题（写进 commit message）
```

跑完后敲 `/clear` 即可，下次开局看 `CLAUDE.md` 的"快速恢复"段秒速恢复上下文。

### mermaid-verify — Mermaid 图表质量守门员

检查 mermaid 源码能否在 mermaid.live 稳定渲染，自动修复常见陷阱（subgraph alias / HTML entity / 引号嵌套 / 行尾空格 / 替换字符等），并用本地 jsdom + mermaid.parse() + mermaid.render() 双重验证。

| 版本 | 文件 | 说明 |
|------|------|------|
| 中文 | `mermaid-verify/mermaid-verify-zh.md` | 提示词和输出均为中文 |
| English | `mermaid-verify/mermaid-verify-en.md` | 全英文版本 |

#### 安装

```bash
mkdir -p your-project/.claude/commands

# 中文版
cp mermaid-verify/mermaid-verify-zh.md your-project/.claude/commands/mermaid-verify.md

# 或英文版
cp mermaid-verify/mermaid-verify-en.md your-project/.claude/commands/mermaid-verify.md
```

#### 使用

```
/mermaid-verify path/to/diagram.mmd     # 检查文件
/mermaid-verify <粘贴一段 mermaid 源码>  # 检查源码
```
