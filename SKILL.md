---
name: code-refactor-simplify
description: >
  代码重构与简化技能。当用户想要重构代码、优化项目结构、消除重复代码、去除复杂状态变量、
  合并重复模块、修复语法错误（如智能引号）、消除死代码参数、将派生状态替换为直接计算时，
  务必触发此技能。适用于用户说"简化代码"、"重构"、"优化结构"、"清理代码"、"整理"、
  "改善代码质量"、"消除重复"等场景。此技能处理的是重构/简化（reuse、simplification、efficiency、
  altitude），而非正确性检查（bug 修复由 /code-review 处理）。
---

# Code Refactor & Simplify

将代码的**重构逻辑**提炼为可复用的系统化流程。此技能面向软件工程项目，处理代码质量改进、
结构优化、重复消除、状态简化等场景。

## 核心原则：对变更的 diff 做四维审查

当用户给出一个代码库或 diff 进行重构时，按 Phase 0→Phase 1→Phase 2 的顺序执行。

---

## Phase 0 — 收集变更范围

```bash
# 优先：查看上游分支对比
git diff @{upstream}...HEAD
# 若没有上游分支，则对比 main
git diff main...HEAD
# 若有未提交的变更，或上方 diff 为空，包含工作区变更
git diff HEAD
```

如果用户传入了 PR 号、分支名或文件路径，直接审查指定目标。

---

## Phase 1 — 四维审查（4 个并行的审查 Agent）

在同一轮消息中启动 **4 个独立的 Agent**，每个负责一个维度。每个 Agent 返回其发现：

```json
[
  {
    "file": "相对文件路径",
    "line": 行号,
    "summary": "一行描述",
    "detail": "发现的具体问题及改进建议"
  }
]
```

### 1. Reuse（复用性）
- 标记新的代码重新实现了代码库已有的功能
- 通过 Grep 检查 shared/utility 模块和变更位置附近文件
- 指定应调用的已有帮助函数

### 2. Simplification（简化）
- 标记不必要的复杂度：
  - 冗余状态或可通过其他变量派生的状态
  - 复制粘贴后略微修改的代码块（可参数化合并）
  - 深层嵌套（可扁平化）
  - 遗留的死代码
- 命名更简洁的等价实现方式

### 3. Efficiency（效率）
- 标记浪费的计算或 I/O：
  - 独立操作被串行执行
  - 重复的 DOM 查询
  - `setTimeout` 无清理
  - 闭包捕获大范围（内存泄漏风险）
- 命名更高效的替代方案

### 4. Altitude（深度/层级）
- 检查变更是否在正确的层级实现：
  - 特殊 case 叠加在共享基础设施上 → 应深层通用化
  - 两处以上出现相同的特殊模式 → 应提取为通用机制
  - 功能模块嵌入到另一个不相关的功能模块中 → 应独立分离

---

## Phase 2 — 应用修复

等待全部 4 个 Agent 完成，**去重**指向同一行或同一机制的问题，然后逐个修复。

**跳过规则（不修复的场景）：**
- 修复会改变预期行为
- 修复涉及变更范围外的代码
- 经判断为误报的发现

**每类修复的注意事项：**

### 派生状态替换

| ❌ 坏模式 | ✅ 好模式 |
|-----------|----------|
| 用 `let creatingNewProfile = false` 跟踪状态 | 直接从 `!$("#select").value` 推导 |
| 用 `let editingMode = true` 标记编辑中 | 检查 `selectedItem !== null` |
| 用布尔变量记录 open/close | 检查 `modal.hidden` 或 `select.value` |

**核心理念**：如果一个变量的值可以通过读取另一个已有状态的变量直接计算得出，就不需要单独存储。

### 死代码参数消除

| ❌ 坏模式 | ✅ 好模式 |
|-----------|----------|
| 函数定义 `{ keepCurrentModel = true }` 但从未传 `false` | 删除参数，始终执行同一分支 |
| 分支中 `if (!x) { setA(); } else { setA(); }` | 两个分支相同 → 去掉 if-else |

### 智能引号修复

当代码中出现 Unicode 智能引号 `"“"` `"”"` `"‘"` `"’"` 时，它们
**不是 JavaScript 语法**，会导致脚本报错。必须替换为 ASCII 引号。

检测方法：
```bash
grep -Pn '[\x{201C}\x{201D}\x{2018}\x{2019}]' *.js
```

修复方法（Python）：
```python
content = content.replace('“', '\"').replace('”', '\"')
content = content.replace('‘', "'").replace('’', "'")
```

### 复用代码合并

当两处功能几乎相同（如 PlannerSettings 和 LongAiSettings 的 close、populate、save）：
1. 检查共享模块文件（如 `planner-utils.js`、`electron-bridge.js`）看是否有可用的工具函数
2. 如果没有，在共享模块提取一个带参数的函数
3. 两处都调用这个函数

### 重复 DOM 查询消除

```javascript
// ❌ 坏模式：同一元素查询多次
const id = $("#select").value;
if (profile) {
  doSomething($("#select").value);
}
$("#delete").disabled = !$("#select").value;

// ✅ 好模式：缓存到局部变量
const select = $("#select");
const id = select.value;
if (profile) {
  doSomething(select.value);
}
$("#delete").disabled = !select.value;
```

---

## 输出格式

修复完成后，向用户提供清晰的总结：

```
## 已修复

### 1. [类别] - 具体修复名称
- **文件**：path/to/file.js
- **问题**：原问题描述
- **方式**：如何修复

### 2. [类别] - ...

## 未修复（有理由的）

### 1. XX - 解释为什么不该修
```

---
