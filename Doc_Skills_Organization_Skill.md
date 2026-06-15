# Doc Skills 仓库治理与 Skill 整理技能 (Doc Skills Organization Skill)

本 Skill 解决 **「doc-skills 仓库本身怎么组织、怎么合并、怎么避免越拆越乱」**，与「怎么写 RDOQ Dashboard」等业务 Skill 正交。  
当新增模块文档经验、或发现某个模块被单独拆成 Skill 文件时，**先读本 Skill，再动刀**。

---

## 1. 核心原则：按工作流维度分类，不按模块分类

### 1.1 错误模式（Anti-Pattern）

| 表现 | 问题 |
|------|------|
| `RDOQ_Algorithm_Dashboard_Doc_Skill.md` | 一个编码模块 = 一个 Skill，目录随模块膨胀 |
| `Deblock_xxx_Skill.md`、`SAO_xxx_Skill.md` … | AI 无法判断读哪份；规则重复、互相引用成网 |
| README 列出 6+ 份「平级 Skill」 | 用户和 Agent 都失去「先读哪份」的优先级 |

**判定句：** 若文件名里出现 **具体 IP / 模块名**（RDOQ、Deblock、CABAC-only…），且内容主要是「某份 Dashboard 怎么写」，则 **不应** 作为独立 Skill 类型存在。

### 1.2 正确模式（Target Taxonomy）

Skill 按 **工程师在真实流程里问的问题** 划分：

| 维度 | 对应 Skill | 典型问题 |
|------|-----------|----------|
| 怎么写、怎么排、怎么画 | Interactive Doc Generation | TOC 怎么不炸？图放哪？ |
| 什么必须和 C-Model 一致 | Strict Alignment | 数字能不能估？四步 CABAC 要不要写全？ |
| 算法章怎么写硬件妥协 | Co-design | Bypass 会不会 Mismatch？ |
| 怎么推 GitHub Pages | Deployment | 白名单 gitignore 怎么配？ |
| 动态动画 / MathJax | Dynamic Simulation（附） | 扫描动画怎么写？ |

**模块知识（RDOQ、Deblock…）** 只能作为上述 Skill 内的 **§ 实战附录 / 案例**，不能升格为第六类、第七类 Skill。

---

## 2. 内容归属决策树

收到一段新经验（例如「RDOQ 第一章要拆成 1.1/1.2/1.3」）时，按下列顺序归类：

```
新经验
 ├─ 是否只适用于某一个 Dashboard 的章节结构？
 │    └─ 是 → 写入 Interactive Doc §N「实战：XXX Dashboard」
 ├─ 是否规定数字/推演必须与 C-Model 一致？
 │    └─ 是 → 写入 Strict Alignment §N
 ├─ 是否解释算法-硬件非对称 / Bypass / 吞吐妥协？
 │    └─ 是 → 写入 Co-design（或在该 Skill 内加交叉引用）
 ├─ 是否关于 Git / Pages / 仓库净化？
 │    └─ 是 → 写入 Deployment
 ├─ 是否关于 JS 动画 / 状态机 / MathJax？
 │    └─ 是 → 写入 Dynamic Simulation（附）
 └─ 以上皆否，且会重复出现在多个模块？
      └─ 升格为 Interactive Doc 或 Strict Alignment 的 **通用 §**（去掉模块名）
```

**实战案例：** RDOQ 合并（2025）

| 原内容 | 归属 |
|--------|------|
| 1.1/1.2/1.3 编排、图/文分工、检查清单 | Interactive Doc **§9** |
| Bincost 四步推演、禁止粗糙估 bit | Strict Alignment **§9** |
| 独立文件 `RDOQ_..._Skill.md` | **删除** |

---

## 3. 合并操作 SOP（Standard Procedure）

### Step 1 — 审计

```bash
# 列出所有 Skill 文件
ls -1 *_Skill.md *_skill.md 2>/dev/null

# 找模块名文件（可疑独立 Skill）
# 模式：{ModuleName}_*.md 且 ModuleName 是编码 IP
```

对每个可疑文件回答三问：

1. **去掉模块名后，规则是否仍适用于其他 Dashboard？** → 若是，写进通用 §，不要保留模块文件名。
2. **是否 80% 内容与某现有 Skill 重复？** → 合并，不并存。
3. **README 里是否因此多了一行「平级索引」？** → 应改为父 Skill 表格里的一行「§N 实战」。

### Step 2 — 迁入（不丢信息）

