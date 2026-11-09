# HDF-api-finder 最终自由函数接口实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修改 `HDF-api-finder/SKILL.md`，删除"vendor free function 只是中间 helper"规则，把定义在 vendor 域的自由函数与 vendor 类成员函数同等视为最终接口，同时保持最终 JSON schema 不变。

**Architecture:** 只改 6.5 步骤 6 的最终/中间层分类与相关措辞；6.4 直通路径、阶段三查表、目标级扇出、visited、去重、域边界、输出路径全部不变。最终自由函数接口在 `02_impl.json` / `03_cases.json` 中以 `impl_class = null` 表示，最终 JSON 仅通过 case 的 `impl_interface_function` / `impl_interface_file` 体现，`impl_classes` 允许为空。

**Tech Stack:** Markdown、JSON、bash（grep/python3）、Read、Edit。

---

## 0. 输入与约束

**当前分支与提交：**

- 工作分支：`dev`（从 `main` 的 `34ae983` 拉出）。
- 增量设计已提交：`ae6dfd5`，路径：

```text
docs/superpowers/specs/2026-11-09-hdf-api-finder-vendor-free-function-interface-design.md
```

**增量设计（权威来源）：**

```text
docs/superpowers/specs/2026-11-09-hdf-api-finder-vendor-free-function-interface-design.md
```

**前置设计（历史基线，冲突处以增量设计为准）：**

```text
docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md
docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md
docs/superpowers/specs/2026-11-04-hdf-api-finder-dynamic-target-fanout-design.md
```

**交付/修改文件：**

| 文件 | 责任 |
|------|------|
| `HDF-api-finder/SKILL.md` | 唯一需要修改的交付物 |
| `docs/superpowers/specs/2026-11-09-hdf-api-finder-vendor-free-function-interface-design.md` | 增量设计；已在 `ae6dfd5` 提交；Task 14 更新状态 |
| `docs/superpowers/plans/2026-11-09-hdf-api-finder-vendor-free-function-interface.md` | 本计划；Task 1 提交 |

**全局约束：**

- 修改前先完整读取增量设计 §1–§12；所有新规则以该文档为准。
- 只修改 `HDF-api-finder/SKILL.md`；不改历史 spec/plan，不改其他文件（Task 14 例外：更新本增量设计状态与本计划复选框）。
- 示例统一使用中性 demo 命名；不使用需求原始示例中的 Nfc 命名。
- SKILL.md 正文不得出现任何代码图/MCP 代码探索工具名称，不得体现与其他 skill 的关联性。
- 保留阶段一、conf、scope、搜索纪律、输出路径、原子写出、无自动恢复、无 sub-agent。
- 最终 JSON 字段集合不变；不新增 `impl_kind`、`impl_functions` 等字段。
- 提交使用 `git -c user.name='siakam43' -c user.email='429872468@qq.com' commit`，不修改 git config。
- 当前只在 `dev` 分支提交；不合入 `main`；不执行 `git push`。
- 每个 Task 完成后运行验证命令；发现 `missing` 或缺 `present` 时先修复再进入下一 Task。

---

## Task 1: 提交实现计划

**Files:**
- Add: `docs/superpowers/plans/2026-11-09-hdf-api-finder-vendor-free-function-interface.md`

- [x] **Step 1: 确认工作区状态**

Run:

```bash
git status --short
git branch --show-current
```

Expected: 本计划为未跟踪文件；当前分支为 `dev`；增量设计无未提交改动。

- [x] **Step 2: 提交计划**

Run:

```bash
git add docs/superpowers/plans/2026-11-09-hdf-api-finder-vendor-free-function-interface.md
git -c user.name='siakam43' -c user.email='429872468@qq.com' commit -m "$(cat <<'EOF'
docs(HDF-api-finder): add vendor free-function interface implementation plan

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

Expected: 提交成功，包含 1 个文件。

- [x] **Step 3: 验证提交**

Run:

```bash
git log -1 --format='%h | %an <%ae> | %s'
git status --short
```

Expected: 作者为 `siakam43 <429872468@qq.com>`；工作区无未提交改动。

---

## Task 2: 更新第一章核心原则

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第一章，原则 3 与原则 10）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -nE '^3\. \*\*域边界。\*\*.*最终 impl 类与最终接口函数必须在 vendor 域' HDF-api-finder/SKILL.md
grep -n '最终接口必须是 vendor 类成员函数，且类定义与函数定义都在 vendor' HDF-api-finder/SKILL.md
```

Expected: 原则 3 一处、原则 10 一处。

- [x] **Step 2: 替换原则 3**

将：

```text
3. **域边界。** 入口只在 stub 域搜索；初始 impl 类可在 vendor 或 system；最终 impl 类与最终接口函数必须在 vendor 域；system 域函数只作辅助追踪/中间层，不能作为最终接口输出。
```

替换为：

```text
3. **域边界。** 入口只在 stub 域搜索；初始 impl 类可在 vendor 或 system；最终接口函数必须定义在 vendor 域，若为类成员函数则其类定义也必须在 vendor 域；system 域函数只作辅助追踪/中间层，不能作为最终接口输出。
```

