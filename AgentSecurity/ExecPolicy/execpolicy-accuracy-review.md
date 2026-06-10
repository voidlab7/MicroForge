# ExecPolicy 规范文档 vs 源码准确度审核

> 审核对象：`execpolicy-spec.md` + `execpolicy-test-suite.md`
> 对照源码：`codex/codex-rs/execpolicy/src/` + `codex/codex-rs/core/src/exec_policy.rs`
> 审核日期：2026-06-10

---

## 总体结论

两份规范文档对 ExecPolicy 核心架构的**整体理解是准确的**，但存在 **4 处关键差异** 和 **若干细节偏差**。如果用这份 spec 重新实现 ExecPolicy：
- ✅ 核心匹配算法、决策模型、加载时验证、去重写入 —— **可直接复用**
- ⚠️ 需要注意调整 Host Executable 回退逻辑和 banned_prefixes 机制
- ❌ T5 测试套件中的部分用例预期与源码真实行为不一致

---

## 一、完全准确的部分 ✅

### 1.1 三级决策模型（§1.1）

| 维度 | Spec | 源码 (`decision.rs`) | 结论 |
|------|------|---------------------|------|
| 枚举值 | Allow / Prompt / Forbidden | `Decision::Allow / Prompt / Forbidden` | **一致** |
| 排序 | `Forbidden > Prompt > Allow` | `#[derive(Ord, PartialOrd)]`，枚举顺序 = `Allow, Prompt, Forbidden` ⇒ `Forbidden > Prompt > Allow` | **一致** |
| 合并规则 | 多规则命中取 max | `Evaluation::from_matches()`: `matched_rules.iter().map(RuleMatch::decision).max()` | **一致** |
| 决策解析 | — | `"allow"`, `"prompt"`, `"forbidden"`；network_rule 额外支持 `"deny"` ≡ Forbidden | Spec 未提 deny 别名（小差异） |

### 1.2 Pattern Token 模型（§2.2）

| 维度 | Spec | 源码 (`rule.rs`) | 结论 |
|------|------|-----------------|------|
| PatternToken | `Exact(String) \| Alts([String])` | `PatternToken::Single(String) \| Alts(Vec<String>)` | **一致** |
| 备选展开规则 | "第一个 token 的备选项展开为独立规则" | `parser.rs` L389-402: `first_token.alternatives().iter().map(\|head\| Arc::new(PrefixRule{...}))` — 每个备选生成独立 Rule | **一致** |
| `[single]` 降级 | — | `parser.rs` L207: 单个备选自动降级为 `Single` | Spec 未提及（合理的实现优化） |

### 1.3 前缀匹配算法（§3.1）

| 维度 | Spec | 源码 (`rule.rs` `PrefixPattern::matches_prefix`) | 结论 |
|------|------|--------------------------------------------------|------|
| 长度检查 | `cmd.len < pattern.len` → 不匹配 | `cmd.len() < pattern_length` → None | **一致** |
| 首 token 检查 | `cmd[0] != pattern[0]` → 不匹配 | `cmd[0] != self.first.as_ref()` → None | **一致** |
| 后续 token 匹配 | Exact(s) 且 != s → 不匹配；Alts(list) 且 ∉ list → 不匹配 | `pattern_token.matches(cmd_token)` 内：Single 做 `==`，Alts 做 `any(\|alt\| alt == token)` | **一致** |
| 前缀语义 | 匹配前缀而非完整命令 | `Some(cmd[..pattern_length].to_vec())` — 只取前缀长度 | **一致** |

### 1.4 策略加载时验证 match/not_match（§3.4）

| 维度 | Spec | 源码 | 结论 |
|------|------|------|------|
| 验证时机 | 策略文件加载/解析时 | `PolicyBuilder::validate_pending_examples_from()` 在 `parse()` 后立即调用 | **一致** |
| match 验证 | 所有 match 示例必须命中 → 否则加载失败 | `validate_match_examples()`: 遍历 examples，未匹配则 `Error::ExampleDidNotMatch` | **一致** |
| not_match 验证 | 所有 not_match 示例必须不命中 → 否则加载失败 | `validate_not_match_examples()`: 任一匹配则 `Error::ExampleDidMatch` | **一致** |
| 同一命令同时出现 | — | 会分别被 match 和 not_match 检查 → match 先通过，not_match 报错 | **一致** |

