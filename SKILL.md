---
name: four-phase-coding-workflow
description: 4 阶段校正工作流(校验→编码→再校验→总结)。用于"以 base doc 为基础,校正实施 doc,然后改代码"类任务。base doc 不写死,启动前用 clarify 跟用户确认。阶段间串行,阶段内可并行。阶段 4 总结必须亲自反查阶段 3 报告是否属实,不信任子 agent 报告。触发词:"以 X 为基础, 校正 Y, 然后改代码" / "4 阶段" / "校验 doc 然后改代码"。
version: 1.1.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [coding, four-phase, doc-coding, anti-hallucination, parallel-then-serial]
    related_skills: [parallel-task-decomposition, hermes-agent]
---

# 4 阶段校正工作流

## 适用场景

用户给出"以 X 为基础,校正 Y,然后改代码"类指令时自动加载,例如:
- "以 harmonyos-docs 和功能设计为基础,校验实施计划, 然后改代码"
- "4 阶段校正:校对文档 + 改代码 + 再校验 + 总结"
- "按 X 文档,校正 Y 设计,然后实现"

**不适用**(单阶段任务):
- 单纯改代码(用 subagent-driven-development)
- 单纯文档校正(用 doc-validation skill)
- 单纯研究(用 parallel-task-decomposition)

## 核心原则

1. **base doc 不写死**: 阶段 1 启动前用 `clarify` 跟用户确认"哪些是 base doc"
2. **阶段间严格串行**: 上一步输出是下一步输入,绝不并行启动
3. **阶段内可并行**: 真正独立的工作流(改不同文件/不同主题)可拆 N 个子 agent 并行
4. **反幻觉门**: 阶段 3 报告"代码有问题"时,阶段 4 必须**亲自 grep/read_file/git log 反查**,确认属实后才进修复 commit
5. **实施 plan 硬约束传播**: 阶段 1 报告的"实施 doc 应该改成什么"必须**显式传给阶段 2 编码子 agent**,否则失忆

## 阶段 1:校验实施 doc(并行子 agent 池)

### 1.1 启动前:跟用户确认

用 `clarify` 问 1 次(一次性收集,不要分次):

```
问题 1: 哪些是 base doc?(可多选)
  - 技术文档 (X)
  - 功能设计 (Y)
  - UI 设计
  - 安全设计
  - 数据库设计
  - API 文档
  - 代码规范
  - 其他(用户输入)

问题 2: 校验范围?
  - 某个里程碑 (M1-05~08)
  - 某个模块
  - 全量

问题 3: 一次问清,不要追问第二轮
```

### 1.2 拆子 agent

按 base doc 数量拆 1-N 个子 agent,**每个 base doc 独立**:
- 实施计划 agent:读 base doc,对照实施计划(具体到段落),输出不一致清单
- 数据库设计 agent:读 base doc + 数据库设计文档,输出不一致清单
- 技术方案 / UI / 安全 / API agent:同上

**子 agent 任务描述模板**:

```markdown
你是阶段 1 校验子 agent。任务:对照 base doc 校验 X 实施 doc。

【base doc】
{{BASE_DOC_PATH_1}}
{{BASE_DOC_PATH_2}}
...

【实施 doc(校验对象)】
{{TARGET_DOC_PATH}}

【校验维度】
1. 命名风格偏离(类/函数/常量命名 vs 规范)
2. API 形式偏离(函数 vs class static)
3. 数字/枚举值不一致
4. 验收项缺失
5. 模块职责不清(职责污染/单一职责违反)
6. 跨文档引用断裂(链接/版本号)
7. 目标文件/类是否已存在(search_files + git log 验证,区分"新建"与"扩展")

【输出格式】
对每项问题输出:
- 严重度(高/中/低)
- 实施 doc 段落(行号/章节)
- 与 base doc 偏离说明
- 修复建议(具体到哪几行改什么)

【硬约束】
- 只读不写,不修改任何文件
- 必须用 search_files 找 base doc 路径(不要硬编码绝对路径)
- 用中文回复
- 不冗余前缀,直接给结论
```

### 1.3 阶段 1 回收

