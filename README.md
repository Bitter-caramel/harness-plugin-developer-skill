---
AIGC:
    Label: "1"
    ContentProducer: 001191440300708461136T1XGW3
    ProduceID: 91a6ec231204528cf058fec04d8653c4_f1cb17909a3311f19bec525400826444
    ReservedCode1: k5Du4+apMj+hwqu/QpkHfOIfIQ93fjRTfPat2F6dSlOKcvR2gvI2s7W6ktWD78jBVZMVtEOOMe/vG/eELkvRYAcH+zylwOZxRMQV5Th6w+r9Nmm73KVJv3dhmAhW2BcLOhGplNSZwmAmiJjDurAaGcY3ed6pxij2CXiKjQoiC2kAB5tw6Lj0mvu2ftU=
    ContentPropagator: 001191440300708461136T1XGW3
    PropagateID: 91a6ec231204528cf058fec04d8653c4_f1cb17909a3311f19bec525400826444
    ReservedCode2: k5Du4+apMj+hwqu/QpkHfOIfIQ93fjRTfPat2F6dSlOKcvR2gvI2s7W6ktWD78jBVZMVtEOOMe/vG/eELkvRYAcH+zylwOZxRMQV5Th6w+r9Nmm73KVJv3dhmAhW2BcLOhGplNSZwmAmiJjDurAaGcY3ed6pxij2CXiKjQoiC2kAB5tw6Lj0mvu2ftU=
---

# harness-plugin-developer-skill

一个用于指导 **AI（LLM Agent）开发 DeepSeek Harness（dsh）插件**的专业 skill。它定义了从"写第一个插件"到"打包发布"的完整工作流，并通过"知识库为主 → 本地仓库验证兜底 → needs verification 降级"的反幻觉机制，约束 AI 只使用经源码验证的事实。

适用对象：需要让 AI 创建或修改 Harness 插件项目、编写 `cordis.yml` / `cordis.patch.yml` 配置层、或执行 `dsh` CLI 命令（挂载、安装、验证插件）的使用者。

---

## 目录结构

```
harness-plugin-developer-skill/
├── README.md                                    # 本文档（使用说明）
├── LICENSE                                      # MIT License（Copyright (c) 2026 Bitter-caramel）
├── CONTRIBUTING.md                              # 贡献与维护流程（三副本同步、源码验证、PR 检查清单）
├── CODE_OF_CONDUCT.md                           # Contributor Covenant 2.1 行为准则
├── CHANGELOG.md                                 # 变更记录（Keep a Changelog 格式）
├── harness-plugin-developer.skill.md            # 单文件版（634 行，含 derived-from 标记）
└── harness-plugin-developer/
    ├── SKILL.md                                 # 文件夹版（633 行，权威源）
    └── agents/
        └── openai.yaml                          # Agent 元数据（display_name / short_description / default_prompt）
```

**双副本关系（重要）**：同一份 skill 正文以两种形态并存——

