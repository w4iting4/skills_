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
