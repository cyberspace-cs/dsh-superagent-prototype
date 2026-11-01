# 009 架构导读（中文）

> 本文是中文导读，不是上游文档的翻译。它把 `docs/000`–`docs/008` 的内容压缩成一条可以快速读完的线索，引用均指向原文。代码以 `src/` 为准，原文以英文为准。

## 一句话定位

`dsh-superagent` 是一个**面向单个 GitHub 仓库的主动式 Coding Agent**，以 [DeepSeek Harness](https://www.deepseek.com/en/harness/) 的 bundle 形式运行：监听 GitHub（可选飞书）→ 判断这次请求是否该由它接手 → 开一张 Ticket → 在独立 Session 和独立目录里干活 → 回到原对话交付（通常是一个 Pull Request）。配套 Web 面板只展示"需要人处理的事"。

它同时是一套工作方法，核心原则写在 `docs/000-philosophy.md`：

> **Take as little human attention as possible, and give as much value as possible.**

每一条消息、通知、提问、设置项、面板元素、文档段落，都要按"消耗了谁多少注意力"来审判；给不出超额价值，就删掉。

## 全局数据流

```
                 ┌──────────────┐
   GitHub ──────▶│              │
   (Issues/PR/   │   Source     │  一个 Source = 一种可复用实现 + 一份实例配置
    CI/通知)     │              │  自己记读位置，丢掉自己的消息
   飞书 ────────▶│              │
                 └──────┬───────┘
                        │ Trigger（不可变事件，带 actor）
                        ▼
                 ┌──────────────┐
                 │   Thread     │  同一话题的 Trigger 时序流
                 │              │  同时最多一个 Eval
                 └──┬────────┬──┘
       Subscription │        │ 批量
         (上下文)   │        ▼
                    │   ┌──────────────┐
                    │   │    Eval      │  gate（谁在说话）+ assessment（该不该干）
                    │   │  无状态判断   │  输出 Decision: accept / stop / ignore / unavailable
                    │   └──────┬───────┘
                    │          │ accept（附 Mandate）
                    ▼          ▼
                 ┌──────────────┐
                 │   Ticket     │  已授权的工作单元，自带状态机
                 │              │  open → ended，可被人一条消息唤醒
                 └──────┬───────┘
                        │ Assignment
                        ▼
                 ┌──────────────┐
                 │  Executor    │  worker-<ticket id> Session + 独立目录
                 │  (DSH)       │  可后台跑、可交接给人、可恢复
                 └──────┬───────┘
                        │ Report / 提问
                        ▼
                 ┌──────────────┐
                 │   Outbox     │  回执 + 出站消息（唯一写回执的地方）
                 └──────┬───────┘
                        ▼
                 Channel：GitHub 评论 / PR / review reply，飞书话题回复
```

## 代码分层

| 层 | 目录 | 职责 | 关键点 |
| --- | --- | --- | --- |
| 组合根 | `src/app/runtime.ts` | 构造整个应用、启动/关闭、锁状态目录 | 不含领域规则；失败时回滚 Store 与锁 |
| 领域内核 | `src/core/engine.ts` | Trigger / Thread / Decision / Ticket / Assignment 的**唯一写者** | 全部方法同步；每条命令一个事务 |
| 内核部件 | `src/core/part.ts` 及其实现 | `Intake` / `Evals` / `Tickets` / `Assignments` / `Replies` 五段流水 | 依赖单向流动，跨部件命令按序调用 |
| 策略 | `src/policy.ts` | 允许名单、规则匹配、admission 表、版本化 | Policy 变更不撤销 Mandate，只投递给运行中的 Assignment |
| 执行 | `src/executor/dsh.ts`、`background.ts` | 拉起 DSH Session、目录准备、后台作业与槽位 | 一 Ticket 一 Session 一目录（通常 `git worktree`） |
| 接入 | `src/sources/` | GitHub（issue/PR/CI/通知）、飞书（事件/历史/轮询） | 各自翻译成本域的 Thread 与 Channel |
| 副作用 | `src/app/effects.ts`、`outbox.ts` | 事务提交后执行外部动作、保证回执唯一 | 端口在事务内入队，适配器在提交后执行 |
| 面板 | `src/client/` | React 面板：Overview / Source / Trigger / Eval / Execution / Settings | 先答"现在需要我吗" |
| 提示词 | `prompts/` | 路由、评估、派发、执行、团队通信、交接 | 行为靠提示词，不靠新增设置项 |

## 六个值得抄的机制

**1. 授权与上下文彻底分离。** `Mandate` 回答"凭什么做、能做多远"，不可变；`Subscription` 回答"这条线索要不要让已有 Ticket 知道"，只加上下文、不加权限。所以一个 PR 合并后有人回帖，能唤醒原 Ticket，但不会自动获得新授权；机器人消息可以进入上下文，却不能触发新工作。

**2. 单写者 + 事务边界。** `Engine` 是五个聚合的唯一写者，方法全同步，一条命令一个事务。所有外部副作用（发消息、开 Session）都通过端口在事务内**入队**，提交后才由 Effects 执行。这让"状态已改但消息没发"这类半成品状态在结构上无法出现。

**3. 幂等靠不可变事件。** Trigger 一旦落库不可改，重复 accept 同一 Trigger 只会产生一张 Ticket。Source 自己维护读位置，重启不重放。

**4. 执行隔离与人的优先权。** 每张 Ticket 拿到 `worker-<id>` Session 和独立目录，命令以操作者系统用户身份免审批运行。前台 shell 超过 5 秒转后台，让 worker 能在下一步收到新消息而不是干等。并发槽位之外**额外留一个槽给人**：人插话或查状态永远不被新任务堵住。

**5. 只按"需要人"分级上报。** 面板的 Ticket 阶段由"在等什么"决定：`person/delivery|answer|review|blocked|failed` 才算需要人，其余是 `working/queued/idle`。健康状态不占版面。消息只在三种时刻发出：需要人决策、结果可用、无法完成。

**6. 失败可恢复，而不是可绕过。** 交付输入失败按 2 秒起、翻倍至 1 分钟重试；被中断的 Assignment 带恢复说明重启；做不完就如实报 `blocked`/`failed`，绝不为省一次提问而绕过限制。

## 明确被否掉的设计

原文 `docs/001-domain.md` 末尾列了一张"Rejected concepts"表，这张表比功能列表更能说明取舍：

| 被否掉 | 理由 |
| --- | --- |
| 把"人"建成实体，或跨 Source 映射身份 | 人只是 Trigger 上的元数据，身份映射放在提示词里 |
| Eval 自己调度延迟触发的 Trigger | 需要等待的条件本身就是一个 Source，避免 Eval 反向依赖 Source |
| Eval 的"等待中/已撤销"状态 | 等待属于 Source；超时就重新入队或结束为 `unavailable` |
| 一个 Thread 最多一张活跃 Ticket | 一个话题可以承载多件独立工作 |
| 在 Subscription 上做准入检查 | Subscription 覆盖整个 Thread；授权由 Mandate 单独承担 |
| Correlation 用来路由 | 路由只归 Subscription，Correlation 只补上下文 |
| Ticket 状态跟随 DSH Session | Ticket 可交接、可恢复、可被多个 Executor 执行，必须自持状态 |
| Policy 变更时取消运行中的 Ticket | 取消会留下半成品：改为记录并投递变更 |

## 规模与工程形态

- TypeScript，pnpm，`pnpm check` = 类型检查 + 测试 + 构建；MIT。
- 236 个文件，9 篇编号文档（`docs/000`–`008`），其中架构篇约 70 KB。
- 测试目录 `tests/` 覆盖引擎、授权、执行、面板、迁移、i18n，包含引擎级大用例与 panel server 夹具。
- 双语支持在 `locale/` 与 `src/client/locales.ts`，面板右上角 `?` 直接打开术语表。

## 建议阅读顺序

1. `docs/000-philosophy.md` —— 先接受"注意力是预算"这个前提，后面的设计才不显得奇怪。
2. `docs/001-domain.md` —— 只有 8 个概念，且给了完整的"拒绝清单"。
3. `docs/002-authorization.md` —— 两条路径：Eval 决定是否开工，Subscription 决定是否续接。
4. `docs/003-execution.md` —— Ticket 阶段表、槽位、后台任务、恢复。
5. `docs/007-architecture.md` —— 类与文件的对应关系，可当代码索引查。
6. `src/core/engine.ts` → `src/core/parts` 顺序读，最后看 `src/app/runtime.ts` 收口。

## 与本仓库其他项目的关系

上游把它定位为"研究原型，非官方产品"，并明确：文章里讨论的部分设计属于未来方向，本原型未实现。因此把它当作**架构参考实现**来读，比当作可直接上线的产品更合适；真正落地时，至少需要替换 Source 凭据、收紧允许名单、把主动行为先设为"仅记录不执行"。