- [x] **Step 3: 替换原则 10**

将：

```text
10. **调用点单线路与目标级扇出。** 调用点选择用多信号打分只选一个，并列按源码顺序取第一个并记录 `tie_break`；被选中调用点的目标动态不唯一时（函数指针、回调、`dlsym` 符号、多个 virtual override），把 vendor + system 域内全部静态可枚举目标作为下一跳分支继续追踪，不得判并列歧义、不得只选一个；最终接口必须是 vendor 类成员函数，且类定义与函数定义都在 vendor；vendor free function、system 函数、非 vendor 类成员只是中间层。
```

替换为：

```text
10. **调用点单线路与目标级扇出。** 调用点选择用多信号打分只选一个，并列按源码顺序取第一个并记录 `tie_break`；被选中调用点的目标动态不唯一时（函数指针、回调、`dlsym` 符号、多个 virtual override），把 vendor + system 域内全部静态可枚举目标作为下一跳分支继续追踪，不得判并列歧义、不得只选一个；最终接口必须是 vendor 类成员函数（类定义与函数定义都在 vendor 域）或 vendor 自由函数（函数定义在 vendor 域）；system 函数、非 vendor 类成员只是中间层。
```

- [x] **Step 4: 验证替换结果**

Run:

```bash
grep -q '若为类成员函数则其类定义也必须在 vendor 域' HDF-api-finder/SKILL.md && echo "principle3=present"
grep -q '或 vendor 自由函数（函数定义在 vendor 域）' HDF-api-finder/SKILL.md && echo "principle10=present"
grep -nE '^3\. \*\*域边界。\*\*.*最终 impl 类与最终接口函数必须在 vendor 域' HDF-api-finder/SKILL.md || echo "old_principle3=absent"
grep -n '最终接口必须是 vendor 类成员函数，且类定义与函数定义都在 vendor' HDF-api-finder/SKILL.md || echo "old_principle10=absent"
```

Expected: `principle3=present`、`principle10=present`、`old_principle3=absent`、`old_principle10=absent`。

---

## Task 3: 更新域归属判定与第十章约束规则

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第三章「域归属判定」、第十章约束规则 4/5）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '最终 impl 类与最终接口函数必须定义在 vendor 域' HDF-api-finder/SKILL.md
grep -n '最终 impl 必须在 vendor' HDF-api-finder/SKILL.md
```

Expected: 域归属判定有 1 处；约束规则 4 有 1 处。

- [x] **Step 2: 替换域归属判定**

将：

```text
- stub 接口函数必须定义在 stub 域；初始 impl 类可在 vendor 或 system；最终 impl 类与最终接口函数必须定义在 vendor 域；system 域函数只作为辅助追踪/中间层，不能作为最终接口输出。
```

替换为：

```text
- stub 接口函数必须定义在 stub 域；初始 impl 类可在 vendor 或 system；最终接口函数必须定义在 vendor 域，若为类成员函数则其类定义也必须在 vendor 域；system 域函数只作为辅助追踪/中间层，不能作为最终接口输出。
```

- [x] **Step 3: 替换约束规则 4 与 5**

将：

```text
4. 入口只在 stub 域搜索；未指定 entry 时分析全部合法入口；指定 entry 时只分析该入口；初始 impl 可在 vendor/system，最终 impl 必须在 vendor。
5. stub 接口函数必须在 stub 域；初始 impl 可在 vendor/system；最终 impl 类与最终接口函数必须在 vendor 域；system 域函数只作辅助追踪/中间层。
```

替换为：

```text
4. 入口只在 stub 域搜索；未指定 entry 时分析全部合法入口；指定 entry 时只分析该入口；初始 impl 可在 vendor/system，最终接口必须在 vendor（类成员函数还要求类定义在 vendor）。
5. stub 接口函数必须在 stub 域；初始 impl 可在 vendor/system；最终接口函数必须定义在 vendor 域（类成员函数还要求类定义在 vendor）；system 域函数只作辅助追踪/中间层。
```

- [x] **Step 4: 验证替换结果**

Run:

```bash
grep -q '最终接口函数必须定义在 vendor 域，若为类成员函数则其类定义也必须在 vendor 域' HDF-api-finder/SKILL.md && echo "domain_rule=present"
grep -q '初始 impl 可在 vendor/system，最终接口必须在 vendor（类成员函数还要求类定义在 vendor）' HDF-api-finder/SKILL.md && echo "constraint4=present"
grep -q '最终接口函数必须定义在 vendor 域（类成员函数还要求类定义在 vendor）' HDF-api-finder/SKILL.md && echo "constraint5=present"
grep -n '最终 impl 类与最终接口函数必须定义在 vendor 域' HDF-api-finder/SKILL.md || echo "old_domain=absent"
grep -n '最终 impl 必须在 vendor' HDF-api-finder/SKILL.md || echo "old_constraint4=absent"
```

Expected: 3 个 `present`、2 个 `absent`。

---

## Task 4: 重写 6.5 的最终/中间层分类与补充规则

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（6.5 步骤 6 与「补充」）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '最终接口：vendor 类成员函数，且类定义与函数定义都在 vendor' HDF-api-finder/SKILL.md
grep -n '中间层：system 函数、vendor free function' HDF-api-finder/SKILL.md
grep -n 'vendor free function 只是中间 helper，不算最终接口' HDF-api-finder/SKILL.md
```

