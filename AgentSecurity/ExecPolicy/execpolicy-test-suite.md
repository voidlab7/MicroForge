# ExecPolicy Acceptance Test Suite

> 用途：验证你的 ExecPolicy 实现是否正确。全部通过 = 实现正确。
> 格式：每条测试 = (命令, 策略集, 期望决策, 期望命中规则数/原因)

---

## T1: 基础前缀匹配

| # | 命令 | 规则 pattern | rule decision | 期望决策 |
|---|------|-------------|---------------|---------|
| T1.1 | `["git", "reset", "--hard"]` | `["git", "reset", "--hard"]` | Forbidden | **Forbidden** |
| T1.2 | `["git", "reset", "--hard", "HEAD~3"]` | `["git", "reset", "--hard"]` | Forbidden | **Forbidden**（前缀匹配，不是完整命令匹配） |
| T1.3 | `["git", "reset", "--soft"]` | `["git", "reset", "--hard"]` | Forbidden | **不匹配**（走 fallback） |
| T1.4 | `["git", "status"]` | `["git", "reset"]` | Prompt | **不匹配**（前缀不匹配：reset ≠ status） |
| T1.5 | `["git", "reset"]` | `["git", "reset"]` | Prompt | **Prompt**（恰好等于 pattern 也匹配） |
| T1.6 | `["ls"]` | `["git"]` | Forbidden | **不匹配**（cmd[0] = "ls" ≠ "git"） |

---

## T2: 备选 Token（Alternatives）

| # | 命令 | 规则 pattern | 期望 |
|---|------|-------------|------|
| T2.1 | `["bash", "-c", "echo"]` | `[["bash","sh"], ["-c","-l"]]` → Forbidden | **Forbidden** |
| T2.2 | `["sh", "-l", "echo"]` | `[["bash","sh"], ["-c","-l"]]` → Forbidden | **Forbidden** |
| T2.3 | `["zsh", "-c", "echo"]` | `[["bash","sh"], ["-c","-l"]]` → Forbidden | **不匹配**（cmd[0]="zsh" 不在备选中） |
| T2.4 | `["bash", "-x", "echo"]` | `[["bash","sh"], ["-c","-l"]]` → Forbidden | **不匹配**（"-x" 不在备选 ["-c","-l"] 中） |

---

## T3: 多规则冲突合并（最严格优先）

规则集：
```
R1: pattern=["git"],                  decision=Prompt
R2: pattern=["git", "reset", "--hard"], decision=Forbidden
R3: pattern=["git", "push"],          decision=Allow
```

| # | 命令 | 命中规则 | 期望决策 |
|---|------|---------|---------|
| T3.1 | `["git", "reset", "--hard"]` | R1 + R2 | **Forbidden**（max(Forbidden, Prompt)） |
| T3.2 | `["git", "push"]` | R1 + R3 | **Prompt**（max(Allow, Prompt)） |
| T3.3 | `["git", "status"]` | R1 only | **Prompt** |
| T3.4 | `["npm", "install"]` | 无 | **走 fallback** |

---

## T4: match/not_match 加载时验证

加载以下规则应该**失败**（加载阶段报错）：

| # | 规则 | 失败原因 |
|---|------|---------|
| T4.1 | `pattern=["rm"], decision=Forbidden, match=[["git", "status"]]` | match 示例"git status"不匹配"rm" |
| T4.2 | `pattern=["git"], decision=Prompt, not_match=[["git", "status"]]` | not_match 示例"git status"意外匹配了"git"规则 |
| T4.3 | `pattern=["cp"], match=[["cp", "a", "b"]], not_match=[["cp", "a", "b"]]` | 同一命令同时出现在 match 和 not_match 中 |

加载以下规则应该**成功**：

| # | 规则 | 原因 |
|---|------|------|
| T4.4 | `pattern=["docker"], decision=Forbidden, match=[["docker", "rm"]], not_match=[["git", "rm"]]` | 所有 match 命中，not_match 不命中 |

---

## T5: Host Executable 路径回退

策略：`host_executable(name="git", paths=["/usr/bin/git", "/opt/homebrew/bin/git"])`
规则：`pattern=["git"], decision=Prompt`