1. 在 **目标 Skill** 末尾新增 `## N. …（实战：模块名 Dashboard）` 或 `### N.x …`。
2. 保留：**表格、检查清单、反模式、链接到成品 HTML**；删掉与父 Skill 重复的「设计理念」套话。
3. 若 Strict Alignment 与 Interactive Doc 各需一段：各写各的 §，用「详见 XXX Dashboard 1.3 节」交叉引用，**不要复制粘贴两份长文**。

### Step 3 — 清理

```bash
# 删除已合并的独立文件
git rm ModuleName_xxx_Skill.md

# 清除断链
rg -i "ModuleName_xxx_Skill|RDOQ_Algorithm" .

# 更新 README：工作流不变，索引改为「父 Skill + §N」
```

### Step 4 — README 同步（强制）

README 必须同时表达两层结构：

1. **推荐工作流**（箭头链）：Agent 知道先做什么后做什么。
2. **Skill 索引**（4～5 份核心 + 附）：每份用 **一句话职责** + **表格列出 § 实战**，而不是为每个模块加 `### 6. RDOQ Skill`。

### Step 5 — 提交信息

```
Consolidate {Module} doc rules into existing skills instead of a standalone file.

{一句话说明：内容进了哪个 §；README 如何改}
```

---

## 4. 何时允许新建 Skill 文件（高门槛）

**仅当** 新内容满足 **全部** 条件时，才新增 `.md` 文件：

| # | 条件 |
|---|------|
| 1 | 描述的是 **与现有四份 Skill 正交的新工作流阶段**（例如全新的「RTL Review 文档化」流水线） |
| 2 | 无法合理放入现有任一份的「§ 实战」而不破坏该 Skill 的单一职责 |
| 3 | 预计 **≥3 个不同模块** 会复用其中的 **通用** 规则（不是单模块特例） |
| 4 | README 工作流图需要 **新增一个箭头阶段**（而不是多一个并列模块） |

**默认答案：** 不新建文件，追加到现有 Skill 的下一节 §N。

**本仓库的 meta 例外：** 本文件 `Doc_Skills_Organization_Skill.md` 描述的是 **仓库治理**，与业务四 Skill 正交，故单独存在。

---

## 5. 命名与结构规范

### 5.1 文件名

| 类型 | 命名 | 示例 |
|------|------|------|
| 能力 Skill | `{Domain}_{Capability}_Skill.md` 或 `snake_case_skill.md` | `Hardware_Interactive_Doc_Generation_Skill.md` |
| 实战 § | **不进文件名** | Interactive Doc §9 RDOQ Dashboard |
| 禁止 | `{IP}_{Feature}_Doc_Skill.md` | ~~`RDOQ_Algorithm_Dashboard_Doc_Skill.md`~~ |

### 5.2 节号

* 父 Skill 内实战附录用 **递增 §N**（如 §9、§10），标题带 `（实战：XXX）`。
* 同一模块不要在两个父 Skill 里都用「§9」——若两处都需要，标题写清：`§9 Bincost 四步推演（Strict Alignment）` vs `§9 算法原理章编排（Interactive Doc）`。

### 5.3 附 Skill vs 核心 Skill

| 级别 | 含义 | README 写法 |
|------|------|-------------|
| 核心 | 工作流必经阶段 | `### 1. …` 编号 |
| 附 | 某核心 § 的扩展 | `### 附： …` 不占用工作流箭头 |

---

## 6. 给 AI Agent 的检查清单

合并或新增 doc-skills 内容前：

```
- [ ] 没有新增以模块/IP 命名的独立 Skill 文件（除非用户明确要求保留）
- [ ] 新规则已归入 Interactive / Alignment / Co-design / Deployment 之一
- [ ] 模块特例在父 Skill 中以「§N 实战」出现，README 表格有对应一行
- [ ] rg 无指向已删除文件的链接
- [ ] README 工作流箭头数量 = 核心 Skill 数量（附 Skill 不进箭头）
- [ ] 提交说明写清 merge 去向，便于下次审计
```

---

## 7. 给用户的指令模板

> 按 **Doc Skills Organization Skill** 整理 doc-skills：把 `{模块}` 相关规则从独立文件合并进 `{Interactive Doc / Strict Alignment}` 的 §N；更新 README 为工作流 + 核心索引；删除冗余文件并 push。

---

## 8. 参考：RDOQ 合并前后对照

| 合并前 | 合并后 |
|--------|--------|
| 6 份 Skill（含独立 RDOQ） | 4 份核心 + 1 附 + **本治理 Skill** |
| README 平铺 6 条 | 工作流 4 步 + 表格内 §9 实战 |
| Agent 先读 RDOQ 还是 Interactive？不明确 | 永远先 Interactive Doc，RDOQ 只是 §9 案例 |

---

*Good skills scale by workflow, not by RTL block name.*
