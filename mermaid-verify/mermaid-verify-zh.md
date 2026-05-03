---
description: 检查 mermaid 源码能否在 mermaid.live 稳定渲染，自动修复常见陷阱并本地双重验证。用法：/mermaid-verify <mermaid源码 或 .mmd文件路径>
---

你是 mermaid 图表的质量守门员。任务：把任何输入变成"复制粘贴到 mermaid.live 必能渲染"的版本。

## 输入

$ARGUMENTS

- 是 `.mmd` 或 `.md` 文件路径 → 用 Read 读全文
- 是源码（含 `flowchart` / `sequenceDiagram` / `classDiagram` / `stateDiagram` / `erDiagram` 等关键字开头）→ 直接处理
- 既不是路径也不像代码 → 提示用户重新输入

## 执行步骤

### 第一步：静态检查（必修陷阱）

按下面清单逐条扫描并自动修复：

#### 1. subgraph 双参数 alias 语法

某些 mermaid 版本（尤其是 mermaid.live 部分版本）不支持 `subgraph ID [Title]` 这种 id+alias 写法，报：
```
Expecting 'SEMI', 'NEWLINE', 'EOF', got 'SPACE'
```

**修复**：改成单参数 `subgraph TITLE`（同一个标识同时作 id 和显示名）：
```
✗  subgraph cfg [配置层]
✓  subgraph 配置层
```

如果原 id 在边连接里被引用过（`cfg --> something`），把所有引用同步改成新名。

#### 2. HTML entity 在剪贴板被还原

`&amp;` `&gt;` `&lt;` 经过某些工具复制时会还原为 `&` `>` `<`，破坏未转义的内容。

**修复**：直接用纯字符。在 quoted label `"..."` 内可以直接写 `&` `>` `<`，mermaid 不会拦：
```
✗  A["foo &amp; bar"]
✓  A["foo & bar"]
```

#### 3. 节点标签内嵌套引号

```
✗  UNI[("text"<br/>"more text")]
```
解析报 `got 'STR'`。

**修复**：一对外层引号包裹整段，内部用 `<br/>` 换行不再加引号：
```
✓  UNI[("text<br/>more text")]
```

#### 4. 行尾多余空格 / 全角空格

剪贴板有时会把空格替换成全角字符（U+3000）或在行尾加多余空白，触发解析错误。

**修复**：扫到 subgraph / 节点定义行时，去掉行尾所有空白；把全角空格替换为半角。

#### 5. 替换字符 (U+FFFD `�`) / 截断

终端长行复制可能丢字符或出现 `�`。

**修复**：检测到 `�` 或行尾不完整（缺右括号 / 引号不闭合）时，明确告诉用户"输入被剪贴板损坏"，请求重发。

### 第二步：建议优化（可选，告知用户）

下面这些不是错误，但能让图更易读。检查后**告诉用户**这些建议并询问是否应用：

1. **默认曲线乱**：没设置 `curve` 时 mermaid 用 `basis` 平滑曲线，节点多时线条交叉乱视。建议加：
   ```
   %%{init: {"flowchart": {"curve": "stepBefore"}}}%%
   ```
   可选值：`stepBefore` / `stepAfter` / `step` / `linear` / `basis`。

2. **副作用线穿过主流**：日志、监控、只读访问等"副作用"边如果用实线 `-->`，跟主数据流的实线难以区分。建议：
   - 主流用实线 `-->`
   - 副作用用虚线 `-.->`

3. **日志/旁路节点散落**：聚到一个底部 `subgraph logs` 用 `direction LR` 横向排列，所有 `task → log` 的边都用虚线，主流就不会跟日志线交叉。

4. **嵌套 subgraph 分层**：复杂图用嵌套 subgraph 按职责分层（数据源 / 接入 / 处理 / 输出 / 副作用），每层一个 subgraph。

5. **配色分组**：用 `classDef` 给同类节点上色：
   ```
   classDef sourceStyle fill:#d4edda,stroke:#28a745,stroke-width:2px
   classDef coreStyle   fill:#cfe2ff,stroke:#0d6efd,stroke-width:2px
   classDef logStyle    fill:#f0f0f0,stroke:#6c757d,stroke-width:1px
   class A,B,C sourceStyle
   class D,E,F coreStyle
   ```

6. **节点形状的语义约定**：
   ```
   A["text"]      矩形 — task / process / consumer
   B[("text")]    圆柱 — channel / queue / database / store
   C{"text"}      菱形 — decision / branch
   D(["text"])    胶囊 — start / end
   ```

### 第三步：本地双重验证

修复后用 jsdom 跑 `mermaid.parse()` + `mermaid.render()`。两个都过才算通过。

#### 准备（只需一次）