| # | 命令 | 期望 |
|---|------|------|
| T5.1 | `["/usr/bin/git", "status"]` | **Prompt**（路径在白名单中，回退成功） |
| T5.2 | `["/opt/homebrew/bin/git", "status"]` | **Prompt**（路径在白名单中） |
| T5.3 | `["/tmp/evil/git", "status"]` | **不匹配**（路径不在白名单，拒绝回退）→ 走 fallback |
| T5.4 | `["/usr/bin/npm", "install"]` | **不匹配**（npm 没有 host_executable 定义）→ 走 fallback |

---

## T6: 审批持久化 — 禁止列表

禁止前缀：`["python3", "bash", "sudo", "git"]`

| # | 操作 | 期望 |
|---|------|------|
| T6.1 | 持久化 `["python3", "-c", "print(1)"]` | **拒绝**（"python3" 在禁止列表中） |
| T6.2 | 持久化 `["npm", "install"]` | **允许**（"npm" 不在禁止列表中） |
| T6.3 | 持久化 `["bash", "script.sh"]` | **拒绝** |
| T6.4 | 持久化 `["sudo", "apt", "update"]` | **拒绝** |
| T6.5 | 持久化 `["git", "status"]` | **拒绝** |
| T6.6 | 持久化 `["cargo", "build"]` | **允许** |

---

## T7: 审批持久化 — 去重

策略文件已包含：`prefix_rule(pattern=["npm", "install"], decision="allow")`

| # | 操作 | 期望 |
|---|------|------|
| T7.1 | 再次持久化 `["npm", "install"]` | **跳过**（已存在相同规则，不去重写） |

---

## T8: Heuristics Fallback

策略集：空（0 条规则）

Fallback 行为：
- `["ls"]`、`["cat"]`、`["echo"]` → 在已知安全列表中 → Allow
- `["rm", "-rf", "/"]` → 在已知危险列表中 → Forbidden
- `["unknown_command"]` → 不在安全/危险列表 → 默认 Prompt

| # | 命令 | 期望 |
|---|------|------|
| T8.1 | `["ls"]` | **Allow**（已知安全） |
| T8.2 | `["rm", "-rf", "/"]` | **Forbidden**（已知危险） |
| T8.3 | `["unknown_cmd"]` | **Prompt**（未知命令，默认询问用户） |

---

## T9: 网络规则

| # | 域名 | 协议 | 规则决策 | 期望 |
|---|------|------|---------|------|
| T9.1 | `"api.github.com"` | `"https"` | Allow | **Allow** |
| T9.2 | `"evil.com"` | `"https"` | Forbidden | **Forbidden** |
| T9.3 | `"npmjs.org"` | `"https"` | 无规则 | **走默认（Prompt/Forbidden 取决于沙箱模式）** |

**通配符测试**：

| # | 规则定义 | 期望 |
|---|---------|------|
| T9.4 | `network_rule(host="*.github.com")` | **加载失败**（通配符不允许） |

---

## T10: 策略加载 — 文件层级合并

文件 A（用户级）：`pattern=["git", "push"] → Prompt`
文件 B（项目级）：`pattern=["git", "push"] → Forbidden`
文件 C（组织级）：`pattern=["git"] → Prompt`

加载 A + B + C，对 `["git", "push"]` 决策：

- 命中 B 的 Forbidden + C 的 Prompt → **Forbidden**（max）
- 对 `["git", "status"]` → **Prompt**（仅命中 C）

---

## T11: 边界条件

| # | 场景 | 期望 |
|---|------|------|
| T11.1 | 空命令 `[]` | 不匹配任何规则，走 fallback |
| T11.2 | 命令比 pattern 短：`["git"]` vs `pattern=["git","reset"]` | 不匹配（cmd.len < pattern.len） |
| T11.3 | 空策略集（0 条规则） | 所有命令走 heuristics fallback，不崩溃 |
| T11.4 | 策略文件不存在 | 返回空 Policy，不崩溃 |

---

## 验证方式

将本文件 + `execpolicy-spec.md` 喂给 AI：

> "请对刚才的实现运行 execpolicy-test-suite.md 中的所有测试用例。
> 对每个未通过的测试，解释失败原因并修复代码。
> 最终输出通过率：X/47 通过。"
