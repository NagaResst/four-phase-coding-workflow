# 4 阶段编码工作流 (Four-Phase Coding Workflow)

> Hermes Agent skill: 以 base doc 为基础,校正实施 doc,然后改代码 — 4 阶段串行工作流,带反幻觉门。

## 这是什么

一个 [Hermes Agent](https://hermes-agent.nousresearch.com/docs) skill,用于高风险、严格审计场景下"文档校正 → 代码实现"的链式任务。适用:

- "以 X 为基础,校正 Y,然后改代码"
- "4 阶段"
- "校验 doc 然后改代码"

**4 阶段**:校验实施 doc → 编码 → 再校验 → 总结。每阶段可并行子 agent,阶段间串行。

**核心防护**:阶段 4 必须亲自反查阶段 3 报告是否属实 — **不信任子 agent 报告**。

## 安装

把整个目录复制到你的 Hermes skills 路径:

```bash
# 用户级 skill
mkdir -p ~/.hermes/skills/software-development/
cp -r four-phase-coding-workflow ~/.hermes/skills/software-development/

# 仓库级 skill (项目内,优先级更高)
mkdir -p <your-project>/.hermes/skills/software-development/
cp -r four-phase-coding-workflow <your-project>/.hermes/skills/software-development/
```

重启 Hermes Agent 或 `/new` 进新 session 后自动加载。

## 使用

直接对 Hermes 说触发词:

- "以技术文档和数据库设计为基础,校正实施计划,然后实现 M2-01"
- "4 阶段校正:M1-05~08 文档 + 代码 + 再校验 + 总结"
- "按 X 文档,校正 Y 设计,然后改代码"

skill 自动加载,父 agent 会:
1. 启动前用 `clarify` 跟你确认 base doc 列表
2. 拆 1-N 个并行子 agent 做阶段 1 校验
3. 串行进阶段 2 编码
4. 阶段 3 再校验(并行子 agent)
5. 阶段 4 父 agent 亲自反查 + 总结

## 文档结构

```
four-phase-coding-workflow/
├── SKILL.md                                          ← 主 skill,加载入口
├── LICENSE                                           ← MIT
├── README.md                                         ← 本文件
└── references/
    ├── stage-4-verification-checklist.md             ← 14 项反查清单(A-F 分类)
    ├── worked-example-time-scroll-m1.md              ← M1-05~08 实战:HTTP 429 + sibling 冲突
    ├── worked-example-time-scroll-m2-01.md           ← M2-01 实战:目标已存在陷阱
    └── worked-example-time-scroll-m2-01-02.md       ← M2-01 + M2-02 双轮复用
```

## 4 阶段快速对照

| 阶段 | 任务 | 子 agent 模式 | 父 agent 职责 |
|---|---|---|---|
| 1 | 校验实施 doc | 并行(按 base doc 拆) | 亲自 grep 验证,整理改动清单 |
| 2 | 编码 | 串行(同文件必串) | 显式列硬约束,防自由发挥 |
| 3 | 再校验 | 并行(对照 base doc) | 收报告,不写代码 |
| 4 | 总结 + 反查 | 父 agent 亲自 | grep/read_file/git log 反查每条 |

## 关键铁律

- **base doc 不写死** — 启动前 clarify 确认
- **阶段 4 必须亲自反查** — 不接受子 agent 转述
- **决策 ABC+D 评审** — 子 agent 报互斥方案时,父 agent 浓缩 + clarify + 拍板
- **目标文件/类已存在陷阱** — 阶段 1 启动前先 `ls` + `git log` 校准现状
- **patch 工具扩大修改范围** — 每个 patch 前亲自 read_file 完整读 + 脑内跑 diff

## Pitfalls 精选

完整列表见 SKILL.md 第 247-285 行。摘要:

1. 跳过 clarify 直接执行 → base doc 错位 → 全程返工
2. 阶段 1 报告不传阶段 2 → 编码 agent 不知道"实施 doc 怎么改",自由发挥
3. 阶段 3 报告直接当结论 → 失实风险高,必须阶段 4 反查
4. 阶段 4 写 commit → 跳过用户确认
5. base doc 与实施 doc 混用 → 必须区分,**base doc 是"上游真理"**

## 与其他 skill 的关系

- `parallel-task-decomposition` — 包含 "Two-Phase Coding Template"(doc→code, 2 phase),是本 skill 的**轻量版**。本 skill 适用于高风险(实施 doc 改动复杂 + 多文件代码 + 验收要严格对齐)。

## License

MIT — 详见 [LICENSE](LICENSE)。