Expected: 三处都有命中。

- [x] **Step 2: 替换步骤 6 的最终/中间层两条**

将：

```text
   - 最终接口：vendor 类成员函数，且类定义与函数定义都在 vendor → 记录到 `final_impl_functions`（含该分支 trace 与 `target_evidence`），分支结束。
   - 中间层：system 函数、vendor free function、所属类不在 vendor、vendor 类但函数/类定义不在 vendor → trace 追加该目标与调用证据，继续步骤 1。
```

替换为：

```text
   - 最终接口（任一满足）：vendor 类成员函数且类定义与函数定义都在 vendor；vendor 自由函数且函数定义在 vendor → 记录到 `final_impl_functions`（含该分支 trace 与 `target_evidence`），分支结束。
   - 中间层：system 函数、所属类不在 vendor、vendor 类但函数/类定义不在 vendor → trace 追加该目标与调用证据，继续步骤 1。
```

- [x] **Step 3: 替换「补充」前两条**

将：

```text
- vendor free function 只是中间 helper，不算最终接口。
- 最终 impl 类定义与最终接口函数定义都必须在 vendor。
```

替换为：

```text
- vendor 自由函数与 vendor 类成员函数同等作为最终接口，到达即终止该分支。
- 最终接口函数定义必须在 vendor；若为类成员函数，其类定义也必须在 vendor。
```

- [x] **Step 4: 验证替换结果**

Run:

```bash
grep -q '最终接口（任一满足）：vendor 类成员函数且类定义与函数定义都在 vendor；vendor 自由函数且函数定义在 vendor' HDF-api-finder/SKILL.md && echo "6.5_final=present"
grep -q '中间层：system 函数、所属类不在 vendor、vendor 类但函数/类定义不在 vendor' HDF-api-finder/SKILL.md && echo "6.5_middle=present"
grep -q 'vendor 自由函数与 vendor 类成员函数同等作为最终接口，到达即终止该分支' HDF-api-finder/SKILL.md && echo "6.5_supp=present"
grep -n 'vendor free function 只是中间 helper，不算最终接口' HDF-api-finder/SKILL.md || echo "old_free_rule=absent"
awk '/^### 6.5 /,/^### 6.6 /' HDF-api-finder/SKILL.md | grep -n 'vendor free function' || echo "6.5_normative_freefn=absent"
```

Expected: 3 个 `present`、2 个 `absent`。

---

## Task 5: 更新 6.6 的字段规则

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（6.6「字段与规则」）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '入口级 `impl_classes` 语义不变' HDF-api-finder/SKILL.md
grep -n '每个元素对应一条最终接口' HDF-api-finder/SKILL.md
```

Expected: 两处都有命中。

- [x] **Step 2: 在 `final_impl_functions` 元素说明后补 null 语义**

将：

```text
- `final_impl_functions` 每个元素对应一条最终接口（追踪分支或 6.4 直通映射）；元素自带该分支的 `trace`（含初始 impl 方法，不含最终函数）与 `target_evidence`；初始 impl 方法本身是最终接口时 `trace = []`。
```

替换为：

```text
- `final_impl_functions` 每个元素对应一条最终接口（追踪分支或 6.4 直通映射）；元素自带该分支的 `trace`（含初始 impl 方法，不含最终函数）与 `target_evidence`；初始 impl 方法本身是最终接口时 `trace = []`。
- 最终自由函数接口的 `impl_class` 与 `impl_class_file` 为 `null`；其余字段与类成员最终接口一致。
```

- [x] **Step 3: 替换 `impl_classes` 语义说明**

将：

```text
- 入口级 `impl_classes` 语义不变：全部最终 vendor 类按 `(impl_class, impl_file)` 去重排序。
```

替换为：

```text
- 入口级 `impl_classes` 只收录类成员最终接口所属的 vendor 类，按 `(impl_class, impl_file)` 去重排序；允许为空（全部最终接口均为自由函数时）；最终自由函数接口不产生类条目。
```

- [x] **Step 4: 验证替换结果**

Run:

```bash
grep -q '最终自由函数接口的 `impl_class` 与 `impl_class_file` 为 `null`' HDF-api-finder/SKILL.md && echo "6.6_null=present"
grep -q '允许为空（全部最终接口均为自由函数时）' HDF-api-finder/SKILL.md && echo "6.6_empty=present"
grep -n '入口级 `impl_classes` 语义不变' HDF-api-finder/SKILL.md || echo "old_implclasses=absent"
```

Expected: 2 个 `present`、1 个 `absent`。

---

## Task 6: 更新 7.5 与 7.6

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（7.5 字段说明、7.6 unresolved 表）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '追踪耗尽仍没有 vendor 类成员函数作为最终接口' HDF-api-finder/SKILL.md
grep -n '^### 7.6 unresolved 类型' HDF-api-finder/SKILL.md
```