### 1.5 NetworkRule 通配符禁止（§2.4, T9.4）

| 维度 | Spec | 源码 (`rule.rs` `normalize_network_rule_host`) | 结论 |
|------|------|-----------------------------------------------|------|
| 禁止 `*` | `*.github.com` 无效 | `normalized.contains('*')` → `Error::InvalidRule("wildcards are not allowed")` | **一致** |
| 精确匹配 | 域名精确匹配 | host 不做通配符展开 | **一致** |
| 协议 | https / http / socks5_tcp / socks5_udp | 同上 + `https_connect` / `http-connect` 作为 https 别名 | Spec 未提别名（小差异） |

### 1.6 审批持久化去重（§3.5, T7）

| 维度 | Spec | 源码 (`amend.rs` `append_locked_line`) | 结论 |
|------|------|---------------------------------------|------|
| 去重检查 | "检查是否已有相同规则（去重）" | `contents.lines().any(\|existing\| existing == line)` — 精确行匹配 | **一致** |
| 文件锁 | flock/fcntl 获取排他写权限 | `file.lock()` — Unix advisory lock | **一致** |
| 追加到文件末尾 | 追加新规则 | `file.write_all(format!("{line}\n").as_bytes())` — append mode | **一致** |
| 自动补换行 | — | `if !contents.ends_with('\n') { file.write_all(b"\n") }` — 自动处理末尾无换行 | Spec 未提及（细节增强） |

### 1.7 T4 系列测试：match/not_match 加载时验证

所有 T4 测试用例（T4.1-T4.4）的预期行为与源码实现完全一致。

### 1.8 T11 系列测试：边界条件

T11.1（空命令）、T11.2（cmd 比 pattern 短）、T11.3（空策略集）、T11.4（策略文件不存在）的预期均与源码行为一致。

### 1.9 T6 审批持久化禁止列表

T6.1-T6.6 中涉及的禁止前缀（`python3`, `bash`, `sh`, `git`, `sudo`, `node`, `ruby`, `perl`）都在源码 `BANNED_PREFIX_SUGGESTIONS` 中存在。**逻辑正确，但实现机制不同**（详见差异部分）。

---

## 二、关键差异 ⚠️

### 差异 1：`banned_prefixes` 不是可配置的 Policy 字段

| | Spec (§2.3) | 源码 |
|------|------------|------|
| 定义位置 | `Policy.banned_prefixes: [String]` — 策略数据模型的字段 | `core/src/exec_policy.rs` `BANNED_PREFIX_SUGGESTIONS` — 硬编码静态常量 |
| 可配置性 | 暗示可通过 Starlark 规则文件配置 | **完全不可配置**，编译期固定 |

```rust
// 源码：硬编码的 98 条禁止前缀
static BANNED_PREFIX_SUGGESTIONS: &[&[&str]] = &[
    &["python3"], &["python3", "-"], &["python3", "-c"],
    &["python"], &["python", "-"], &["python", "-c"],
    &["git"], &["bash"], &["bash", "-lc"], &["sh"], &["sh", "-c"],
    &["sh", "-lc"], &["zsh"], &["zsh", "-lc"],
    &["sudo"], &["node"], &["node", "-e"],
    &["perl"], &["perl", "-e"], &["ruby"], &["ruby", "-e"],
    &["php"], &["php", "-r"], &["lua"], &["lua", "-e"],
    &["osascript"],
    // ... 更多，包括 pwsh/powershell 等
];
```

**影响**：Spec 的 `Policy` 数据模型定义不准确。`banned_prefixes` 不存在于 `Policy` 结构体中。