```bash
mkdir -p /tmp/mmd-verify && cd /tmp/mmd-verify
[ -f package.json ] || npm init -y > /dev/null
npm install --silent mermaid jsdom dompurify 2>&1 | tail -3
```

如果 `npm` 不可用 / 安装失败 → 跳过本地验证，仅输出静态修复版 + 提示"环境无 node，未做渲染验证"。

#### 写两个验证脚本

`/tmp/mmd-verify/check.mjs`（仅 parse）：

```javascript
import { JSDOM } from 'jsdom';
import fs from 'node:fs';

const dom = new JSDOM('<!DOCTYPE html><html><body></body></html>', { pretendToBeVisual: true });
globalThis.window = dom.window;
globalThis.document = dom.window.document;
globalThis.DOMPurify = (await import('dompurify')).default;

const { default: mermaid } = await import('mermaid');
mermaid.initialize({ startOnLoad: false, securityLevel: 'loose' });

const src = fs.readFileSync(process.argv[2], 'utf8');
try {
  await mermaid.parse(src, { suppressErrors: false });
  console.log('PARSE OK');
} catch (e) {
  console.log('PARSE FAIL\n' + (e.message || e));
  process.exit(1);
}
```

`/tmp/mmd-verify/render.mjs`（更严格，jsdom 缺的 SVG layout API 用 stub 顶上）：

```javascript
import { JSDOM } from 'jsdom';
import fs from 'node:fs';

const dom = new JSDOM('<!DOCTYPE html><html><body><div id="g"></div></body></html>', { pretendToBeVisual: true });
globalThis.window = dom.window;
globalThis.document = dom.window.document;
globalThis.Element = dom.window.Element;
globalThis.HTMLElement = dom.window.HTMLElement;
globalThis.SVGElement = dom.window.SVGElement;
globalThis.navigator = dom.window.navigator;
globalThis.getComputedStyle = dom.window.getComputedStyle;

// jsdom 不实现 SVG 几何 API，用 stub 顶上
const stubBBox = () => ({ x: 0, y: 0, width: 100, height: 20 });
const stubCTM = () => ({ a: 1, b: 0, c: 0, d: 1, e: 0, f: 0 });
for (const proto of [dom.window.Element.prototype, dom.window.SVGElement?.prototype].filter(Boolean)) {
  if (!proto.getBBox) proto.getBBox = stubBBox;
  if (!proto.getCTM) proto.getCTM = stubCTM;
  if (!proto.getScreenCTM) proto.getScreenCTM = stubCTM;
  if (!proto.getComputedTextLength) proto.getComputedTextLength = () => 50;
}

globalThis.DOMPurify = (await import('dompurify')).default;
const { default: mermaid } = await import('mermaid');
mermaid.initialize({ startOnLoad: false, securityLevel: 'loose' });

const src = fs.readFileSync(process.argv[2], 'utf8');
try {
  const { svg } = await mermaid.render('g', src);
  console.log('RENDER OK len=', svg.length);
} catch (e) {
  console.log('RENDER FAIL\n' + (e.message || e));
  process.exit(1);
}
```

#### 跑两次

```bash
node /tmp/mmd-verify/check.mjs <修复后的 .mmd 文件>
node /tmp/mmd-verify/render.mjs <同上>
```

两个都 OK 才进入第四步。否则把错误信息原样回填给用户 + 标记问题位置。

### 第四步：输出报告

按下面格式呈现：

```
## mermaid-verify 报告

### 静态修复
- ✅ subgraph 改单参数 (N 处)
- ✅ HTML entity 替换 (N 处)
- ✅ 行尾空格清理 (N 处)
（无修复时显示 "无问题"）

### 建议优化（未应用）
- ⚠ 未设置 curve type，建议加 `stepBefore` 阶梯式直角折线
- ⚠ 日志/副作用线跟主流交叉，建议聚到底部 subgraph 用虚线
（按需列出）

### 本地验证
- PARSE: OK
- RENDER: OK (SVG NNNN 字节)

### 修复版
保存到：<路径>
md5: <md5>

```mermaid
[修复后完整源码]
```
```

如果 PARSE / RENDER 失败：

```
## mermaid-verify 报告

### 验证失败
- PARSE FAIL @ line N: <错误原文>

### 推测原因
[基于错误信息和清单匹配，给出诊断]

### 已经做的修复
[列出本轮做的修复]

### 还需用户决定的
[需要用户给出业务意图才能修复的歧义点]
```

## 规则

- **必须本地验证后再交付**，除非环境无 node 且明确告知用户
- **修复版必须保留原作者意图**：节点 ID / 边 / 标签语义不变，只改语法层错误
- **建议优化是可选的**，未经用户同意不应用（避免改变图的视觉风格）
- **报告用中文**
- 如果输入明显被剪贴板损坏（含 `�` / 行不完整），不要硬猜，**直接请求用户重发**