Expected: 两处都有命中。

- [x] **Step 2: 在 7.5 末尾、7.6 之前补充字段说明**

将：

```text
### 7.6 unresolved 类型
```

替换为：

```text
- 最终自由函数接口的 `impl_class` 与 `impl_class_file` 为 `null`；`evidence_chain` 末尾使用其命名空间限定名。

### 7.6 unresolved 类型
```

- [x] **Step 3: 替换 7.6 的 `final_impl_not_found` 行**

将：

```text
| `final_impl_not_found` | 映射 | 追踪耗尽仍没有 vendor 类成员函数作为最终接口 |
```

替换为：

```text
| `final_impl_not_found` | 映射 | 追踪耗尽仍没有 vendor 域最终接口函数（类成员函数或自由函数） |
```

- [x] **Step 4: 验证替换结果**

Run:

```bash
grep -q '最终自由函数接口的 `impl_class` 与 `impl_class_file` 为 `null`；`evidence_chain` 末尾使用其命名空间限定名' HDF-api-finder/SKILL.md && echo "7.5_note=present"
grep -q '追踪耗尽仍没有 vendor 域最终接口函数（类成员函数或自由函数）' HDF-api-finder/SKILL.md && echo "7.6_row=present"
grep -n '追踪耗尽仍没有 vendor 类成员函数作为最终接口' HDF-api-finder/SKILL.md || echo "old_7.6=absent"
```

Expected: 2 个 `present`、1 个 `absent`。

---

## Task 7: 更新 8.1、8.2、8.3 与第二节完成报告

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（8.1 规则 5、8.2 自检 4/5/12、第二节与 8.3 完成报告）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '`impl_classes` 只收录最终 vendor 类' HDF-api-finder/SKILL.md
grep -n '每个有 case 的入口 `impl_classes` 非空' HDF-api-finder/SKILL.md
grep -n '且所属类也在 vendor 域' HDF-api-finder/SKILL.md
grep -n '所属类也在 vendor' HDF-api-finder/SKILL.md
grep -c '最终 impl 类数量；' HDF-api-finder/SKILL.md
```

Expected: 前四处都有命中；`最终 impl 类数量；` 共 2 处（第二节完成报告与 8.3）。

- [x] **Step 2: 替换 8.1 规则 5**

将：

```text
5. `impl_classes` 只收录最终 vendor 类，按 `(impl_class, impl_file)` 去重，按 `impl_class` → `impl_file` 排序。
```

替换为：

```text
5. `impl_classes` 只收录类成员最终接口所属的最终 vendor 类，按 `(impl_class, impl_file)` 去重，按 `impl_class` → `impl_file` 排序；允许为空（全部最终接口均为自由函数时）。
```

- [x] **Step 3: 替换 8.2 自检第 4 项**

将：

```text
4. 每个有 case 的入口 `impl_classes` 非空；每个 case 的最终 impl 函数所属类在 `impl_classes` 中。
```

替换为：

```text
4. 每个 case 的最终接口若为类成员函数，其 `(impl_class, impl_file)` 必须在 `impl_classes` 中；全部最终接口均为自由函数时 `impl_classes` 允许为空，入口仍必须保留。
```

- [x] **Step 4: 替换 8.2 自检第 5 项**

将：

```text
5. 每条最终 case 的 stub 接口函数定义在 stub 域且文件在 `scope_stub_files.txt` 内；impl 接口函数定义在 vendor 域、文件在 `scope_vendor_files.txt` 内，且所属类也在 vendor 域。
```

替换为：

```text
5. 每条最终 case 的 stub 接口函数定义在 stub 域且文件在 `scope_stub_files.txt` 内；impl 接口函数定义在 vendor 域、文件在 `scope_vendor_files.txt` 内；若 impl 接口函数为类成员函数，所属类也在 vendor 域。
```

- [x] **Step 5: 替换 8.2 自检第 12 项**

将：

```text
12. 每个最终接口函数定义在 vendor、文件在 `scope_vendor_files.txt` 内，所属类也在 vendor。
```

替换为：

```text
12. 每个最终接口函数定义在 vendor、文件在 `scope_vendor_files.txt` 内；若为类成员函数，所属类也在 vendor。
```

- [x] **Step 6: 替换完成报告的类数量统计（两处）**

将（使用 replace_all，第二节完成报告与 8.3 各一处）：

```text
- stub 类数量、初始 impl 类数量、最终 impl 类数量；
```

替换为：

```text
- stub 类数量、初始 impl 类数量、最终 impl 类数量（仅类成员最终接口）、最终自由函数接口数；
```

- [x] **Step 7: 验证替换结果**

Run:

```bash
grep -q '`impl_classes` 只收录类成员最终接口所属的最终 vendor 类' HDF-api-finder/SKILL.md && echo "8.1_rule5=present"
grep -q '全部最终接口均为自由函数时 `impl_classes` 允许为空，入口仍必须保留' HDF-api-finder/SKILL.md && echo "8.2_item4=present"
grep -q '若 impl 接口函数为类成员函数，所属类也在 vendor 域' HDF-api-finder/SKILL.md && echo "8.2_item5=present"
grep -q '若为类成员函数，所属类也在 vendor。' HDF-api-finder/SKILL.md && echo "8.2_item12=present"
grep -c '最终 impl 类数量（仅类成员最终接口）、最终自由函数接口数；' HDF-api-finder/SKILL.md
grep -n '每个有 case 的入口 `impl_classes` 非空' HDF-api-finder/SKILL.md || echo "old_selfcheck4=absent"
grep -n '且所属类也在 vendor 域' HDF-api-finder/SKILL.md || echo "old_class_check=absent"
grep -q '最终 impl 类数量；' HDF-api-finder/SKILL.md || echo "old_report_count=absent"
```

Expected: 4 个 `present`；计数为 `2`；`old_selfcheck4=absent`、`old_class_check=absent`、`old_report_count=absent`。

---

## Task 8: 更新术语表与抗理性化表

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第十章术语表、第十一章抗理性化检查）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '最终 impl 类 | 最终接口函数所属的 vendor 类' HDF-api-finder/SKILL.md
grep -n 'impl 接口函数 | stub 接口经初始/中间 impl 最终调用的、定义在 vendor 域的类成员函数' HDF-api-finder/SKILL.md
grep -n 'vendor free function 也算接口' HDF-api-finder/SKILL.md
grep -n '直到 vendor 类成员或明确终止条件' HDF-api-finder/SKILL.md
```