**匹配方式差异**：源码中禁止检查是**精确长度匹配**：
```rust
// exec_policy.rs L876-882
BANNED_PREFIX_SUGGESTIONS.iter().any(|banned| {
    prefix_rule.len() == banned.len()
        && prefix_rule.iter().map(String::as_str).eq(banned.iter().copied())
})
```
这意味着 `["python3"]` 匹配禁止项 `["python3"]`，但 `["python3", "script.py"]`（长度2）不会匹配禁止项 `["python3"]`（长度1）。Spec 中暗示的是"以禁止前缀开头的命令都不能学"，但实际上 `["python3", "script.py"]` 可能不会被拦截（因为它不精确匹配任何 banned 项的 token 数）。

---

### 差异 2：Host Executable 回退逻辑（影响 T5 测试预期）

| | Spec (§3.2 Step 2) | 源码 (`policy.rs` `match_host_executable_rules`) | README |
|------|-------------------|------------------------------------------------|--------|
| 规则 | "如果 basename 在 host_executables 白名单中 → 重新匹配" | 如果 `host_executables_by_name` 有该 basename 的条目 → 检查路径是否在条目中 → 不在则拒绝。**如果根本没有该 basename 的条目 → 回退允许！** | "If no `host_executable()` entry exists for a basename, basename fallback is allowed." |

源码逻辑（`policy.rs` L317-324）：
```rust
let Some(rules) = self.rules_by_program.get_vec(&basename) else {
    return Vec::new();  // basename 没有规则 → 直接返回空
};
if let Some(paths) = self.host_executables_by_name.get(&basename)
    && !paths.iter().any(|path| path == &program)  // 有白名单但路径不在里面
{
    return Vec::new();  // 拒绝回退
}
// 如果 host_executables_by_name 没有该 basename → 代码走到这里 → 回退允许！
```

**这意味着两种场景的行为不同：**
- **未定义** `host_executable(name="git", ...)` → 所有绝对路径 git 调用都会回退到 basename 规则
- **定义了** `host_executable(name="git", paths=["/usr/bin/git"])` → 只有 `/usr/bin/git` 能回退，`/opt/homebrew/bin/git` 不能

**对 T5 测试的影响：**

| 测试 | Spec 预期 | 实际行为（源码） |
|------|----------|----------------|
| T5.3 `["/tmp/evil/git", "status"]` | 不匹配（路径不在白名单） | **取决于是否定义了 `host_executable(name="git", ...)`**：如果定义了但 `/tmp/evil/git` 不在其中 → 拒绝（✓ 与 Spec 一致）。如果未定义 → **回退允许**（✗ 与 Spec 不一致） |
| T5.4 `["/usr/bin/npm", "install"]` | 不匹配（没有 npm host_executable） | 如果没有 `host_executable(name="npm", ...)` → **回退允许**（如果 `npm` 有 basename 规则）。这与 Spec 预期相反！ |

---

### 差异 3：match/not_match 不存储在 Rule 中

| | Spec (§2.2) | 源码 |
|------|-----------|------|
| Rule 数据模型 | `Rule = { pattern, decision, match, not_match, justification }` | `PrefixRule = { pattern, decision, justification }` — **无 match/not_match 字段** |
| 验证后 | — | match/not_match 仅在解析时用于 `validate_pending_examples_from()`，验证通过后被丢弃 |

Spec 在 §2.2 将 `match` 和 `not_match` 定义为 `Rule` 结构体的字段，这是不准确的。它们是 `prefix_rule()` Starlark 函数的参数，用于加载时验证，但不是 Rule 的持久字段。

---

### 差异 4：Fail-Closed 不变量不完全成立

| | Spec (§4.7) | 源码 |
|------|-----------|------|
| 断言 | "策略引擎初始化失败或文件不可读 → 默认返回 Forbidden" | 策略文件不存在/不可读时，`load_exec_policy_with_warning` 可能返回空 Policy，此时所有命令走 heuristics fallback，最终决策取决于 heuristics 逻辑（可能是 Allow） |