- **文件夹版 `harness-plugin-developer/SKILL.md` 是权威源**（authoritative source），skill 加载系统实际读取的也是它；
- **单文件版 `harness-plugin-developer.skill.md` 是派生副本**，frontmatter 中标有 `derived-from: harness-plugin-developer/SKILL.md`，用于分发场景；
- **Codex 安装副本** `C:\Users\Elysia\.codex\skills\harness-plugin-developer\` 是 Codex 实际加载 skill 的位置，内容需与权威源保持字节一致。

> ⚠️ 维护提醒：修改时**以文件夹版为权威源**，并同步更新单文件版与 Codex 安装副本。`derived-from` 只是身份声明，不提供自动同步，各副本内容需要人工保持一致。

---

## 核心能力概览

| 能力 | 位置 | 说明 |
| --- | --- | --- |
| **Quick Start（90% 路径）** | Quick Start 节 | 约 30 行即可跑通第一个本地插件：一个 TS 模块 + 一个 `cordis.yml` overlay + 一条 `dsh web --patch` 命令；明确"只有打印出加载日志后才继续加功能" |
| **四阶段开发流程** | Development Workflow（Phase 1-4） | 澄清 → MVP 原型 → 反馈循环（严格三分支）→ 部署挂载（Option A 永久 / Option B 按需） |
| **CLI 使用** | §7 Command cheat sheet | `npx @deepseek-ai/dsh web`、`--patch` 热挂载、`dsh plugin --profile <name> add/remove`、`--dump-config` / `--dump-default-config` 验证 |
| **Patch 结构与配置层规则** | §3 | 顶层 YAML 数组、insert / by-id override 两种形式、later layers win、config 整值覆盖（无深合并）、`!!js` 运行时表达式、isolate 服务隔离 |
| **Bundle 与 Profile 管理** | §4 | `dsh.bundle` manifest、profile 组合、git / tarball 安装、`allowBuilds` 供应链授权规则 |
| **代码骨架** | §5 | 函数 / 对象 / 类三种插件形态、Schemastery 配置、defineTool 工具注册、生命周期与清理、事件（emit/bail/serial/waterfall）、LLM adapter |
| **核心可插拔机制地图** | §9 | 11 个经源码验证的定制面（system-prompt、compaction、token-meter、spill、session 系列、模型路由等），逐行标注机制 → 可验证表面 → row→package → 源码位置 |
| **替换影响面与决策** | §9.5 | 联动影响矩阵（14 行，全部源码验证）+ 8 步决策检查清单（何时替换、消费者/事件/状态审计、回滚、推荐默认组合） |
| **替换内置组件** | §10-§11 | 三种姿势（disable+insert / intercept 合并 config / 类形式提供新服务）+ compaction 引擎端到端替换示例（可编译 stub、overlay、验证步骤、契约注释） |
| **组件协作与性能定制** | §12-§13 | 服务注入、事件组合、层顺序、隔离边界；性能定制只给方法与候选热路径，不写未实测的性能结论 |
| **知识库版本锚定** | Knowledge Base Version Anchor | 绑定 deepseek-harness 提交 `99f6f02f`（2026-08-18），声明绑定事实范围、上游更新比对流程、源码优先权威规则 |
| **反幻觉机制** | Ground Rules + Reality-check table | 只使用 KB 内事实；未覆盖的先在本地仓库用 `rg` 验证；找不到就标 "needs verification"；8 条常见幻觉形态对照表 |
| **验收清单** | §16 Delivery checklist | 交付前逐项核对：name/apply 导出、Config Schema、绝对路径、构建产物、`--dump-config` 层顺序、内置 row 替换确认、用户安装方式询问 |

---

## 文档章节导航

SKILL.md 主体分为三大块：**角色与规则**（前部）、**开发工作流**（Quick Start + Phase 1-4）、**知识库**（§1-§16）。

| 章节 | 作用 |
| --- | --- |
| Role & Goal | 定义 AI 的角色（senior architect）与知识来源权威性 |
| allowed-tools | 允许使用的工具范围（文件读写、终端、`rg` 代码搜索） |
| Ground Rules (Anti-Hallucination) | 反幻觉总纲：KB 为主、本地仓库验证兜底、离线文档索引、Reality-check 纠错表 |
| Knowledge Base Version Anchor | 知识库版本锚定：绑定 deepseek-harness 提交 `99f6f02f`（2026-08-18）、绑定事实范围、上游更新比对流程、源码优先权威规则 |
| Quick Start | 最快跑通第一个插件的 90% 路径 |
| Development Workflow | Phase 1 澄清（能默认就不提问）→ Phase 2 MVP → Phase 3 反馈分支 → Phase 4 部署挂载 |
| §1 Core concepts | 运行时模型：Cordis Fiber 状态机、依赖驱动加载、自动清理、HMR |
| §2 Project layouts | 本地开发 overlay 与可分发 bundle 的目录结构、三包分层模型 |
| §3 Entry row fields and patch layer rules | patch 语义核心：两种 patch 形式、字段清单、层顺序、整值覆盖、`!!js` 表达式、isolate |
| §4 Bundle manifest (package.json) | `dsh.bundle` / `dsh.profile.bundles`、git 安装与供应链安全 |
| §5 Code skeletons | 插件三种形态、Schemastery 配置、defineTool、生命周期、事件、LLM adapter 的可编译示例 |
| §6 TypeScript vs JavaScript | 两种运行时（source checkout vs installed CLI）的加载差异，解释为什么 `.ts` 能直接加载 |
| §7 Command cheat sheet | 全部常用 CLI 命令速查 |
| §8 Built-in services | 内置服务速查；已在 §9 映射的服务用指针引用，`shell` / `workspace` / `terminal` / `web` / `skills` 等其余服务以 `docs/subsystems/` 为准 |
| §9 Core pluggability map | 核心可插拔机制地图：11 个经源码验证的定制面（机制 → 可验证表面 → row→package → 源码位置） |
| §9.5 Impact analysis and design decisions | 替换影响面分析：14 行源码验证的联动矩阵 + 8 步决策检查清单（消费者/事件/状态审计、回滚、默认组合） |
| §10 Replacing built-in components: three approaches | 替换内置组件的三种姿势：disable+insert、intercept 合并 config、类形式提供新服务 |
| §11 Walkthrough: custom compaction engine | 端到端示例：子类化 `CompactionEngine` 替换 `compaction-basic`，含可编译 stub、overlay、`--dump-config` 验证步骤、契约注释 |
| §12 Core components working together | 核心组件协作：服务注入、事件组合、层顺序、隔离边界 |
| §13 Performance-oriented customization | 性能导向定制：候选热路径 + "先基准、单点改动、同负载复测、自动化"的方法；不写未实测的性能结论 |
| §14 Testing | 测试约定：vitest、HMR-safety 测试、mock 边界 |
| §15 Common pitfalls | 7 条已知坑清单（相对路径、config 覆盖、waterfall next()、工具契约、presenter 纯度、TS-only bundle、硬编码参数） |
| §16 Delivery checklist | 交付前验收清单（10 项） |

---

## 设计亮点

1. **反幻觉机制（全文最突出的设计）**
   - **Reality-check table**：主动列出 8 个最常见的幻觉形态（`plugin.yaml`、`harness` 字段、`harness plugins --patch`、`resources/` 目录约定、覆盖 row `name`、加载第二个 provider、`intercept` 语义、核心机制可交换性）并逐一给出 verified reality，让 AI 在犯错之前就知道真相；
   - **三级事实策略**：KB 内事实直接用 → KB 外先在本地仓库 `D:\Ai\deepseek-harness` 用 `rg` 验证 → 验证不到就标 "needs verification"，并明确"never write unverified findings back into this skill"。

2. **执行节奏控制**
   - Phase 1："If you can proceed with a default, the question is not blocking"——能默认就不提问，减少阻塞；
   - Phase 3：第一次负面反馈只修局部、不扩大范围；**第二次负面反馈立即停止、不得猜测意图**，产出至少 5 个具体问题等用户答复后再判断重构或修补；
   - Quick Start："Only add features after this loads"——防止过度工程。

3. **源码级细节准确性**
   - patch 语义、bail 短路规则、TS 加载行为（`node --import tsx/esm`）均与真实仓库源码逐条吻合；
   - 覆盖了只有源码级作者才写得出的细节：config 整值覆盖（无深合并）、`!!js` 运行时表达式、waterfall listener 必须调用 `next()`、presenter 纯函数约束（session-log replay）、HMR 自动清理、`allowBuilds` 供应链授权；
   - §9 核心可插拔地图的 11 行映射与配置值（如 `maxBytes: 65536`、`maxInlineBytes: 50000`）均与 `packages/bundle/base/cordis.patch.yml`（78 个 row）及对应包源码逐项核对；§10-§11 的替换姿势与 `CompactionEngine` 抽象签名同样以源码为准。

---

## 已知注意事项（客观声明）

以下为当前版本已知的局限，均为轻微缺陷或设计取舍，不影响主流程使用：

1. **多副本需人工同步**：单文件版、文件夹版与 Codex 安装副本三处并存，`derived-from` 标记只声明身份、不提供自动同步，维护时存在"只改一份"的漂移风险。
2. **无 troubleshooting 专节**：§15 pitfalls 是"已知坑清单"而非"排查流程"；§11 已提供 `--dump-config` 与 `/compact` smoke 的验证步骤，但"加载失败后如何用日志、模块解析错误定位问题"的分诊指引仍缺失。
3. **多插件协同场景示例有限**：§12 已覆盖服务注入、层顺序与隔离边界，但两个 bundle 同时 patch 同一 row、group 嵌套、依赖循环等冲突情形仍无示例。
4. **依赖本地 checkout**：知识库设计假设 `D:\Ai\deepseek-harness` 本地仓库存在（已提供"不存在则询问用户"的降级，Ground Rule 2），在无仓库环境中 KB 外事实只能标 needs verification，能力受限。
5. **性能结论以实测为准**：§13 只写方法与候选热路径，不提供"性能提升 X%"类断言——性能判断必须基于使用者自己工作负载上的实测基准（这是反幻觉约束的有意设计）。
6. **全英文内容**：指令与说明均为英文，对中文使用者不直接友好（对 LLM 可翻译执行）。

---

## 维护与贡献指引

1. **以文件夹版为权威源**：修改 skill 内容时编辑 `harness-plugin-developer/SKILL.md`。
2. **同步派生副本**：修改后同步更新 `harness-plugin-developer.skill.md`（保持 `derived-from` 标记）与 `C:\Users\Elysia\.codex\skills\harness-plugin-developer\SKILL.md`，避免副本漂移。
3. **遵守反幻觉约束**：新增事实必须来自本地仓库源码或官方文档；无法验证的内容不得写入 skill，应标注 "needs verification"。
4. **更新 Reality-check table**：发现新的常见幻觉形态时，优先补充到 Ground Rules 的对照表中。
5. **保持代码骨架可编译**：§5、§11 的代码示例是使用者直接复制的模板，任何修改都需保证类型与语义正确。
6. **元数据同步**：`agents/openai.yaml` 与 frontmatter 中重复维护的字段（如 short-description）需保持一致。
7. **更新版本锚定**：deepseek-harness 上游更新后，按 SKILL.md「Knowledge Base Version Anchor」的比对流程更新 pinned sha/日期，并同步三副本。
8. **更新 CHANGELOG.md**：每次修改在 `CHANGELOG.md` 的 Unreleased 段记录变更。
*（内容由AI生成，仅供参考）*