- 等待所有子 agent 完成
- 我**亲自** grep 验证关键项(防止子 agent 报告失实)
- 整理成"实施 doc 改动清单"(用于阶段 2 编码子 agent 输入)

### 1.4 决策评审门（条件触发）

当阶段 1 子 agent 报告出现**互斥方案**（N 个子 agent 对同一设计点给不同建议）：
1. 父 agent 浓缩为"建议项"列表（每项 2-3 行：采/不采 + 理由 + 风险）
2. 喂给 clarify，等用户拍板
3. 拍板后才进阶段 2

**不触发条件**：子 agent 报告无互斥方案、或用户已授权自主推进模式。

## 阶段 2:编码(并行子 agent 池,可选串行)

### 2.1 输入

- 阶段 1 报告(实施 doc 应该改成什么)
- base doc 路径列表(给**指引**,不放全文)
- 实施 doc 校正后版本(commit hash)
- 任务描述中**显式写**硬约束(API 形式 / 命名风格 / 必须遵守的设计)

### 2.2 拆子 agent 规则

**必须串行**触发条件(违反即报告失实):
- 两个任务触及同一文件路径 → 串行
- 任务 A 输出是任务 B 输入 → 串行

**可并行**触发条件:
- 改不同文件
- 不同 feature/不同模块
- 互相不引用

### 2.3 子 agent 任务描述模板

```markdown
你是阶段 2 编码子 agent。任务:实现 X feature。

【输入】
- base doc 指引: {{BASE_DOC_PATHS}}(不读全文,只读相关章节)
- 实施 doc 校正后版本(必读): {{TARGET_DOC_LATEST_COMMIT}}
- 阶段 1 校验报告(已落实在最新 commit 中): {{STAGE1_REPORT_COMMIT_HASH}}

【必须遵守】
1. API 形式: {{API_FORM}}(从实施 doc 校正版读)
2. 命名风格: 读 代码规范 v2.0 §2
3. 数据模型: 读 数据库设计 v2.1(只读相关表)
4. 错误处理: Result<T> 包装 + 内部堆栈 → hilog(实施 doc M1-05 验收已定)

【绝对禁止】
- 自由发挥 API 形式(必须严格按实施 doc 校正版)
- 修改任务描述外的文件
- 跳过错误脱敏
- 跳过测试

【输出】
- commit + commit message
- 简明说明(改了什么 + 为什么)

【硬约束】
- patch 之前必须先 read_file 完整读取目标区域
- old_string 必须唯一,带 3-5 行上下文
- 跨章节标题边界拆成更小 patch
- 出现 "modified by sibling" / "last read with offset/limit" 警告立即停止
- 用中文回复,输出格式: 动作清单 + grep 验证 + git log
```

### 2.4 阶段 2 回收

- 等待所有子 agent 完成
- 我**亲自** git log + grep 验证 commit 真实存在
- 整理成"代码改动清单"

## 阶段 3:再校验(并行子 agent 池)

### 3.1 输入

- 阶段 2 所有 commit hash
- base doc(给**指引**)
- 实施 doc 校正版(给**指引**)

### 3.2 子 agent 任务

读 代码 diff + 实施 doc 校正版 + base doc,**验证代码是否真的遵守实施 doc**:
- API 形式是否一致
- 命名是否一致
- 错误处理是否脱敏
- 数据模型映射是否完整
- 单元测试是否覆盖关键路径

### 3.3 阶段 3 回收

- 报告(可能含"代码与文档不匹配"或"代码违反了 X 规范")
- **不修改任何代码**

## 阶段 4:总结(关键:反幻觉门)

### 4.1 输入

- 阶段 3 报告**原文**
- 阶段 2 commit hash
- base doc + 实施 doc

### 4.2 阶段 4 核心:**亲自反查阶段 3 报告是否属实**

不信任阶段 3 报告,做"反查":
- 它说的文件存在吗?(ls / wc -c)
- 它说的行号对吗?(sed -n 'N'p)
- 它说的 commit 真的存在吗?(git log)
- 它说的代码片段真的在那行吗?(grep)

**对每条阶段 3 报告,标记**:
- ✅ 确认属实 → 进入修复建议
- ❌ 失实/夸大 → 推翻,记入"阶段 3 误报"列表