Expected: 四处都有命中。

- [x] **Step 2: 替换术语「最终 impl 类」并新增「最终自由函数接口」**

将：

```text
| 最终 impl 类 | 最终接口函数所属的 vendor 类，类定义与函数定义都必须在 vendor |
```

替换为：

```text
| 最终 impl 类 | 类成员形式的最终接口函数所属的 vendor 类，类定义与函数定义都必须在 vendor；最终自由函数接口不属于任何类 |
| 最终自由函数接口 | 定义在 vendor 域、不属于任何类的最终接口函数 |
```

- [x] **Step 3: 替换术语「impl 接口函数」**

将：

```text
| impl 接口函数 | stub 接口经初始/中间 impl 最终调用的、定义在 vendor 域的类成员函数；即最终接口函数 |
```

替换为：

```text
| impl 接口函数 | stub 接口经初始/中间 impl 最终调用的、定义在 vendor 域的最终接口函数（vendor 类成员函数或 vendor 自由函数） |
```

- [x] **Step 4: 替换抗理性化表中的两条并新增一条**

将：

```text
| “追踪太深，先停在这里” | 不设深度上限，直到 vendor 类成员或明确终止条件。 |
| “vendor free function 也算接口” | 不算；继续追到 vendor 类成员函数。 |
```

替换为：

```text
| “追踪太深，先停在这里” | 不设深度上限，直到 vendor 类成员、vendor 自由函数或明确终止条件。 |
| “vendor free function 只是中间 helper” | 不是；vendor 自由函数与 vendor 类成员函数同等作为最终接口，到达即终止。 |
| “自由函数没有类，跳过不报” | 不允许；按最终接口输出，`impl_class` 记 null。 |
```

- [x] **Step 5: 验证替换结果**

Run:

```bash
grep -q '最终自由函数接口 | 定义在 vendor 域、不属于任何类的最终接口函数' HDF-api-finder/SKILL.md && echo "glossary_freefn=present"
grep -q '最终接口函数（vendor 类成员函数或 vendor 自由函数）' HDF-api-finder/SKILL.md && echo "glossary_implfn=present"
grep -q '“vendor free function 只是中间 helper”' HDF-api-finder/SKILL.md && echo "anti_rational=present"
grep -q '“自由函数没有类，跳过不报”' HDF-api-finder/SKILL.md && echo "anti_rational_new=present"
grep -n '“vendor free function 也算接口”' HDF-api-finder/SKILL.md || echo "old_anti_rational=absent"
grep -n '直到 vendor 类成员或明确终止条件' HDF-api-finder/SKILL.md || echo "old_depth_row=absent"
```

Expected: 4 个 `present`、2 个 `absent`。

- [x] **Step 6: 修复残留在抗理性化表中的旧规则表述（质量复核发现）**

将：

```text
| 中间 impl 类 | 初始 impl 在 system 时的透传类；其接口方法不是最终接口 |
```

替换为：

```text
| 中间 impl 类 | 初始 impl 在 system 时的透传类；其接口方法不是最终接口 |
| 最终接口函数 | vendor 类成员函数（类定义与函数定义都在 vendor）或 vendor 自由函数（函数定义在 vendor 域） |
```

将：

```text
| “system impl 就是最终接口” | 不是；system 只是中间层，必须追到 vendor 类成员函数。 |
```

替换为：

