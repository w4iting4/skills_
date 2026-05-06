---
description: 在 /clear 之前用此 skill。引导 Claude 把本次会话精炼成"下次 1 分钟可上手"的简报，写进项目 CLAUDE.md 的"快速恢复"章节，并 commit + push。强调精炼判断而非机械 dump。用法：/handoff [可选：本次主题一句话]
---

你是会话交接的总编辑。任务：把本次 Claude Code 会话精炼成**下次会话 1 分钟可上手**的简报，写进项目根目录 `CLAUDE.md`（必要时也改 `README.md`），commit + push 后用户才放心 `/clear`。

## 输入

$ARGUMENTS

- 有参数：作为主题摘要（写进 commit message）
- 无参数：根据 git log 自己推断主题

## 核心理念

**精炼判断 > 机械 dump**。Hook 自动化只能 append git log，做不出有判断力的总结。三周后翻 CLAUDE.md 看到一堆 commit hash ≠ 能上手。

## 执行步骤

### 第一步：拉数据

```bash
git log origin/$(git rev-parse --abbrev-ref HEAD)..HEAD --oneline
git status --short
git rev-parse --abbrev-ref HEAD
```

`git log origin..HEAD` 为空 → 本会话无变化，告诉用户"无需 handoff，直接 /clear"，跳过所有后续步骤。

### 第二步：分类整理（关键判断 #1）

按四个槽位归类本会话产物：

| 槽位 | 写入 CLAUDE.md 的标准 |
|---|---|
| **A 完成事项** | 一行 = 一个功能模块，附 commit hash；琐碎 fix 合并为 "minor fixes (...)" |
| **B 关键决策** | 满足全部 3 条：① 讨论过且做了选择；② 不在代码注释里；③ 不写下来三个月后会重新讨论。否则不写 |
| **C 踩过的坑** | 根因不在代码里（环境/时序/多进程/配置/外部服务）。代码 bug 进 `history_bugs/` 即可 |
| **D 未完成事项** | 必须有可操作的"下一步"。模糊想法不写 |

某槽位无产物则跳过（不硬凑）。

### 第三步：精炼过滤（关键判断 #2）

3 条压缩规则：

1. **去重**：CLAUDE.md 已有的不重复写
2. **过期**：上次写的"未完成"如果本次完成了，**从列表移除**（不是追加"已完成"标记）
3. **总量上限**：快速恢复段总行数 ≤ 80 行；超过先砍最老的"完成事项"

### 第四步：编辑 CLAUDE.md

在 cwd 找 `CLAUDE.md`。只动 `## 快速恢复` 章节，5 个子段：

- 最新进展（最近 5-10 条 commits + 简短描述）
- 当前生产/部署配置（如变化）
- 关键决策点（追加，不删旧的）
- 踩过的坑（追加，不删旧的）
- 未完成事项（按规则 2 重写整个 list）

**绝不**重写整份 CLAUDE.md。**绝不**自动删除旧的决策/坑（除非用户显式说"那条已废"）。用 Edit 做精确字符串替换。

项目根没 CLAUDE.md → 创建最小骨架（`# CLAUDE.md` + `## 快速恢复` + 5 个子段空 list）。

### 第五步：视情况改 README.md

满足以下任一条件才动：

- 新增/启用核心特性（用户面对的）
- 模块结构变化（新目录 / 新 binary / 新 config 文件）
- 部署配置变化（端口 / 路径 / 环境依赖）

bug fix / 内部重构 / 注释优化 → **跳过 README**。

### 第六步：Commit + Push

**前置检查**：

```bash
git status --short
```

除 CLAUDE.md / README.md 外有别的改动 → **停下问用户**。避免漏 commit 真功能代码 + 只 commit 文档；或意外把 logs / temp 产物连带提交。

**Commit**：

```bash
git add CLAUDE.md
git add README.md   # 仅当第五步动过
git commit -m "docs: pre-clear handoff [本次主题]

- 完成: <bullet 列表>
- 待办: <bullet 列表>
"
```

**Push**：

```bash
git push origin $(git rev-parse --abbrev-ref HEAD)
```

push 失败（网络/权限/远程领先）→ **保留本地 commit，停下问用户**。**绝不** `--force`。

### 第七步：交接报告

```
✓ Handoff 已提交: <commit hash>
✓ 远程已同步: origin/<branch>
✓ CLAUDE.md "快速恢复" 章节 <N> 行（上限 80）
✓ 你可以 /clear 了

下次会话开局提示（如有）：<具体提示>
```

## 关键防呆

- **绝不** `--force push`
- **绝不**重写整份文件，只在章节内做精确 Edit
- **绝不**在 commit message 包含敏感信息（IP / token / 密钥 / 钱包地址 / 域名）
- **绝不**连带 commit logs / .env / 临时产物（`git add` 必须显式列文件）
- 涉及敏感配置（密码 / API key）→ 报告与 commit message 里**仅描述类别**（"修改了某 service 连接配置"），**不要**写具体值
- 当前分支与 origin 同分支同步即可，**不要**自己 checkout 别的分支

## 与已有 skills 的关系

- 与 `review` 互补：review 看代码、handoff 看会话进展
- 与 `mermaid-verify` 独立：架构图变更建议在做完 handoff commit 后单独跑 `/mermaid-verify`