### 4.3 输出

**真实施记录**(不是阶段 3 报告的复述):
```
阶段 3 报告: 共 N 项
  ✅ 确认属实: M 项
  ❌ 失实: K 项
  ⚠️ 部分属实(部分对部分错): P 项

修复建议(只针对 ✅ 确认属实项):
  优先级 1: ...
  优先级 2: ...
  不建议修: ...
```

**不直接修代码**——用户确认后才修。

## 常见错误门

| 错误 | 症状 | 应对 |
|---|---|---|
| 子 agent 报告失实 | 子 agent 说"文件 X 是 0 字节",但 git log 显示已 commit | **阶段 4 反查,不信子 agent 报告** |
| 同文件并行写竞争 | patch 工具报"modified by sibling" | 拆串行(同一文件必须串行) |
| 阶段 2 编码子 agent 自由发挥 | API 形式 / 命名 / 错误处理偏离实施 doc | 任务描述里**显式写硬约束**,不只说"按实施 doc" |
| base doc 写死 | skill 锁死只能用于 harmonyos | 启动前用 clarify 问,运行时填占位符 |
| 总结 agent 写修复 commit | 跳过用户确认直接改 | **永远不**,阶段 4 只给 report + 建议,等用户拍板 |
| **目标文件/类已存在(其他阶段演示残留),实施 doc 却描述为"新建"** | 子 agent 把"实施 doc 与 base doc 偏离"和"实施 doc 与现状偏离"混为一谈,报告里既写"应该用 Promise<Result<T>>" 又写"应该新建文件" | 阶段 1 启动前父 agent **亲自 search_files/git log 确认目标文件存在状态**,在阶段 1 任务描述中显式声明"目标可能已存在,先 search_files 验证" + 阶段 4 必须反查实施 doc 段落与现状的实际关系,只对"真偏离"项打 ✅ |

## 硬约束

- 4 阶段必须**全部执行**,不能跳过任一阶段
- 阶段 4 必须**亲自反查**,不接受子 agent 转述
- base doc 列表不写死在 skill,启动时跟用户确认
- 阶段 2 编码 agent 任务描述必须**显式列硬约束**(API 形式/命名/错误处理)
- 阶段 4 不写代码,只给报告 + 修复建议

## Pitfalls

- 跳过 clarify 直接执行: base doc 错位 → 全程返工
- 阶段 1 报告不传给阶段 2: 编码 agent 不知道"实施 doc 怎么改",自由发挥
- 阶段 3 报告直接当结论: 失实风险高,必须阶段 4 反查
- 阶段 4 写 commit: 跳过用户确认
- base doc 与实施 doc 混用: 必须区分,**base doc 是"上游真理"**,实施 doc 是"待校验对象"
- 阶段 2 子 agent 太多: 难管理,建议 1-3 个并行
- 阶段 1 子 agent 报告"无问题": 警惕,可能是子 agent 没找到 base doc,不是真的无问题
- 阶段 3 子 agent 写代码: 违反硬约束,阶段 3 只读不写
- **base doc 写死导致 skill 不可复用**: 本会话初次起草此 skill 时,我在"基础原则"和"阶段 2 模板"里硬编码了 `harmonyos-docs` skill 路径。用户立刻纠正:"这个不应该写死,可以跟用户确认"。**正确做法:skill 模板用占位符 `{{BASE_DOC_PATHS}}`,启动时通过 clarify 问用户**。这是 class-level 教训——任何以"特定技术栈/特定文档"为基础的 skill,都必须留可替换接口,否则锁死不可迁移。
- **用户授权自主推进模式(2026-06-20 time-scroll M2 阶段捕获)**: 用户原话 "你来负责管理进度 做决策 把 M2 推进完"。**模式识别**:
  - 触发:用户明确授权(动词"做决策" / "推进完" / "你来负责")
  - 响应:**不要再 clarify 单任务决策点**,自主决定并在 commit message / doc 中记录理由
  - 已有锁定决策(如 M2 Repository 的 Q1-Q6)直接采纳
  - 推进策略:用 todo 工具维护全局进度,每个任务完成后 mark
  - 决策门:**新决策**(之前未遇到的设计点)仍可走 clarify
  - **反例**:M2-03 走完整 clarify(用户未授权时);M2-04~11 不走 clarify(用户授权后)