```text
| “system impl 就是最终接口” | 不是；system 只是中间层，必须追到 vendor 域最终接口函数（类成员函数或自由函数）。 |
```

将：

```text
| “追踪太深，先停在这里” | 不设深度上限，直到 vendor 类成员、vendor 自由函数或明确终止条件。 |
```

替换为：

```text
| “追踪太深，先停在这里” | 不设深度上限，直到 vendor 类成员函数、vendor 自由函数或明确终止条件。 |
```

将：

```text
| “自由函数没有类，跳过不报” | 不允许；按最终接口输出，`impl_class` 记 null。 |
```

替换为：

```text
| “自由函数没有类，跳过不报” | 不允许；按最终接口输出，`impl_class` / `impl_class_file` 记 null。 |
```

Run:

```bash
grep -q '最终接口函数 | vendor 类成员函数（类定义与函数定义都在 vendor）或 vendor 自由函数（函数定义在 vendor 域）' HDF-api-finder/SKILL.md && echo "glossary_finalfn=present"
grep -q '必须追到 vendor 域最终接口函数（类成员函数或自由函数）' HDF-api-finder/SKILL.md && echo "anti_system=present"
grep -q '直到 vendor 类成员函数、vendor 自由函数或明确终止条件' HDF-api-finder/SKILL.md && echo "anti_depth=present"
grep -q '`impl_class` / `impl_class_file` 记 null' HDF-api-finder/SKILL.md && echo "anti_null=present"
grep -n '追到 vendor 类成员函数' HDF-api-finder/SKILL.md || echo "old_anti_system=absent"
```

Expected: 4 个 `present`、1 个 `absent`。

---

## Task 9: 新增示例六

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第十二章使用示例，文件末尾）

- [x] **Step 1: 确认文件末尾内容**

Run:

```bash
tail -n 20 HDF-api-finder/SKILL.md
```

Expected: 示例五的 JSON 代码块之后还有一句 `追踪继续，不中止、不判并列歧义。`，文件以该句结束。

- [x] **Step 2: 在文件末尾追加示例六**

将文件最后一句：

```text
追踪继续，不中止、不判并列歧义。
```

替换为：

````text
追踪继续，不中止、不判并列歧义。

### 示例六：vendor 自由函数接口

`DemoSystemService::VendorOpen` 调用 vendor 自由函数 `demo::hal_open(...)`；该函数定义在 `/abs/vendor/demo_hal.cpp`，不属于任何类。`demo::hal_open` 即最终接口，分支在该函数终止。

1. `02_impl.json` 的 `final_impl_functions` 元素：`function = "demo::hal_open"`，`impl_class = null`，`impl_class_file = null`，`trace` 含 `DemoSystemService::VendorOpen`。
2. 最终 JSON 对该 cmdId 输出 case：`impl_interface_function = "demo::hal_open"`，`impl_interface_file = "/abs/vendor/demo_hal.cpp"`。
3. 该入口没有类成员最终接口时，`impl_classes` 为空数组，入口仍正常输出。
````

- [x] **Step 3: 验证新增结果**

Run:

