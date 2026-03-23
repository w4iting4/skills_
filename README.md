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
