# MicroForge

> AI Agent 基础设施组件库 — Design Blueprint + Acceptance Tests + Reference Implementations

## 目录

```
MicroForge
├── AgentSDK/         → 造 Agent 的 SDK
├── AgentSecurity/    → 安全护栏 & 策略引擎  
│   └── ExecPolicy/   → 命令安全策略引擎
└── AgentSandbox/     → 隔离执行 & 快照回滚
```

## 使用方式

每个组件包含两份文件：
- `*-spec.md` — 设计规格书（喂给 AI 让它实现）
- `*-test-suite.md` — 验收测试集（验证实现正确性）

将两文件一起喂给 AI，生成代码后跑测试验证。