源码中，策略文件解析失败时会记录 warning，但仍可能构建出一个部分/空的 Policy：
```rust
// exec_policy.rs L260-266
let (policy, warning) = load_exec_policy_with_warning(config_stack).await?;
if let Some(err) = warning.as_ref() {
    tracing::warn!("failed to parse rules: {err}");
}
Ok(Self::new(Arc::new(policy)))
```

**实际行为**：空 Policy 下，安全命令（如 `ls`）通过 heuristics 判定为 Allow → 不会走到 Forbidden。Fail-Closed 仅在 heuristics fallback 也无法工作的情况下成立。

---

## 三、细节偏差 📝

### 3.1 三层策略合并比 Spec 更复杂

| | Spec (§3.3) | 源码 |
|------|-----------|------|
| 层级 | `~/.codex/rules/*.rules` < `.codex/rules/*.rules` < `requirements.exec_policy` | 通过 `ConfigLayerStack` 管理，支持 `ignore_user_and_project_exec_policy_rules()` 开关 |
| 合并方式 | "所有规则一起加载到 Policy.rules 中" | 基础 Policy（用户+项目）与 requirements overlay 通过 `Policy::merge_overlay()` 合并 |

Spec 描述的"三层规则平等竞争，取 max"在语义上是正确的，但实现上更多了一层：requirements 层通过 `merge_overlay` 叠加到基础 Policy 之上。合并行为等价于"所有规则一起匹配"。

### 3.2 Tokenization 方式

| | Spec (§1.2) | 源码 |
|------|-----------|------|
| 命令 token 化 | "必须先经过 shell-like tokenization（如 shlex 解析）" | **命令 token 在传入前已完成 tokenization**（由调用方完成）。`shlex` 只用于解析 `match`/`not_match` 中的**示例字符串** |
| 示例解析 | — | `parse_string_example()` 用 `shlex::split()` 将字符串示例拆成 token 列表 |
| 示例也支持列表格式 | — | `parse_list_example()` 直接接受 `["cmd", "arg1"]` 格式 |

Spec 的表述容易让人误以为 ExecPolicy 引擎内部做了 shell tokenization，实际上命令 tokenization 是调用方的责任。

### 3.3 Heuristics Fallback 复杂度

| | Spec (§3.2 Step 3) | 源码 |
|------|-------------------|------|
| 描述 | "已知安全→Allow, 已知危险→Forbidden, 沙箱够严→Allow, 否则→Prompt" | 实际有更复杂的逻辑链：`UnmatchedCommandContext` 包含 approval_policy、permission_profile、sandbox_permissions 等多项上下文 |
| 分类 | — | 区分 `ExecPolicyCommandOrigin::Generic` 和 `PowerShell`（Windows），分别用不同的安全/危险命令分类器 |
| Sandbox 感知 | 提及 | 实际通过 `sandbox_permissions` + `file_system_sandbox_policy` 多维度判断 |

Spec 的简化描述方向正确，但实际 heuristics 比描述的更复杂。

### 3.4 NetworkRule decision "deny" 别名

源码 `parse_network_rule_decision()` 支持 `"deny"` 作为 `Forbidden` 的别名：
```rust
fn parse_network_rule_decision(raw: &str) -> Result<Decision> {
    match raw {
        "deny" => Ok(Decision::Forbidden),  // 网络规则特有
        other => Decision::parse(other),
    }
}
```
Spec 未提及此别名。

### 3.5 审批持久化——不仅是用户批准→追加规则

源码中有一套完整的 "amendment derivation" 逻辑（`core/src/exec_policy.rs` L812-926），并非简单的"用户点了批准→追加 Allow 规则"：

1. `try_derive_execpolicy_amendment_for_prompt_rules()` — 仅当所有 Prompt 都来自 Heuristics（非策略规则）时才建议 amendment
2. `try_derive_execpolicy_amendment_for_allow_rules()` — 仅当无策略规则命中时才建议 sandbox 绕过 amendment
3. `derive_requested_execpolicy_amendment_from_prefix_rule()` — 综合检查 banned_prefixes + 已有规则 + `prefix_rule_would_approve_all_commands()` 模拟验证

Spec §3.5 描述的流程是简化版，缺少这些前置判断逻辑。