- **用户偏好子 agent 编码而非父 agent 亲自写(2026-06-20 time-scroll M2 捕获)**: 用户原话 "你亲自写也并不能保证质量 记住 用子agent写代码是为了保护你的 因为子agent 的上下文更干净"。**class-level 教训**:
  - 编码工作(Repository.ets / 测试 / 任何 .ets / .ts 实现)优先委托子 agent
  - 父 agent 亲自只做:(1) 基线核查 (2) 反幻觉门 (3) 限流/超时恢复 (4) 简单 1-2 行修正
  - 父 agent 不亲自写整个 .ets 文件(上下文"污染"风险——对话历史可能 50K+ tokens,子 agent 看到的就是给它的 clean context)
  - **应用范围**:所有 coding 任务,不限 time-scroll
- **子 agent 限流/超时恢复模式(2026-06-20 time-scroll M2 捕获)**: 子 agent 在 patch / write_file / 长 context read 上耗时 10min 触发超时,或 token 触发 429 限流。**恢复模式**:
  1. **场景 1 (全部超时)**:工作区无新文件 → 放弃该子 agent,改用下一轮重试
  2. **场景 2 (限流 429 部分完成)**:工作区有未提交 .ets 文件 → 父 agent 亲自 `git add` + `git commit`(沿用子 agent 准备的 commit message)
  3. **场景 3 (已 commit)**:工作区干净 → `git log --oneline -5` 验证后进入下一轮
  4. **场景 4 (patch 工具超时但工作已落盘)**: `git status` 看是否有未提交 diff;若有,patch 成功,父 agent 亲自审查 + 修正已知错误 + 提交
  5. **常见子 agent 写错点**(写入子 agent 任务描述时主动防御): 单位秒/分钟歧义、测试路径错(`entry/src/test` vs `entry/src/ohosTest/ets/test`)、asString 签名错(应接 `ValueType | null` 不是接 `(row, column)`)、commit message 模板
- **用户偏好"用子 agent 一次性验证整个 M2 的文档"(2026-06-20)**: 用户原话 "用子agent 让他们一次性验证完整个M2的文档"。**修正**:本 skill 阶段 1 默认"4 子 agent 并行校验"应改为"1 串行子 agent 一次性校验全阶段"。**触发条件**:
  - 多任务批量推进(用户授权模式)
  - base doc 数量固定(不需要 clarify)
  - 任务数量 > 3
  - 串行 1 子 agent 比并行 4 子 agent 更省 token(避免重复 base doc read)
  - 串行更稳(避免 3-4 个并行触发的 429 限流)
- **patch 工具扩大修改范围**: 本会话 M1-07 编码阶段,patch 工具多次吞掉紧邻章节标题/段落。教训已在 user 显式纠正下写入 memory。**skill-level 应用**: 阶段 2 编码子 agent 任务模板的"硬约束"段已经包含 patch discipline,但**我必须**在每个 patch 前亲自 read_file 全量 + 脑中跑 diff + 跨标题拆小 patch,不能信任 patch 工具的"自动 fuzzy match"行为。警告 `modified by sibling` 或 `last read with offset/limit` 出现时立即停止。
- **阶段 1 启动前必须校准"项目现状"**: 任务说"实现 X"但 X 可能已部分存在(上轮 commit 留下的"演示版"或"骨架版")。**M2-01 教训**: 实施 doc 说"实现 ProjectRepository",但 commit `3bf93a2` 早已实现 138 行骨架,目标从"新增"变为"扩展方法 + 校正 doc"。**做法**: 阶段 1 启动前(向子 agent 发任务前),亲自 `ls` + `wc` + `git log --all -- <file>` 校准目标文件状态。如果"现状"与"任务描述"有偏差,在发给子 agent 的任务里显式标注"现状 = X,本任务 = Y",子 agent 才知道边界。
- **阶段 2 子 agent 的"主动简化"需在 commit message 标注**: 子 agent 可能用更简洁的等价形式替换任务原文(如字面量联合 → 类型别名)。**做法**: 任务模板里加一条硬约束:"如替换类型/接口/常量,必须在 commit message body 注明"原: X,新: Y(等价/改进/偏离)""。阶段 4 反查时看到这种标注能快速验证。
- **用户偏好: 列选项时同时给推荐 + 理由 + 风险**: 经验证 (M2-01 + M2-02 两次 clarify 都得到"你的建议呢"回应),用户**不要**纯选项列表。**做法**: clarify 列出 A/B/C/D 选项后,如果用户问"最佳建议",立即给"我的推荐配置(全采/部分采)+ 逐项理由 + 风险与代价",而不是"中立比较"。这是 user preference,适用于所有 clarify 后续环节。

