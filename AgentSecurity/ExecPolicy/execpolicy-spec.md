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

ExecPolicy 引擎接收的是**已 token 化的命令列表**（`[String]`），tokenization 是调用方的责任。策略文件中的 `match`/`not_match` 示例字符串使用 `shlex` 解析为 token 列表（字符串示例与列表示例均支持）。

```
调用方传入: ["git", "reset", "--hard", "HEAD~3"]

策略中示例写法（两种等价）：
  match = [["git", "reset", "--hard"]]      ← 列表格式（直接使用）
  match = ["git reset --hard"]              ← 字符串格式（shlex 解析后使用）
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
    pattern:        [PatternToken], // 要匹配的命令前缀 token 列表
    decision:       Decision,       // 命中后的决策
    justification:  String?,        // 人类可读的理由
}
```

> **注意**：`match` 和 `not_match` 是 `prefix_rule()` Starlark 函数的参数，用于**加载时验证**（详见 §3.4）。它们不是 Rule 结构体的持久字段——验证通过后即被丢弃。Rule 的持久数据只有 `pattern`、`decision`、`justification`。

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
    rules:              [Rule],                         // 已加载的所有规则
    network_rules:      [NetworkRule],                  // 网络访问规则
    host_executables:   {String: [AbsolutePath]},       // 可执行文件路径白名单
}
```

> **关于 banned_prefixes**：禁止自动学习的前缀列表是编译期硬编码常量（非 Policy 可配置字段）。详见 §3.5 完整列表。

### 2.4 NetworkRule（网络规则）

```
NetworkRule ::= {
    host:           String,       // 精确域名（不支持通配符）
    protocol:       "https" | "http" | "socks5_tcp" | "socks5_udp",
    decision:       Decision,     // "allow" | "prompt" | "forbidden" | "deny"
    justification:  String?,
}
```

域名必须精确匹配，不支持 `*` 通配符。`*.github.com` 是无效的。

> **别名**：`"deny"` 是 `"forbidden"` 的别名（仅 network_rule）。`"https_connect"` 和 `"http-connect"` 是 `"https"` 协议别名。

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
  → 查找 host_executables_by_name 中是否有该 basename 的条目：
      - 如果有条目 → 检查 cmd[0] 是否在该条目的路径白名单中
        · 在白名单中 → 用 ["git", ...] 重新做 Step 1 匹配
        · 不在白名单中 → 拒绝回退，不匹配
      - 如果没有条目 → 允许回退（无白名单 = 无限制），用 ["git", ...] 重新做 Step 1 匹配

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

> **实现注记**：Codex 源码中，用户/项目级规则先合并为 base Policy，组织级 requirements 再通过 `Policy::merge_overlay()` 叠加。效果等价于"所有规则平等匹配取 max"。

### 3.4 策略加载时验证（match/not_match）

策略文件解析后、生效前，必须验证：

1. 对每条规则，遍历其 `match` 列表中的每个示例命令：
   - 用规则 pattern 做前缀匹配 → 如果没命中 → **加载失败，报错**
2. 对每条规则，遍历其 `not_match` 列表中的每个示例命令：
   - 用规则 pattern 做前缀匹配 → 如果命中了 → **加载失败，报错**

**验证时机**：策略文件被加载/解析时。不是运行时。

### 3.5 审批持久化（动态学习）

当用户批准了一条命令后，系统经历**两阶段判断**：

#### Phase A: 前置条件检查（Amendment Derivation）

不是每次批准都会触发持久化。系统首先判断是否需要建议 amendment：

1. **策略规则 Prompt 优先**：如果任何策略规则（非 heuristics）给出了 Prompt 决策，**不触发持久化**——因为已有显式规则，amendment 不会改变结果
2. **Sandbox 绕过路径**：如果命令在 sandbox 中失败、用户在 sandbox 外批准，且无策略规则命中 → 可触发持久化（允许未来跳过 sandbox）
3. **Heuristics Prompt 路径**：如果 Prompt 仅来自 heuristics（无策略规则命中）→ 可触发持久化

#### Phase B: 执行持久化

如果 Phase A 判断应持久化：

1. 构造一条 Allow 规则：`prefix_rule(pattern=<cmd>, decision="allow")`
2. **禁止前缀检查**：用**精确长度匹配**检查 `<cmd>` 是否在硬编码的禁止列表中
   - 如果 `<cmd>` 的 token 数和值都精确匹配某条禁止项 → **拒绝持久化**
   - 注意：`["python3", "script.py"]`（2 token）不会匹配禁止项 `["python3"]`（1 token），因为长度不同
3. 用文件锁（flock/fcntl）获取排他写权限
4. 读取现有策略文件，按行做精确文本匹配去重
5. 追加新规则到文件末尾
6. 释放文件锁
7. 更新内存中的 Policy

**禁止自动学习的前缀**（核心安全要求，编译期硬编码）：

```
["python3"], ["python3", "-"], ["python3", "-c"],
["python"],  ["python", "-"],  ["python", "-c"],
["py"], ["py", "-3"], ["pythonw"], ["pyw"], ["pypy"], ["pypy3"],
["git"],
["bash"], ["bash", "-lc"],
["sh"], ["sh", "-c"], ["sh", "-lc"],
["zsh"], ["zsh", "-lc"], ["/bin/zsh"], ["/bin/zsh", "-lc"],
["/bin/bash"], ["/bin/bash", "-lc"],
["pwsh"], ["pwsh", "-Command"], ["pwsh", "-c"],
["powershell"], ["powershell", "-Command"], ["powershell", "-c"],
["powershell.exe"], ["powershell.exe", "-Command"], ["powershell.exe", "-c"],
["env"],
["sudo"],
["node"], ["node", "-e"],
["perl"], ["perl", "-e"],
["ruby"], ["ruby", "-e"],
["php"], ["php", "-r"],
["lua"], ["lua", "-e"],
["osascript"],
```

原因：如果用户批准了一次 `python3 -c "print('hello')"`，系统不能因此把所有后续的 `python3` 调用自动放行——因为 `python3` 可以做任何事。禁止列表覆盖了所有"可以执行任意代码"的程序前缀。

---

## 4. 不变量（Invariants）

实现必须始终满足以下约束。任何违反 = bug。

1. **最严格优先**：多规则命中时，决策 = max(所有命中规则的 decision)
2. **安全只紧不松**：任何层级的规则文件都不能放宽已有规则的效果
3. **加载即验证**：策略文件解析后立即运行 match/not_match 测试，任何失败 = 拒绝加载
4. **禁止自动学习危险前缀**：编译期硬编码的禁止前缀列表中的前缀永远不会被自动持久化
5. **通配符禁止**：network_rule 的 host 字段不接受 `*`
6. **去重写入**：审批持久化时按精确行文本检查是否有相同规则已存在
7. **优雅降级**：策略引擎初始化失败或规则文件不可读 → 尽可能以空 Policy 降级运行（走 heuristics fallback），而非硬性报错。若 heuristics 也不可用 → 返回 Forbidden
8. **确定性**：同一 (cmd, policy) 多次匹配，结果必须完全相同

---

## 5. 边界条件

以下边缘情况必须正确处理：

| 边界条件 | 期望行为 |
|---------|---------|
| 空命令 `[]` | 不匹配任何规则，进入 heuristics fallback |
| 命令比 pattern 短 | 不匹配（前缀匹配要求 cmd.len >= pattern.len） |
| 空策略集（0 条规则） | 所有命令走 heuristics fallback |
| HOST_EXECUTABLE 未定义该 basename | 回退允许（无白名单 = 无限制），用 basename 重新匹配 |
| HOST_EXECUTABLE 已定义但路径不在白名单 | 拒绝 basename 回退，不匹配 |
| host_executables 白名单为空 | 等价于未定义，所有路径均可回退 |
| 两条规则 pattern 完全一致 | 两者都加载，决策取 max |
| match 列表为空 | 无验证负担（合法，但不推荐） |
| 并发追加规则到同一文件 | 文件锁保证串行写入 |
| pattern 第一个 token 是备选项 | 展开为多条独立规则（等价于写了多个 prefix_rule） |

---

## 6. 与沙箱/网络的交互

ExecPolicy 是独立模块，但它与两个外部系统交互：

- **沙箱**：当 heuristics fallback 判断"沙箱足够严格"时，未知命令降级为 Allow。这要求 ExecPolicy 能感知当前沙箱状态（传入参数）。此外，sandbox 失败后会触发特殊的 amendment 路径——允许用户在 sandbox 外执行，并将该命令持久化为 Allow 规则（跳过未来 sandbox）。
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