```bash
grep -q '^### 示例六：vendor 自由函数接口' HDF-api-finder/SKILL.md && echo "example6_title=present"
grep -q 'impl_class = null' HDF-api-finder/SKILL.md && echo "example6_null=present"
grep -q 'impl_classes` 为空数组，入口仍正常输出' HDF-api-finder/SKILL.md && echo "example6_empty=present"
grep -c '^### 示例' HDF-api-finder/SKILL.md
```

Expected: 3 个 `present`；示例计数为 `6`。

---

## Task 10: 结构与禁用内容验证

**Files:**
- Verify: `HDF-api-finder/SKILL.md`

- [x] **Step 1: 旧规则清零**

Run:

```bash
grep -n 'vendor free function 只是中间 helper，不算最终接口' HDF-api-finder/SKILL.md || echo "old_free_rule=absent"
grep -n 'vendor free function、system 函数、非 vendor 类成员只是中间层' HDF-api-finder/SKILL.md || echo "old_final_rule=absent"
grep -n '最终 impl 类与最终接口函数必须在 vendor 域' HDF-api-finder/SKILL.md || echo "old_domain_rule=absent"
grep -n '每个有 case 的入口 `impl_classes` 非空' HDF-api-finder/SKILL.md || echo "old_selfcheck=absent"
grep -n '且所属类也在 vendor 域' HDF-api-finder/SKILL.md || echo "old_class_check=absent"
grep -n '追到 vendor 类成员函数' HDF-api-finder/SKILL.md || echo "old_anti_system=absent"
```

Expected: 6 个 `absent`。

- [x] **Step 2: 新规则到位**

Run:

```bash
grep -q '或 vendor 自由函数（函数定义在 vendor 域）' HDF-api-finder/SKILL.md && echo "principle10=present"
grep -q '最终接口（任一满足）：vendor 类成员函数且类定义与函数定义都在 vendor；vendor 自由函数且函数定义在 vendor' HDF-api-finder/SKILL.md && echo "6.5_final=present"
grep -q 'vendor 自由函数与 vendor 类成员函数同等作为最终接口，到达即终止该分支' HDF-api-finder/SKILL.md && echo "6.5_supp=present"
grep -q '最终自由函数接口的 `impl_class` 与 `impl_class_file` 为 `null`' HDF-api-finder/SKILL.md && echo "null_semantics=present"
grep -q '允许为空（全部最终接口均为自由函数时）' HDF-api-finder/SKILL.md && echo "empty_allowed=present"
grep -q '追踪耗尽仍没有 vendor 域最终接口函数（类成员函数或自由函数）' HDF-api-finder/SKILL.md && echo "unresolved_row=present"
grep -q '最终自由函数接口 | 定义在 vendor 域、不属于任何类的最终接口函数' HDF-api-finder/SKILL.md && echo "glossary=present"
grep -q '“自由函数没有类，跳过不报”' HDF-api-finder/SKILL.md && echo "anti_rational=present"
grep -q '最终接口函数 | vendor 类成员函数（类定义与函数定义都在 vendor）或 vendor 自由函数（函数定义在 vendor 域）' HDF-api-finder/SKILL.md && echo "glossary_finalfn=present"
grep -q '必须追到 vendor 域最终接口函数（类成员函数或自由函数）' HDF-api-finder/SKILL.md && echo "anti_system=present"
grep -q '`impl_class` / `impl_class_file` 记 null' HDF-api-finder/SKILL.md && echo "anti_null=present"
grep -q '^### 示例六：vendor 自由函数接口' HDF-api-finder/SKILL.md && echo "example6=present"
```

Expected: 12 个 `present`。

- [x] **Step 3: 确认 `vendor free function` 只出现在抗理性化旧观念中**

Run:

```bash
grep -n 'vendor free function' HDF-api-finder/SKILL.md
```

Expected: 仅 1 处命中，位于抗理性化表：

```text
| “vendor free function 只是中间 helper” | 不是；vendor 自由函数与 vendor 类成员函数同等作为最终接口，到达即终止。 |
```

若出现第二处，说明 6.5 或核心原则仍有残留，回到 Task 2/Task 4 修复。

- [x] **Step 4: 结构、JSON 与禁用内容**

Run:

```bash
grep -c '^## ' HDF-api-finder/SKILL.md
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
print(f"json_blocks={len(blocks)}")
for b in blocks:
    json.loads(b)
