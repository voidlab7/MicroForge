# ExecPolicy Design Blueprint

> 用途：将本文件喂给 AI 编程助手，让它生成一个完整的命令安全策略引擎。
> 语言无关：用类型定义 + 行为契约描述，可在任意语言实现。
> 搭配 `execpolicy-test-suite.md` 使用——跑通全部测试 = 实现正确。

---

## 1. 核心概念

ExecPolicy 是一个命令执行的安全决策引擎。它的输入是一条 tokenized 命令和一组已加载的策略规则，输出一个三级决策。

### 1.1 三级决策

```
Allow     — 自动放行，无需用户交互
Prompt    — 需要用户确认后才能执行  
Forbidden — 绝对禁止，即使用户想确认也不行
```

**合并规则**：当多条规则命中同一命令时，取最严格者。

```
Forbidden > Prompt > Allow
```

### 1.2 命令 Tokenization

命令必须先经过 shell-like tokenization（如 shlex 解析），被拆成独立的 token 列表，然后才进行匹配。

```
输入: 'git reset --hard HEAD~3'
输出: ["git", "reset", "--hard", "HEAD~3"]

输入: 'git  reset  --hard'（多余空格）
输出: ["git", "reset", "--hard"]（token 化后自动规范化）
```

---

## 2. 数据模型

### 2.1 Decision（决策）

```
Decision ::= Allow | Prompt | Forbidden
```

排序关系：`Allow < Prompt < Forbidden`

当多条规则命中同一命令时，取 max（排序最大 = 最严格）。

### 2.2 Rule（规则）

```
Rule ::= {
    pattern:        [Token],        // 要匹配的命令前缀 token 列表
    decision:       Decision,       // 命中后的决策
    match:          [[Token]],      // 正例测试用例（必须全部命中）
    not_match:      [[Token]],      // 反例测试用例（必须全部不命中）
    justification:  String?,        // 人类可读的理由
}
```

**pattern 支持备选 Token：**

```
PatternToken ::= Exact(String) | Alts([String])

// 示例：pattern = [["bash", "sh"], ["-c"]]
// 匹配 bash -c ...  和  sh -c ...
```

**备选展开规则**：第一个 token 的备选项展开为独立规则（因为规则按第一个 token 索引）；后续 token 的备选项是运行时"或"匹配。

### 2.3 Policy（策略集）

```
Policy ::= {
    rules:  [Rule],     // 已加载的所有规则
    banned_prefixes: [String],  // 禁止自动学习的前缀
    host_executables: {String: [AbsolutePath]},  // 可执行文件路径白名单
}
```

### 2.4 NetworkRule（网络规则）

```
NetworkRule ::= {
    host:           String,       // 精确域名（不支持通配符）
    protocol:       "https" | "http" | "socks5_tcp" | "socks5_udp",
    decision:       Decision,
    justification:  String?,
}
```

域名必须精确匹配，不支持 `*` 通配符。`*.github.com` 是无效的。

---

## 3. 算法契约

### 3.1 前缀 Token 匹配

**输入**：一条 token 化命令 `cmd: [String]`，一条规则的 `pattern: [PatternToken]`

**输出**：是否命中（布尔值）

**算法**：

1. 如果 `cmd.length < pattern.length` → 不匹配
2. 如果 `cmd[0] != pattern[0]` → 不匹配
3. 对 i = 1 到 pattern.length - 1：
   - 如果 `pattern[i]` 是 Exact(s) 且 `cmd[i] != s` → 不匹配
   - 如果 `pattern[i]` 是 Alts(list) 且 `cmd[i] ∉ list` → 不匹配
4. 全部通过 → 匹配成功

**关键语义**：匹配的是命令的**前缀**，不是完整命令。`pattern=["git"]` 匹配所有以 `git` 开头的命令。

### 3.2 完整匹配流程

**输入**：token 化命令 `cmd`，策略 `policy`，启发式回调 `heuristics`

**输出**：`(Decision, [matched_rules])`

**流程**：

```
Step 1: 精确规则匹配
  → 遍历 policy.rules，找出所有 pattern 匹配 cmd 前缀的规则
  → 如果找到，collect 所有匹配规则，取 max(decision) 返回

Step 2: Host Executable 回退匹配
  → 如果 cmd[0] 是绝对路径（如 "/usr/bin/git"）
  → 提取 basename（"git"）
  → 如果 basename 在 policy.host_executables 白名单中
  → 用 ["git", ...] 重新做 Step 1 匹配

Step 3: Heuristics Fallback
  → 如果前两步都没命中，调用 heuristics(cmd)
  → heuristics 行为：
      - 已知安全命令 → Allow
      - 已知危险命令 → Forbidden
      - 沙箱足够严格 → Allow（沙箱兜底）
      - 以上都不成立 → Prompt

Step 4: Decision 取 max
  → 如果 Steps 1-3 生成了多个决策 → 取最严格的
```

### 3.3 多策略文件合并

**策略文件层级**（优先级从低到高）：

```
~/.codex/rules/*.rules          ← 用户全局规则（最低）
.codex/rules/*.rules            ← 项目级规则
requirements.exec_policy        ← 组织级强制规则（最高，不可覆盖）
```