### 3.6 禁止前缀列表不完整

| | 数量 | 覆盖 |
|------|------|------|
| Spec (§3.5) | 9 条（python3, bash, sh, git, sudo, node, ruby, perl 及对应参数形式） | 基础覆盖 |
| 源码 | 98 条 | 额外包含：python 多版本、pwsh/powershell、zsh、env、osascript、php、lua 等 |

---

## 四、测试套件准确度评估

| 测试组 | 预期行为准确度 | 问题 |
|--------|-------------|------|
| T1 基础前缀匹配 | ✅ 完全准确 | — |
| T2 备选 Token | ✅ 完全准确 | — |
| T3 多规则冲突合并 | ✅ 完全准确 | — |
| T4 match/not_match 加载验证 | ✅ 完全准确 | — |
| **T5 Host Executable 回退** | ⚠️ **有问题** | T5.3 和 T5.4 的预期取决于是否定义了对应 host_executable。见差异2 |
| T6 审批持久化禁止列表 | ⚠️ 逻辑正确，机制不准确 | 禁止检查使用精确长度匹配，不是"前缀匹配" |
| T7 审批持久化去重 | ✅ 完全准确 | — |
| T8 Heuristics Fallback | ✅ 方向正确 | 简化了实际逻辑但预期结果合理 |
| T9 网络规则 | ✅ 完全准确 | T9.1-T9.4 均正确 |
| T10 策略文件层级合并 | ✅ 语义正确 | — |
| T11 边界条件 | ✅ 完全准确 | — |

---

## 五、建议修改

### 对 `execpolicy-spec.md` 的建议

1. **§2.3 Policy 数据模型**：删除 `banned_prefixes: [String]` 字段。banned_prefixes 不是 Policy 的可配置字段，是硬编码的编译期常量。

2. **§2.2 Rule 数据模型**：将 `match: [[Token]]` 和 `not_match: [[Token]]` 从 Rule 结构体中移除，或标注为"仅用于加载时验证，不持久化"。

3. **§3.2 Step 2 Host Executable 回退**：修正描述为：
   > 如果 `host_executables_by_name` 中存在该 basename 的条目 → 检查绝对路径是否在白名单中，不在则拒绝回退。如果不存在该条目 → 允许回退（无白名单 = 无限制）。

4. **§4.7 Fail-Closed 不变量**：修改为：
   > 策略引擎初始化失败 → 尽可能以空 Policy 降级运行（走 heuristics fallback），而非硬性 Forbidden。

5. **§3.5 审批持久化**：标注实际源码中还有更复杂的 amendment derivation 前置逻辑（sandbox 感知、已有规则冲突检测等），简化版可用于独立实现。

### 对 `execpolicy-test-suite.md` 的建议

1. **T5 组测试**：需要明确指定是否定义了对应的 `host_executable()` 条目，否则 T5.3/T5.4 的预期不确定。

---

## 六、总结

| 维度 | 评分 |
|------|------|
| 核心匹配算法 | ⭐⭐⭐⭐⭐ 完全准确 |
| 决策模型 | ⭐⭐⭐⭐⭐ 完全准确 |
| 加载时验证 | ⭐⭐⭐⭐⭐ 完全准确 |
| 去重写入 | ⭐⭐⭐⭐⭐ 完全准确 |
| 数据模型定义 | ⭐⭐⭐⭐ 有2处偏差（banned_prefixes、match/not_match 位置） |
| Host Executable 回退 | ⭐⭐⭐ 逻辑方向正确但细节有关键差异 |
| Fail-Closed 不变量 | ⭐⭐⭐ 过于简化，实际行为是"降级运行" |
| 审批持久化流程 | ⭐⭐⭐⭐ 核心流程正确，缺少 amendment derivation 前置逻辑 |

**两份文档作为 ExecPolicy 的"设计蓝图"质量很高，核心算法的正确性经得起源码验证。** 如果用它重新实现一个兼容的 ExecPolicy 引擎，只需调整上述 4 个关键差异点即可。测试套件 80%+ 的用例可直接用于验证。