print("json_valid=ok")
PY
grep -nE 'codegraph|hal-service-api-finder|NfcVendorAdaptions|nfcHalInf|phNxpNciHal_open' HDF-api-finder/SKILL.md || echo "forbidden_strings=none"
grep -n 'trace_ambiguous' HDF-api-finder/SKILL.md || echo "trace_ambiguous=absent"
```

Expected: 章节数 `12`；`json_valid=ok`；`forbidden_strings=none`；`trace_ambiguous=absent`。

- [x] **Step 5: 发现缺口时修复并重跑**

任一检查缺少 `present` 或未输出 `absent`/`none` → 回到对应任务补充，再重跑 Task 10，直到全部通过。

---

## Task 11: 流程一致性验证

**Files:**
- Verify: `HDF-api-finder/SKILL.md`

- [x] **Step 1: 阶段边界与既有规则不变**

Run:

```bash
grep -q '阶段二做三件事' HDF-api-finder/SKILL.md && echo "phase2_scope=present"
grep -q '阶段三只做一件事' HDF-api-finder/SKILL.md && echo "phase3_scope=present"
grep -q '阶段三不解析 impl 方法定义' HDF-api-finder/SKILL.md && echo "phase3_lookup=present"
grep -q '不设深度上限，不设扇出上限' HDF-api-finder/SKILL.md && echo "no_limit=present"
grep -q '每分支独立' HDF-api-finder/SKILL.md && echo "branch_visited=present"
grep -q 'unresolved_branches' HDF-api-finder/SKILL.md && echo "branch_failure=present"
grep -q '方法定义在 vendor 类成员函数中，且该类的完整定义也在 vendor → 直接作为最终接口' HDF-api-finder/SKILL.md && echo "6.4_direct=present"
```

Expected: 7 个 `present`。

- [x] **Step 2: stub 侧 free 函数排除规则未被误改**

Run:

```bash
grep -q 'free 函数、`GetDescriptor`' HDF-api-finder/SKILL.md && echo "stub_side_exclusion=present"
grep -q '构造/析构/运算符、free 函数' HDF-api-finder/SKILL.md && echo "case_side_exclusion=present"
```

Expected: 2 个 `present`。

- [x] **Step 3: 最终 JSON schema 未变化**

Run:

```bash
git diff ae6dfd5 -- HDF-api-finder/SKILL.md | grep -E '^[+-].*"(impl_interface_function|impl_interface_file|impl_class|cmd_id_macro)"' || echo "json_sample_fields_unchanged"
```

Expected: `json_sample_fields_unchanged`（本次只改规则措辞与示例六，不改两个 JSON 样例的字段）。

- [x] **Step 4: 自由函数判定与类成员判定并存**

Run:

```bash
grep -q 'vendor 类成员函数且类定义与函数定义都在 vendor；vendor 自由函数且函数定义在 vendor' HDF-api-finder/SKILL.md && echo "both_kinds=present"
grep -q '若为类成员函数，其类定义也必须在 vendor' HDF-api-finder/SKILL.md && echo "class_conditional=present"
grep -q '最终自由函数接口不产生类条目' HDF-api-finder/SKILL.md && echo "no_class_entry=present"
```

Expected: 3 个 `present`。

- [x] **Step 5: 发现不一致时修复**

若上述检查失败，先回到对应 Task 修复，再重跑 Task 10 与 Task 11。

---

## Task 12: 最终通读与交付报告

**Files:**
- Verify: `HDF-api-finder/SKILL.md`

- [x] **Step 1: 分段通读全文**

Run:

```bash
wc -l HDF-api-finder/SKILL.md
```

Expected: 记录行数。分 3 段 Read 全文（每段不超过 300 行），检查：

- 12 个章节齐全、顺序正确；
- 核心原则 3/10、域归属判定、约束规则 4/5 口径一致；
- 6.4 仍只处理类成员直通，6.5 最终/中间层分类正确；
- `impl_classes` 在 6.6、8.1、8.2、术语表、抗理性化表中的语义一致；
- 示例六使用中性 demo 命名，示例一/二/四/五未被改动；
- 无旧规则句、无 `impl_kind`/`impl_functions` 等新字段、无代码图工具名称、无其他 skill 名称。

- [x] **Step 2: 最终结构复检**

Run:

```bash
grep -c '^## ' HDF-api-finder/SKILL.md
grep -c '^### 示例' HDF-api-finder/SKILL.md
grep -n 'vendor free function' HDF-api-finder/SKILL.md
```

Expected: `12`、`6`；`vendor free function` 仅 1 处（抗理性化表旧观念）。

- [x] **Step 3: 交付报告**

报告：

- SKILL.md 绝对路径；
- 行数与字节数；
- 章节数（12）与示例数（6）；
- JSON 代码块数量与 `json_valid=ok`；
- 旧规则清零与新规则检查结果；
- 明确声明：仅验证流程与文本一致性，未在真实项目中验证接口识别准确性；真实项目回归需要用户环境配合。

---

## Task 13: 提交 SKILL.md

**Files:**
- Modify: `HDF-api-finder/SKILL.md`

- [x] **Step 1: 检查待提交内容**

Run:

```bash
git status --short
git diff --stat
git diff -- HDF-api-finder/SKILL.md | head -200
```

Expected: 只有 `HDF-api-finder/SKILL.md` 被修改；diff 仅涉及第 1、3、6.5、6.6、7.5、7.6、8.1、8.2、8.3、第二节报告、第十章约束/术语表、第十一章抗理性化、第十二章示例六。

- [x] **Step 2: 提交**

Run:

```bash
git add HDF-api-finder/SKILL.md
git -c user.name='siakam43' -c user.email='429872468@qq.com' commit -m "$(cat <<'EOF'
docs(HDF-api-finder): treat vendor free functions as final interfaces

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

Expected: 提交成功，包含 1 个文件。

- [x] **Step 3: 验证提交**

Run:

```bash
git log -1 --format='%h | %an <%ae> | %s'
git status --short
git branch --show-current
```

Expected: 作者为 `siakam43 <429872468@qq.com>`；工作区干净；当前分支为 `dev`。

---

## Task 14: 标记设计与计划完成

**Files:**
- Modify: `docs/superpowers/specs/2026-11-09-hdf-api-finder-vendor-free-function-interface-design.md`
- Modify: `docs/superpowers/plans/2026-11-09-hdf-api-finder-vendor-free-function-interface.md`

- [x] **Step 1: 更新增量设计状态**

将：

```text
- 状态：待实现
```

替换为：

```text
- 状态：已实现，待用户验证
```

- [x] **Step 2: 勾选本计划全部复选框**

对本计划文件执行 replace_all：

```text
- [x] → - [x]
```

- [x] **Step 3: 提交完成标记**

Run:

```bash
git add docs/superpowers/specs/2026-11-09-hdf-api-finder-vendor-free-function-interface-design.md docs/superpowers/plans/2026-11-09-hdf-api-finder-vendor-free-function-interface.md
git -c user.name='siakam43' -c user.email='429872468@qq.com' commit -m "$(cat <<'EOF'
docs(HDF-api-finder): mark vendor free-function plan complete

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
git log -1 --format='%h | %an <%ae> | %s'
git status --short
```

Expected: 提交成功，包含 2 个文件；工作区干净；作者为 `siakam43 <429872468@qq.com>`。

- [x] **Step 4: 最终分支状态**

Run:

```bash
git log --oneline --decorate -5
git branch --show-current
```

Expected: `dev` 分支领先 `main` 4 个提交（设计、计划、SKILL.md、完成标记）；未合并到 `main`；未 push。