## Stage 4 反查清单(可复用)

阶段 4 反查门是整个 skill 的核心防护,反查项可沉淀为清单。见 `references/stage-4-verification-checklist.md`。

### 子 agent 报告失实早期信号

以下信号出现时,在阶段 4 反查中重点关注:
1. 子 agent 报告"无问题" — 可能没找到 base doc
2. 子 agent 报告"目标文件不存在,需要新建" — 可能未 search_files 验证
3. 子 agent 报告的索引名/函数名与 base doc 不一致 — 张冠李戴
4. 多个子 agent 对同一文件的修改建议互相矛盾 — sibling 冲突前兆

## Reuse pattern (M2-01 + M2-02 双轮)

第二轮 M2-02 完整复用 M2-01 模式(同 base doc 列表,同子 agent 数量,同 3-commit 结构),只换 base doc 章节 + 实施 doc 段。**关键复用点**:
- base doc 列表不变 → 跳过 clarify(节省一次轮次)
- 实施 doc 段结构模板沿用 M2-01 校正后版本(文件 / 职责边界 / 抽象方法 / 字段映射 / 查询方法 / 状态变更 / 索引 / 时间戳 / 错误约定 / 验收)
- 测试重写模式沿用(TestableRepository 包装 protected + StubExplodingRepository 覆写 getStore 抛错)

**风险**: 复用不等于省略澄清。如果 base doc 变了、任务范围扩大、需要新设计决策,仍要 clarify。复用仅适用于"同一项目同模块同模板"的连续任务。

完整第二轮实战见 `references/worked-example-time-scroll-m2-01-02.md`。

## References

- `references/worked-example-time-scroll-m1.md` — worked example of running this workflow against time-scroll's M1-05~08 docs + code, including HTTP 429 recovery (3/4 sub-agents 被打断, 父 agent 自己补 1 个)、sibling coordination bug (`ba730af` 修复 BaseRepository import 路径)、base-doc-not-hardcoded lesson
- `references/worked-example-time-scroll-m2-01.md` — 2026-06-16 ProjectRepository 扩展单轮: 目标已存在陷阱(父 agent 未 search_files 验证 → 子 agent D 报告"已存在 138 行" → 4 份子 agent 报告混杂两种逻辑);阶段 4 反查门救命;阶段 1→决策评审→阶段 2 模式确立
- `references/worked-example-time-scroll-m2-01-02.md` — 2026-06-16 ProjectRepository + TagRepository 双轮复用: 同 base doc 跳过 clarify;实施 doc 段结构模板沿用;测试重写模式(TestableRepository 包装 + StubExplodingRepository 覆写 getStore)沿用;Reuse decision tree 沉淀
- `references/stage-4-verification-checklist.md` — 阶段 4 反查门 14 项可复用清单(commit 真实存在 / 代码覆盖率 / 测试覆盖率 / 类型设计 / 实施 doc 与代码一致性 / 测试子类设计),按类别 A-F 组织,可按需裁剪
- 本 skill 与 `parallel-task-decomposition` 的关系: 那里的 "Two-Phase Correction Template" (doc→code, 2 phase) 是本 skill 的轻量版,适用于快速 doc+code 同步; 本 skill 适用于高风险、多文件、严格审计场景,加了阶段 3 再校验 + 阶段 4 反查门。两者并存,不合并(参见 `parallel-task-decomposition` 的 "Cross-Skill Reference vs Merge" 决策规则)。
