<!-- sunic4/dev-skills:skills/cc-review/SKILL.md | https://github.com/sunic4/dev-skills/blob/HEAD/skills/cc-review/SKILL.md -->

---
name: "cc-review"
description: "五轴代码审查。正确性/安全/性能/可维护/测试覆盖。"
triggers: [审查, review, 代码审查, PR, 代码评审]
---

# Review - 代码审查

## 职责
**独立第三方视角的代码质量评估** — 不是自验收(feat-accept)，而是 peer review。

## 触发方式

| 方式 | 条件 |
|------|------|
| **流程触发** (必须) | cc-feat-accept 验收通过后 |
| **显式调用** | 用户要求"review"/PR准备提交前 |

## 与 feat-accept 的区别

| | feat-accept (自验收) | review (代码审查) |
|---|---|---|
| **视角** | 开发者自己 | 第三方 (code-reviewer persona) |
| **标准** | "功能是否按需求实现" | "Staff 工程师会 approve 吗？" |
| **输出** | acceptance.md | **review-report.yaml** |

---

## 审查流程

```
feat-accept → Step1:范围评估 → Step2:五轴审查 → Step3:生成报告
                                                    │
                                              approved / request_changes / rejected
```

### Step 1: 变更范围评估

```bash
node wiki/tools/read-yaml.mjs `features/{slug}/impl-checklist.yaml` --summary
```

| 指标 | 阈值 |
|------|------|
| 变更文件数 | > 10 个 → large |
| 代码行数 | > 500 行 → large |
| 模块跨度 | > 3 个模块 → 需拆分建议 |

**变更大小判定**:
- **small**: ≤100行, ≤3文件 → 标准五轴
- **medium**: 101-500行, 4-8文件 → 标准五轴
- **large**: >500行 或 >8文件 → **必须拆分**，先 review 第一部分

> ⚠️ 单次变更超过400行 → 建议拆分为多个小 PR

### Step 2: 五轴逐项审查

**权重差异化**: Security 和 Correctness 是**一票否决轴**，其他是建议轴 (短板效应)。

| 轴 | 权重 | 一票否决? |
|----|------|----------|
| **Security** | 🔴 Critical | **是** — must级finding → rejected |
| **Correctness** | 🔴 Critical | **是** — <2 → rejected |
| **Performance** | 🟡 Standard | 否 — should/fyi级 |
| **Maintainability** | 🟡 Standard | 否 — should/fyi级 |
| **Test Coverage** | 🟢 Advisory | 否 — 必须评估但不阻止合并 |

#### 各轴检查要点

**Axis 1: Correctness**
- 边界条件/空值处理/类型安全/竞态条件
- 异常捕获/错误信息/fallback
- 数据一致性 (原子更新/副作用可控)
- API契约符合 arch 定义
- **Spike 结论继承**: design.md 中标注 `[spike-validated]` 的技术可行性项，跳过可行性审查，仅检查实现是否与 spike 结论一致

**Axis 2: Security** (增量传递: 读取 impl-checklist security_check 字段跳过已通过项)

| 关注点 | 严重级别 |
|--------|---------|
| 输入验证 (XSS/SQL/命令注入) | Critical |
| 认证授权 (越权/IDOR) | Critical |
| 敏感数据 (明文密码/token/日志泄露) | High |
| 会话管理 (固定/劫持/Cookie安全) | High |
| 依赖安全 (已知CVE) | Medium |
| 错误处理 (堆栈暴露) | Medium |
| 加密使用 (弱算法/硬编码密钥) | High |
| 文件操作 (路径遍历/上传校验) | High |

**Axis 3: Performance** (AI静态分析，标注 `[需运行时验证]`)
- O(n²)/N+1查询 / 内存泄漏风险 / 不必要请求 / 不必要重渲染

**Axis 4: Maintainability**
- 命名语义清晰 / 函数>50行? 文件>300行? / 嵌套>4层? / 圈复杂度>10?
- DRY vs DAMP (可读性优先) / 注释解释 why not how

**Axis 5: Test Coverage**
- 核心路径有测试? 边界值覆盖? / 测试验证行为非实现? / 空/null/异常输入?

### Step 3: 生成报告

```
node wiki/tools/review-generate.mjs --feature {slug}
```
输出: `{project-path}/wiki/features/{slug}/review-report.yaml`

**Verdict 判定与后续行动**:

| 条件 | verdict | 后续行动 |
|------|---------|---------|
| 所有轴≥4 且无 must finding | **approved** | → 流程完成，可发布 |
| 有 should 但无 must | **request_changes** | → 回 cc-feat-impl 修复 should 项，修复后 re-review |
| 有 must finding | **request_changes** | → 回 cc-feat-impl 修复 must 项，修复后 re-review |
| correctness<2 或 security must | **rejected** | → 回 cc-feat-design 重新设计，重新走 impl→accept→review |

---

## Finding 严重性标签

| 标签 | 阻止合并? |
|------|-----------|
| **must** | ✅ 是 |
| **should** | ⚠️ 条件性 |
| **fyi** / **optional** / **nit** | ❌ 否 |

## Anti-Rationalization

| "借口" | 反驳 |
|--------|------|
| "内部工具不需要严格" | 内部工具漏洞往往是攻击入口 |
| "后面再重构" | 临时代码是最持久的代码 |
| "覆盖率够了" | 覆盖率≠质量，检查关键路径验证 |
| "能跑就行" | "能跑"和"能维护"是两件事 |

## Red Flags

- ⚠️ 只说 "LGTM" → 要求具体反馈 (每轴至少一条)
- ⚠️ TODO/HACK/FIXME 无对应 issue → 必须创建 issue
- ⚠️ 新 dependency 无版本锁定 → 必须 lock
- ⚠️ 删除测试无解释 → 必须说明原因
- ⚠️ console.log/debugger 残留 → 必须清除
