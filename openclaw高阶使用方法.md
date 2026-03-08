# OpenClaw 高阶使用方法

> 本文档测试于 2026-03-08，基于 OpenClaw v0.2.x

## 一、核心能力概览

| 能力 | 说明 | 典型场景 |
|------|------|----------|
| **Cron 定时任务** | Gateway 内置调度器，支持消息推送 | 每日报告、周期巡检 |
| **长任务模式** | 结构化状态 + 定时汇报 + 用户交互 | 复杂多步骤任务 |
| **多 Agent 协作** | 隔离会话 + 角色分工 | 大型项目拆解 |
| **Skills 扩展** | ClawHub 安装自定义能力 | 领域特定功能 |
| **消息推送** | 跨频道/线程发消息 | 多渠道通知 |

---

## 二、Cron 定时任务

### 2.1 基本概念

- **运行位置**: Gateway 进程内部（非模型）
- **存储**: `~/.openclaw/cron/jobs.json`
- **运行日志**: `~/.openclaw/cron/runs/<jobId>.jsonl`

### 2.2 两种执行模式

| 模式 | 特点 | 适用场景 |
|------|------|----------|
| **main session** | 复用主会话上下文，心跳触发 | 需要主会话上下文的任务 |
| **isolated** | 独立会话，直接执行 | 后台杂务、频繁任务 |

### 2.3 消息推送配置

```bash
# 每日早报推送到 Discord
openclaw cron add \
  --name "Morning Brief" \
  --cron "0 8 * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "生成今日简报" \
  --announce \
  --channel discord \
  --to "channel:1234567890"
```

### 2.4 关键参数

| 参数 | 说明 |
|------|------|
| `--session isolated` | 独立会话模式 |
| `--announce` | 执行结果推送到聊天 |
| `--channel` | 目标渠道（discord/telegram/whatsapp） |
| `--to` | 目标地址（channel:user:xxx） |
| `--best-effort-deliver` | 推送失败不阻塞任务 |
| `--timeout` | 单次执行超时（ms） |

### 2.5 故障排查

| 症状 | 可能原因 | 解决方法 |
|------|----------|----------|
| 任务执行但无消息 | 设备未配对 | 检查 `~/.openclaw/devices/pending.json` |
| LLM timeout | 后端响应慢 | 检查模型后端，考虑换模型 |
| 消息发错地方 | target 格式错误 | 使用 `channel:xxx` 格式 |

---

## 三、长任务模式

### 3.1 核心原则

**单出口 + 结构化状态 + 定时汇报**

```
┌─────────────────────────────────────────────┐
│                 用户指令                      │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│  Planner: 分解计划 → STATE.md (APPROVED=false)│
└─────────────────┬───────────────────────────┘
                  ▼ 用户确认
┌─────────────────────────────────────────────┐
│  STATE.md: APPROVED=true                     │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│  Executor: 执行子任务 → 更新 STATE.md         │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│  Reporter: 定时汇报进度到线程                 │
└─────────────────────────────────────────────┘
```

### 3.2 STATE.md 结构

```yaml
# 状态字段（必填）
STAGE: 1                    # 当前阶段
JUST_DONE: 任务描述          # 刚完成的
NOW_DOING: 任务描述          # 正在做的
NEXT_STEP: 任务描述          # 下一步
RISK: 风险描述 / 无          # 当前风险
NEED_USER: true/false       # 是否需要用户输入

# 门控
APPROVED: false             # 未确认
APPROVED: true              # 已确认，可执行

# 元信息
RUN_ID: my-task-20260308
CREATED: 2026-03-08 14:00
DONE: false
```

### 3.3 子任务拆分原则

1. **每个子任务 ≤ 15 分钟**
2. **每个子任务开始前汇报**
3. **每完成一个子任务更新 STATE.md**
4. **支持随时中断和恢复**

### 3.4 后台执行方式

```bash
# 方式1: cron job 定时触发
openclaw cron add \
  --name "Task Reporter" \
  --cron "*/5 * * * *" \
  --session isolated \
  --message "读取 STATE.md 汇报进度" \
  --announce \
  --channel discord \
  --to "channel:xxx"

# 方式2: coding-agent 后台执行
# (需要 coding-agent skill)
```

### 3.5 用户交互

| 用户回复 | 含义 |
|----------|------|
| `CONFIRM` | 批准计划，开始执行 |
| `CHANGE: xxx` | 修改计划 |
| `PAUSE` | 暂停执行 |
| `STOP` | 终止任务 |

---

## 四、多 Agent 协作

### 4.1 会话隔离

- **每个频道/线程是独立会话**
- **共享 memory 文件**（MEMORY.md + memory/*.md）
- **不共享对话历史**

### 4.2 子 Agent 创建

```bash
# 通过工具调用
sessions_spawn(
  runtime="acp",      # ACP harness
  thread=true,        # 创建线程
  agentId="codex"     # 指定 agent
)
```

### 4.3 角色分工建议

| 角色 | 职责 | 对外发言 |
|------|------|----------|
| Planner | 制定计划 | 仅在确认时 |
| Executor | 执行任务 | 不发言，写文件 |
| Reporter | 汇报进度 | 是（单出口） |
| Supervisor | 监督检查 | 不发言，写文件 |

---

## 五、Skills 扩展

### 5.1 安装 Skills

```bash
# 从 ClawHub 安装
openclaw clawhub install <skill-name>

# 或通过工具
clawhub install <skill-name>
```

### 5.2 自定义 Skill 结构

```
my-skill/
├── SKILL.md          # 必须存在，描述和用法
├── scripts/          # 可执行脚本
├── templates/        # 模板文件
└── reference/        # 参考资料
```

### 5.3 Skill 调用

当任务匹配 skill 描述时，自动读取 SKILL.md 并遵循其指导。

---

## 六、最佳实践清单

### 6.1 Cron 任务

- [ ] 使用 `--session isolated` 避免污染主会话
- [ ] 使用 `--best-effort-deliver` 防止推送失败阻塞
- [ ] 设置合理的 `--timeout`
- [ ] 定期检查 `openclaw cron list` 清理无用任务

### 6.2 长任务

- [ ] 分解为 ≤15 分钟的子任务
- [ ] 使用 STATE.md 记录进度
- [ ] 设置门控 `APPROVED=false`
- [ ] 每个子任务开始前汇报
- [ ] 单出口发言，避免多角色冲突

### 6.3 多 Agent

- [ ] 一个任务一个线程
- [ ] 长任务与闲聊分离
- [ ] 角色明确分工
- [ ] 仅 Reporter 对外发言

---

## 七、故障排查速查表

| 问题 | 检查项 | 解决方法 |
|------|--------|----------|
| Cron 执行但无消息 | `~/.openclaw/devices/pending.json` | 配置自动批准 |
| LLM 超时 | 模型后端状态 | 换模型或检查网络 |
| 长任务卡住 | STATE.md 的 STAGE/NOW_DOING | 手动恢复或重启 |
| 消息重复 | 多角色发言 | 收敛为单出口 |
| 子任务超时 | 单次执行时间 | 进一步拆分 |

---

## 八、参考链接

- OpenClaw 官方文档: https://docs.openclaw.ai
- Cron Jobs: https://docs.openclaw.ai/automation/cron-jobs
- ClawHub Skills: https://clawhub.com
- 本地文档: `/opt/homebrew/lib/node_modules/openclaw/docs/`

---

*本文档由 Claw 生成，作为 Git 能力测试的一部分。*