**合并语义**：所有文件中的规则一起加载到 `Policy.rules` 中。匹配时全部规则平等竞争，最终决策取 max。这意味着：
- 低层级可以添加更严格的规则（收紧安全）
- 低层级不能覆盖高层级已存在的规则（安全不能放宽）

### 3.4 策略加载时验证（match/not_match）

策略文件解析后、生效前，必须验证：

1. 对每条规则，遍历其 `match` 列表中的每个示例命令：
   - 用规则 pattern 做前缀匹配 → 如果没命中 → **加载失败，报错**
2. 对每条规则，遍历其 `not_match` 列表中的每个示例命令：
   - 用规则 pattern 做前缀匹配 → 如果命中了 → **加载失败，报错**

**验证时机**：策略文件被加载/解析时。不是运行时。

### 3.5 审批持久化（动态学习）

当用户批准了一条之前命中的 Prompt 决策命令后：

1. 构造一条 Allow 规则：`prefix_rule(pattern=<cmd>, decision="allow")`
2. 检查 pattern[0] 是否在 `banned_prefixes` 列表中
   - 如果在 → **拒绝持久化**，下次此命令仍然 Prompt
3. 用文件锁（flock/fcntl）获取排他写权限
4. 读取现有策略文件，检查是否已有相同规则（去重）
5. 追加新规则到文件末尾
6. 释放文件锁
7. 更新内存中的 Policy（无需重新加载文件）

**禁止自动学习的前缀**（核心安全要求）：

```
["python3", "python3", "-c"]
["bash", "bash", "-lc"]
["sh", "-c"]
["git"]
["sudo"]
["node", "-e"]
["ruby", "-e"]
["perl", "-e"]
// 实质上：任何"可以执行任意代码"的程序前缀
```

原因：如果用户批准了一次 `python3 -c "print('hello')"`，系统不能因此把所有后续的 `python3` 调用自动放行——因为 `python3` 可以做任何事。

---

## 4. 不变量（Invariants）

实现必须始终满足以下约束。任何违反 = bug。

1. **最严格优先**：多规则命中时，决策 = max(所有命中规则的 decision)
2. **安全只紧不松**：任何层级的规则文件都不能放宽已有规则的效果
3. **加载即验证**：策略文件解析后立即运行 match/not_match 测试，任何失败 = 拒绝加载
4. **禁止自动学习危险前缀**：banned_prefixes 中的前缀永远不会被自动持久化
5. **通配符禁止**：network_rule 的 host 字段不接受 `*`
6. **去重写入**：审批持久化时检查是否有相同规则已存在
7. **Fail-Closed**：策略引擎初始化失败或文件不可读 → 默认返回 Forbidden 而非 Allow
8. **确定性**：同一 (cmd, policy) 多次匹配，结果必须完全相同

---

## 5. 边界条件

以下边缘情况必须正确处理：

| 边界条件 | 期望行为 |
|---------|---------|
| 空命令 `[]` | 不匹配任何规则，进入 heuristics fallback |
| 命令比 pattern 短 | 不匹配（前缀匹配要求 cmd.len >= pattern.len） |
| 空策略集（0 条规则） | 所有命令走 heuristics fallback |
| HOST_EXECUTABLE 路径不在白名单 | 拒绝 basename 回退，不匹配 |
| 两条规则 pattern 完全一致 | 两者都加载，决策取 max |
| match 列表为空 | 无验证负担（合法，但不推荐） |
| 并发追加规则到同一文件 | 文件锁保证串行写入 |
| pattern 第一个 token 是备选项 | 展开为多条独立规则（等价于写了多个 prefix_rule） |

---

## 6. 与沙箱/网络的交互

ExecPolicy 是独立模块，但它与两个外部系统交互：

- **沙箱**：当 heuristics fallback 判断"沙箱足够严格"时，未知命令降级为 Allow。这要求 ExecPolicy 能感知当前沙箱状态（传入参数）。
- **Network Proxy**：网络规则 (`network_rule`) 与命令规则共用同一套 Starlark 解析引擎和 Decision 模型。区别仅在于匹配字段不同（host+protocol vs pattern）。

---

## 7. 最小可行 API

以下是推荐的模块公开接口（语言无关）：

```
// 核心决策
evaluate(cmd: [String], policy: Policy, heuristics: Fallback) → (Decision, [Rule])

// 策略管理
load_policy(files: [Path]) → Result<Policy, Error>
merge_policies(policies: [Policy]) → Policy

// 审批持久化
persist_decision(cmd: [String], policy_path: Path) → Result<(), Error>

// 扩展性
register_network_rule(rule: NetworkRule) → void
validate_match_examples(rule: Rule) → Result<(), Error>
```

---

## 使用方式

将本文件 + `execpolicy-test-suite.md` 一起喂给你的 AI 编程助手：

> "请根据 execpolicy-spec.md 中的设计规格，实现一个命令安全策略引擎。
> 语言使用 [你的语言]。
> 实现完成后，用 execpolicy-test-suite.md 中的测试用例验证正确性。"
