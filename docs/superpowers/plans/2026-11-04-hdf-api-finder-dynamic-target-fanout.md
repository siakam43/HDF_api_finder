# HDF-api-finder 动态目标扇出实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修改 `HDF-api-finder/SKILL.md`，把单线路限定为调用点选择；调用目标动态不唯一时全部分支追踪，`02_impl.json` 改为 `final_impl_functions` 数组，最终 JSON 并列输出多个最终接口。

**Architecture:** 调用点层继续多信号打分（并列按源码顺序取第一个并记录 `tie_break`）；目标层对函数指针、回调、`dlsym` 符号、多个 virtual override 做目标级扇出；每个分支独立 trace 与 visited，成功分支合并去重，失败分支隔离记录。阶段一、conf、scope、搜索纪律、输出路径、原子写出不变。

**Tech Stack:** Markdown、JSON、bash（grep/find/xargs）、Read、Write。

---

## 0. 输入与约束

**增量设计（权威来源）：**

```text
docs/superpowers/specs/2026-11-04-hdf-api-finder-dynamic-target-fanout-design.md
```

**前置设计（历史基线，冲突处以增量设计为准）：**

```text
docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md
docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md
```

**交付/修改文件：**

| 文件 | 责任 |
|------|------|
| `HDF-api-finder/SKILL.md` | 唯一需要修改的交付物 |
| `docs/superpowers/specs/2026-11-04-hdf-api-finder-dynamic-target-fanout-design.md` | 增量设计；已在提交 `07a1704` 中提交 |
| `docs/superpowers/plans/2026-11-04-hdf-api-finder-dynamic-target-fanout.md` | 本计划；Task 1 提交 |

**全局约束：**

- 修改前先完整读取增量设计 §3–§13；所有新规则以该文档为准。
- 示例统一使用中性 demo 命名；不使用需求原始示例中的 Nfc 命名。
- SKILL.md 正文不得出现任何代码图/MCP 代码探索工具名称，不得体现与其他 skill 的关联性。
- 保留阶段一、conf、scope、搜索纪律、输出路径、原子写出、无自动恢复、无 sub-agent。
- 提交使用 `git -c user.name='siakam43' -c user.email='429872468@qq.com' commit`，不修改 git config。
- 本次未收到推送授权，只提交本地，不执行 `git push`。
- 每个 Task 完成后运行验证命令；发现 `missing` 或缺 `present` 时先修复再进入下一 Task。

---

## Task 1: 提交实现计划

**Files:**
- Add: `docs/superpowers/plans/2026-11-04-hdf-api-finder-dynamic-target-fanout.md`

- [ ] **Step 1: 确认工作区状态**

Run:

```bash
git status --short
```

Expected: 本计划为未跟踪文件；`HDF-api-finder/SKILL.md` 尚未修改；增量设计无未提交改动。

- [ ] **Step 2: 提交计划**

Run:

```bash
git add docs/superpowers/plans/2026-11-04-hdf-api-finder-dynamic-target-fanout.md
git -c user.name='siakam43' -c user.email='429872468@qq.com' commit -m "$(cat <<'EOF'
docs(HDF-api-finder): add dynamic target fanout implementation plan

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

Expected: 提交成功，包含 1 个文件。

- [ ] **Step 3: 验证提交**

Run:

```bash
git log -1 --format='%h | %an <%ae> | %s'
git status --short
```

Expected: 作者为 `siakam43 <429872468@qq.com>`；工作区无未提交改动。

---

## Task 2: 更新第一章核心原则

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第一章）

- [ ] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '证据优先，不臆造' HDF-api-finder/SKILL.md
grep -n '单线路追踪与最终接口' HDF-api-finder/SKILL.md
```

Expected: 两处都有命中。

- [ ] **Step 2: 替换原则 5**

将：

```text
5. **证据优先，不臆造。** 每个入口、每个 cmdId 值、每个 stub/impl 对应关系都要有代码证据；无法确定的分析项写入 unresolved，记录 `detail` 与 `tried`，不得猜测。
```

替换为：

```text
5. **证据优先，不臆造。** 每个入口、每个 cmdId 值、每个 stub/impl 对应关系都要有代码证据；调用点打分并列必须记录 `tie_break`（候选、分数、选中项），目标级扇出必须记录 `target_evidence`；无法确定的分析项写入 unresolved，记录 `detail` 与 `tried`，不得猜测。
```

- [ ] **Step 3: 替换原则 10**

将：

```text
10. **单线路追踪与最终接口。** system 透传层每层用多信号打分，只选最可能的下一跳；并列记 `trace_ambiguous`，不猜测；最终接口必须是 vendor 类成员函数，且类定义与函数定义都在 vendor；vendor free function、system 函数、非 vendor 类成员只是中间层。
```

替换为：

```text
10. **调用点单线路与目标级扇出。** 调用点选择用多信号打分只选一个，并列按源码顺序取第一个并记录 `tie_break`；被选中调用点的目标动态不唯一时（函数指针、回调、`dlsym` 符号、多个 virtual override），把 vendor + system 域内全部静态可枚举目标作为下一跳分支继续追踪，不得判并列歧义、不得只选一个；最终接口必须是 vendor 类成员函数，且类定义与函数定义都在 vendor；vendor free function、system 函数、非 vendor 类成员只是中间层。
```

- [ ] **Step 4: 验证第一章**

Run:

```bash
grep -q '调用点单线路与目标级扇出' HDF-api-finder/SKILL.md && echo "principle10=present"
grep -q 'tie_break' HDF-api-finder/SKILL.md && echo "tiebreak=present"
grep -q '目标级扇出必须记录 `target_evidence`' HDF-api-finder/SKILL.md && echo "evidence=present"
```

Expected: 3 个 `present`。

---

## Task 3: 重写第六章阶段二

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第六章）

- [ ] **Step 1: 确认第六章边界**

Run:

```bash
grep -n '^## 六、\|^## 七、' HDF-api-finder/SKILL.md
grep -n '^### 6.4\|^### 6.5\|^### 6.6\|^### 6.7' HDF-api-finder/SKILL.md
```

Expected: 得到第六章、第七章起始行与 6.4–6.7 小节行号。

- [ ] **Step 2: 替换 6.4 第 4、5 步**

将：

```text
4. 方法定义在 vendor 类成员函数中，且该类的完整定义也在 vendor → 直接作为最终接口：记录 `final_impl_function/file/line`、`final_impl_class/class_file`，`trace = []`。
5. 其他情况 → 进入 6.5 单线路追踪。
```

替换为：

```text
4. 方法定义在 vendor 类成员函数中，且该类的完整定义也在 vendor → 直接作为最终接口：写入 `final_impl_functions` 数组（一个元素，`trace = []`，`target_evidence = []`）。
5. 其他情况 → 进入 6.5 目标级扇出追踪。
```

- [ ] **Step 3: 用增量设计 §4 重写 6.5**

将 `### 6.5 单线路追踪` 到 `### 6.6 02_impl.json` 之前的内容整体替换为：

````markdown
### 6.5 目标级扇出追踪

从当前中间函数出发循环。调用点选择保持单线路；调用目标动态不唯一时全部分支追踪。

1. 完整读取当前函数定义；定义不在 scope 或无法唯一确定 → 当前分支 `definition_not_found`，终止。
2. 收集调用点候选：成员调用、虚函数调用、free 函数调用、可静态确定的函数指针/回调；排除日志/断言/纯校验/构造/析构/运算符/框架 sbuf-parcel 转换。
3. 解析每个调用点的目标集合：
   - 直接调用：定义唯一 → 单目标；重载无法按实参唯一确定 → 该调用点 `impl_overload_ambiguous`，不参与打分；定义不在 scope → `definition_not_found`，不参与打分。
   - 虚函数调用：收集 vendor + system 域内全部 override；静态方法非纯虚且 scope 内有函数体时，静态定义本身也计入；无 override 且无函数体 → `definition_not_found`，不参与打分。
   - 函数指针/回调/成员函数指针调用：在 vendor + system 域回溯该指针的赋值点（`=&F`、`=F`、`=&C::M`、`=dlsym(...)`、注册点直接赋值）。`dlsym` 第二参数按字面量段固定、运行时段通配解析；运行时段先在 scope 内枚举常量值（字面量、`const`、`constexpr`、枚举、宏、多点赋值），可枚举则精确匹配，否则通配；按非限定函数名匹配有函数体的函数定义。所有赋值目标与符号匹配目标按 `(函数, 文件, 行)` 去重。找不到赋值点，或符号表达式无字面量段且运行时段不可枚举 → `target_not_enumerable`，不参与打分。
   - 目标集合为空 → 该调用点不参与打分；全部调用点都为空 → 当前分支 `trace_no_candidate`，终止。
4. 多信号打分（评分对象是调用表达式；名称与命名空间信号按目标集合中最匹配的目标计算）：
   1. 实参包含 stub 传入的 data/reply 或派生值；
   2. 方法名相同，或去掉命名空间/类限定后是前缀/后缀扩展且多出部分 ≤ 8 字符；
   3. 接收者是下一层服务对象，不是日志/校验/框架 helper；
   4. 返回值被返回或用于构造 reply；
   5. 命名空间/接口类型匹配。
5. 唯一最高分 → 选中；并列 → 按源码顺序取第一个，并在 trace 中记录 `tie_break`（候选、分数、选中项、原因 `source_order`）。
6. 对选中调用点的目标集合逐目标建立分支：
   - 目标集合大小 1 → 单分支；大于 1 → 目标级扇出，每个目标创建独立分支，继承当前分支 trace 与 visited 副本；可递归扇出。
   - 目标按 `(定义文件, 定义行)` 升序处理。
   - 最终接口：vendor 类成员函数，且类定义与函数定义都在 vendor → 记录到 `final_impl_functions`（含该分支 trace 与 `target_evidence`），分支结束。
   - 中间层：system 函数、vendor free function、所属类不在 vendor、vendor 类但函数/类定义不在 vendor → trace 追加该目标与调用证据，继续步骤 1。
7. visited 每分支独立，键 `(文件, 函数签名行)`，只含当前分支祖先；同一分支重复 → `cycle`。分支间不共享 visited，DAG 收敛不视为环。
8. 分支终止原因：`definition_not_found`、`target_not_enumerable`、`trace_no_candidate`、`cycle`、`final_impl_not_found`；失败分支记入 `unresolved_branches`，其他分支继续。
9. 不设深度上限，不设扇出上限。
10. 合并：最终接口按 `(函数, 文件, 行)` 去重；同一最终接口来自多条分支时保留排序后第一条分支的 trace，`target_evidence` 合并去重；`final_impl_functions` 非空 → 映射记录 `status = "resolved"`。

补充：

- vendor free function 只是中间 helper，不算最终接口。
- 最终 impl 类定义与最终接口函数定义都必须在 vendor。
- 目标枚举只在 vendor + system 域进行，stub 域不参与。
- `tie_break` 与 `target_evidence` 只用于中间证据，不进入最终 JSON。
````

- [ ] **Step 4: 重写 6.6 02_impl.json**

将 `### 6.6 02_impl.json` 到 `### 6.7 入口状态与失败处理` 之前的内容整体替换为：

````markdown
### 6.6 02_impl.json

```json
{
  "service_name": "demo_service",
  "entries": [
    {
      "on_remote_request_file": "/abs/stub/demo_interface_stub.cpp",
      "on_remote_request_line": 45,
      "stub_class": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub",
      "interface_type": "OHOS::HDI::Demo::V1_0::IDemoInterface",
      "impl_member": {
        "name": "impl_",
        "type": "sptr<IDemoInterface>",
        "file": "/abs/stub/demo_interface_stub.h",
        "line": 35
      },
      "initial_impl_class": "OHOS::HDI::Demo::V1_0::DemoSystemService",
      "initial_impl_file": "/abs/system/demo_system_service.h",
      "initial_impl_domain": "system",
      "initial_strategy": "feature_match",
      "initial_evidence": [
        {
          "file": "/abs/system/demo_system_service.h",
          "line": 20,
          "expression": "class DemoSystemService : public IDemoInterface {"
        }
      ],
      "stub_function_map": [
        {
          "stub_interface_function": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::VendorOpen",
          "stub_interface_file": "/abs/stub/demo_interface_stub.cpp",
          "stub_interface_line": 60,
          "initial_impl_function": "OHOS::HDI::Demo::V1_0::DemoSystemService::VendorOpen",
          "initial_impl_file": "/abs/system/demo_system_service.cpp",
          "initial_impl_line": 40,
          "final_impl_functions": [
            {
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorA::HalOpen",
              "file": "/abs/vendor/demo_vendor_a.cpp",
              "line": 80,
              "impl_class": "OHOS::HDI::Demo::V1_0::DemoVendorA",
              "impl_class_file": "/abs/vendor/demo_vendor_a.h",
              "trace": [
                {
                  "function": "OHOS::HDI::Demo::V1_0::DemoSystemService::VendorOpen",
                  "file": "/abs/system/demo_system_service.cpp",
                  "line": 40,
                  "evidence": "demoHalInf.demoHalOpen(pCback, pDataCback)",
                  "tie_break": {
                    "candidates": [
                      {"call": "demoHalInf.demoHalOpen(...)", "score": 5},
                      {"call": "demoHalInf.demoHalOpenEx(...)", "score": 5}
                    ],
                    "chosen": "demoHalInf.demoHalOpen(...)",
                    "reason": "source_order"
                  }
                }
              ],
              "target_evidence": [
                {
                  "file": "/abs/vendor/demo_vendor_a.cpp",
                  "line": 120,
                  "expression": "demoHalInf.demoHalOpen = (fn)dlsym(handle, \"demo_hal_open\" + variant)",
                  "match": "dlsym_glob",
                  "pattern": "demo_hal_open*"
                }
              ]
            }
          ],
          "unresolved_branches": [],
          "status": "resolved",
          "unresolved_reason": null
        }
      ],
      "impl_classes": [
        {
          "impl_class": "OHOS::HDI::Demo::V1_0::DemoVendorA",
          "impl_file": "/abs/vendor/demo_vendor_a.h"
        }
      ],
      "status": "resolved",
      "unresolved_reason": null
    }
  ],
  "unresolved": []
}
```

字段与规则：

- `final_impl_functions` 每个元素对应一条扇出分支的最终接口；元素自带该分支的 `trace`（含初始 impl 方法，不含最终函数）与 `target_evidence`。
- `target_evidence` 汇总该分支上每次目标级选择产生的证据；直接单目标且无动态选择时可为空数组。
- `target_evidence.match` 取值：`direct_assign` / `dlsym_exact` / `dlsym_glob` / `virtual_override` / `base_implementation`。
- `tie_break` 为可选字段，仅出现在发生并列选择的 trace 条目；`candidates` 记录参与并列的调用与分数，`chosen` 记录选中项，`reason` 固定为 `source_order`。
- `unresolved_branches` 记录失败分支：`kind` / `detail` / `tried` / 分支 `trace`；按 `(kind, detail)` 去重，`tried` 合并去重。
- `status = "resolved"` 当且仅当 `final_impl_functions` 非空；`unresolved_reason` 仅在 unresolved 时非 null。
- 元素按 `(函数, 文件, 行)` 去重，`target_evidence` 合并去重。
- 入口级 `impl_classes` 语义不变：全部最终 vendor 类按 `(impl_class, impl_file)` 去重排序。
- 写入失败 → 写 `error.json`（`phase=resolve_impl`，`code=snapshot_write_failed`），停止。
````

- [ ] **Step 5: 更新 6.7 入口状态与失败处理**

将：

```text
- 单映射失败 → unresolved，继续其他映射。
- 单入口全部映射失败 → entry unresolved，继续其他入口。
```

替换为：

```text
- 单分支失败 → 记入 `unresolved_branches`，同映射其他分支继续。
- 单映射全部分支失败 → 映射 unresolved，继续其他映射。
- 单入口全部映射失败 → entry unresolved，继续其他入口。
```

- [ ] **Step 6: 验证第六章**

Run:

```bash
for t in '6.5 目标级扇出追踪' 'final_impl_functions' 'unresolved_branches' 'target_evidence' 'tie_break' 'target_not_enumerable'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
awk '/^## 六、/{f=1} /^## 七、/{f=0} f' HDF-api-finder/SKILL.md | grep -q '单线路追踪' && echo "old_term_in_phase2=present" || echo "old_term_in_phase2=absent"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
objs = [json.loads(b) for b in blocks]
print(f"json_blocks={len(blocks)}")
recs = []
for obj in objs:
    if not isinstance(obj, dict) or 'entries' not in obj:
        continue
    for e in obj['entries']:
        for rec in e.get('stub_function_map', []):
            recs.append(rec)
assert recs, 'no stub_function_map record'
for rec in recs:
    assert 'final_impl_functions' in rec, 'missing final_impl_functions'
    assert 'final_impl_function' not in rec, 'old final_impl_function remains'
    assert 'trace' not in rec, 'top-level trace remains'
print("phase2_schema_ok")
PY
```

Expected: 无 `missing ...`；`phase2_schema_ok`；`old_term_in_phase2=absent`。

---

## Task 4: 更新第七章阶段三

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第七章）

- [ ] **Step 1: 确认第七章边界**

Run:

```bash
grep -n '^## 七、\|^## 八、' HDF-api-finder/SKILL.md
grep -n '^### 7.2\|^### 7.5\|^### 7.6' HDF-api-finder/SKILL.md
```

Expected: 得到第七章、第八章起始行与 7.2/7.5/7.6 小节行号。

- [ ] **Step 2: 替换 7.2 查表规则**

将：

```text
2. 映射记录 resolved → 用 `final_impl_function` / `final_impl_file` / `final_impl_class` / `final_impl_class_file` 生成结果；`evidence_chain` 记录 `[stub 调用] + trace + [final_impl_function]`。
3. 映射记录 unresolved → 把该记录的 `unresolved_reason` 写入用例 unresolved。
4. 一个 stub 函数对应多条 resolved 映射记录 → 生成多条 `(stub, final impl)` 对。
5. 阶段三不解析 impl 方法定义，不读取 system 调用链，也不做 impl 域判定。
```

替换为：

```text
2. 每条映射记录遍历 `final_impl_functions`，每个元素生成一条 `(stub 接口函数, 最终 impl 函数)` 对。
3. `impl_interfaces` 每个元素的 `evidence_chain` 为 `[stub 接口函数] + 该元素 trace + [该元素最终函数]`。
4. 映射记录无最终接口 → 把该记录的 `unresolved_reason` 写入用例 unresolved。
5. 映射记录有部分失败分支 → 成功分支照常生成对应对；失败分支写入用例 `unresolved`；用例 `status = "done"`。
6. 一个 stub 函数对应多条 resolved 映射记录 → 全部展开，生成多对。
7. 阶段三不解析 impl 方法定义，不读取 system 调用链，也不做 impl 域判定。
```

- [ ] **Step 3: 替换 7.5 03_cases.json 示例**

将 7.5 小节中的示例 JSON 整体替换为：

````markdown
```json
{
  "service_name": "demo_service",
  "entries": [
    {
      "on_remote_request_file": "/abs/stub/demo_interface_stub.cpp",
      "on_remote_request_line": 45,
      "cmd_param": "code",
      "data_param": "data",
      "cases": [
        {
          "id": 1,
          "case_site": {"file": "/abs/stub/demo_interface_stub.cpp", "line": 50},
          "case_expr": "CMD_DEMO_VENDOR_OPEN",
          "cmd_id_macro": "CMD_DEMO_VENDOR_OPEN",
          "cmd_id_value": 1,
          "outer_cmd_macro": null,
          "outer_cmd_value": null,
          "stub_interface_function": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::VendorOpen",
          "stub_interface_file": "/abs/stub/demo_interface_stub.cpp",
          "stub_interface_line": 60,
          "impl_interfaces": [
            {
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorA::HalOpen",
              "file": "/abs/vendor/demo_vendor_a.cpp",
              "line": 80,
              "impl_class": "OHOS::HDI::Demo::V1_0::DemoVendorA",
              "impl_class_file": "/abs/vendor/demo_vendor_a.h",
              "evidence": "demoHalInf.demoHalOpen(pCback, pDataCback)",
              "evidence_chain": [
                "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::VendorOpen",
                "OHOS::HDI::Demo::V1_0::DemoSystemService::VendorOpen",
                "OHOS::HDI::Demo::V1_0::DemoVendorA::HalOpen"
              ]
            },
            {
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorB::HalOpen",
              "file": "/abs/vendor/demo_vendor_b.cpp",
              "line": 90,
              "impl_class": "OHOS::HDI::Demo::V1_0::DemoVendorB",
              "impl_class_file": "/abs/vendor/demo_vendor_b.h",
              "evidence": "demoHalInf.demoHalOpen(pCback, pDataCback)",
              "evidence_chain": [
                "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::VendorOpen",
                "OHOS::HDI::Demo::V1_0::DemoSystemService::VendorOpen",
                "OHOS::HDI::Demo::V1_0::DemoVendorB::HalOpen"
              ]
            }
          ],
          "status": "done",
          "unresolved": null
        }
      ],
      "default_cases": [],
      "unresolved": []
    }
  ],
  "unresolved": []
}
```
````

- [ ] **Step 4: 更新 7.6 unresolved 表**

将：

```text
| `impl_overload_ambiguous` | 映射 | 方法或下一层调用重载无法按实参唯一确定 |
| `trace_ambiguous` | 映射 | system 层多个候选同等置信度，无法唯一选择 |
| `trace_no_candidate` | 映射 | system 层没有可信的下一跳候选 |
```

替换为：

```text
| `impl_overload_ambiguous` | 映射 | 方法或下一跳调用重载无法按实参唯一确定 |
| `target_not_enumerable` | 映射/分支 | 选中调用点的动态目标无法静态枚举：无赋值点、`dlsym` 表达式完全由运行时段构成、符号模式无字面量段等 |
| `trace_no_candidate` | 映射 | 当前函数没有可信的调用点候选 |
```

- [ ] **Step 5: 验证第七章**

Run:

```bash
grep -q 'target_not_enumerable' HDF-api-finder/SKILL.md && echo "target_not_enumerable=present"
awk '/^## 七、/{f=1} /^## 八、/{f=0} f' HDF-api-finder/SKILL.md | grep -q 'trace_ambiguous' && echo "trace_ambiguous_in_phase3=present" || echo "trace_ambiguous_in_phase3=absent"
grep -q 'evidence_chain' HDF-api-finder/SKILL.md && echo "evidence_chain=present"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
objs = [json.loads(b) for b in blocks]
print(f"json_blocks={len(blocks)}")
found = 0
for obj in objs:
    if not isinstance(obj, dict) or 'entries' not in obj:
        continue
    for e in obj['entries']:
        for case in e.get('cases', []):
            if 'impl_interfaces' not in case:
                continue
            assert 'evidence_chain' not in case, 'case-level evidence_chain remains'
            for item in case['impl_interfaces']:
                assert item.get('evidence_chain'), 'missing per-impl evidence_chain'
                found += 1
assert found >= 2, 'expected at least 2 impl_interfaces in example'
print("phase3_schema_ok")
PY
```

Expected: `target_not_enumerable=present`、`trace_ambiguous_in_phase3=absent`、`evidence_chain=present`、`phase3_schema_ok`。

---

## Task 5: 更新第八章输出与自检

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第八章）

- [ ] **Step 1: 更新 8.1 第 2 条**

将：

```text
2. 一个 case 对应一个 `(stub 函数, 最终 impl 函数)` 对；同一 cmdId 命中多个 impl 函数时输出多条 case。
```

替换为：

```text
2. 一个 case 对应一个 `(stub 函数, 最终 impl 函数)` 对；同一 cmdId 命中多个 impl 函数时输出多条 case；目标级扇出的并列结果按此规则并列输出。
```

- [ ] **Step 2: 8.2 自检追加第 11–15 项**

将：

```text
10. mv 后双方法确认最终文件存在（ls + Read）。
```

替换为：

```text
10. mv 后双方法确认最终文件存在（ls + Read）。
11. 每条 `status = "resolved"` 的映射记录至少一个 `final_impl_functions` 元素。
12. 每个最终接口函数定义在 vendor、文件在 `scope_vendor_files.txt` 内，所属类也在 vendor。
13. `final_impl_functions` 已按 `(函数, 文件, 行)` 去重；`target_evidence` 已合并去重。
14. 每个 `status = "done"` 的用例至少一个 `impl_interfaces` 元素，每个元素 `evidence_chain` 非空。
15. 部分失败分支只出现在 unresolved 记录中，不得混入最终 JSON。
```

- [ ] **Step 3: 更新自检失败范围**

将：

```text
- 自检第 1–8 项失败 → `error.json`（`phase=output`，`code=self_check_failed`），停止，不写最终 JSON。
```

替换为：

```text
- 自检第 1–15 项失败 → `error.json`（`phase=output`，`code=self_check_failed`），停止，不写最终 JSON。
```

- [ ] **Step 4: 8.3 完成报告追加统计**

将：

```text
- 已解析映射数（最终 JSON case 总数）；
- unresolved 数量；
```

替换为：

```text
- 已解析映射数（最终 JSON case 总数）；
- 动态扇出分支数：成功到达最终接口的分支数（去重前）/ 记录在 `unresolved_branches` 的失败分支数；
- 最终接口数：`02_impl.json` 中 `final_impl_functions` 元素总数（去重后）；
- unresolved 数量；
```

- [ ] **Step 5: 验证第八章**

Run:

```bash
for t in '目标级扇出的并列结果' '自检第 1–15 项失败' '动态扇出分支数' '最终接口数'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
for b in re.findall(r'```json\n(.*?)```', text, re.S):
    json.loads(b)
print("json_valid=ok")
PY
```

Expected: 无 `missing ...`；`json_valid=ok`。

---

## Task 6: 更新第十章约束、错误码与术语表

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第十章）

- [ ] **Step 1: 替换约束 7**

将：

```text
7. impl 定位顺序：S1/S2/S3 在 vendor + system 统一搜索；S3 绝不在 stub 域搜索；初始 impl 在 system 时按 stub 接口函数做单线路追踪，只选最可能的下一跳，并列记 trace_ambiguous。
```

替换为：

```text
7. impl 定位顺序：S1/S2/S3 在 vendor + system 统一搜索；S3 绝不在 stub 域搜索；调用点选择保持单线路，打分并列按源码顺序取第一个并记录 `tie_break`；目标动态不唯一时全部分支追踪。
```

- [ ] **Step 2: 追加约束 16**

将：

```text
15. 不做自动恢复；中间文件只用于调试与人工查看。
```

替换为：

```text
15. 不做自动恢复；中间文件只用于调试与人工查看。
16. 目标级扇出：函数指针、回调、`dlsym` 符号、多个 virtual override 等动态目标必须在 vendor + system 域内枚举全部候选；每分支独立 visited；分支失败隔离；不设扇出/深度上限。
```

- [ ] **Step 3: 更新术语表**

将：

```text
| 单线路追踪 | system 透传层每层只选择一个最可能的下一跳，直到 vendor 类成员函数 |
```

替换为：

```text
| 调用点选择 | 当前函数体内对多个调用表达式做多信号打分，选出唯一调用表达式；并列按源码顺序取第一个 |
| 目标级扇出 | 调用目标动态不唯一时，全部静态可枚举目标各作为独立追踪分支继续 |
| 追踪分支 | 从某个目标出发的一条独立路径，拥有独立 trace、visited 与终止状态 |
| tie_break | 调用点打分并列时按源码顺序取第一个的记录 |
```

- [ ] **Step 4: 更新部分失败规则**

将：

```text
- 单个映射记录失败：记 unresolved，继续其他映射。
- 单个入口全部映射失败：入口 unresolved，继续其他入口。
```

替换为：

```text
- 单个扇出分支失败：记入 `unresolved_branches`，同映射其他分支继续。
- 单条映射记录全部分支失败：映射 unresolved，继续其他映射。
- 单个入口全部映射失败：入口 unresolved，继续其他入口。
```

- [ ] **Step 5: 验证第十章**

Run:

```bash
for t in '调用点选择' '目标级扇出' '追踪分支' 'tie_break' '目标级扇出：函数指针' '单个扇出分支失败'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
awk '/^## 十、/{f=1} /^## 十一、/{f=0} f' HDF-api-finder/SKILL.md | grep -q '单线路追踪' && echo "old_term_in_chapter10=present" || echo "old_term_in_chapter10=absent"
```

Expected: 无 `missing ...`；`old_term_in_chapter10=absent`。

---

## Task 7: 更新第十一章抗理性化检查

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第十一章）

- [ ] **Step 1: 替换旧行**

将：

```text
| “system 层有多个候选，随便选一个” | 多信号打分；并列记 trace_ambiguous。 |
```

替换为：

```text
| “system 层有多个候选，随便选一个” | 多信号打分；并列按源码顺序取第一个，记录 `tie_break`。 |
```

将：

```text
| “虚函数按静态类型继续” | vendor 内存在唯一 override 时优先该 override。 |
```

替换为：

```text
| “虚函数按静态类型继续” | 收集 vendor + system 域内全部 override，作为目标级扇出分支。 |
```

- [ ] **Step 2: 追加新行**

在抗理性化表末尾（`| “阶段三再解析 impl 方法” | ... |` 之后）追加：

```markdown
| “动态目标有多个实现，判 ambiguous” | 全部扇出，逐分支追踪。 |
| “多个 override 只挑一个” | 全部 override（及可实例化基类实现）作为分支。 |
| “dlsym 带运行时变量就放弃” | 字面量段通配匹配；变量值可静态枚举时精确匹配。 |
| “扇出太多，只追前 N 个” | 不设扇出/深度上限；质量优先。 |
| “分支失败就整体失败” | 单分支失败记 unresolved，其他分支继续。 |
| “所有分支共用一个 visited” | 每分支独立路径 visited，避免 DAG 收敛被误判为环。 |
| “回调注册要展开注册函数内部” | 只认直接赋值点；不展开跨函数指针写入。 |
```

- [ ] **Step 3: 验证第十一章**

Run:

```bash
for t in '记录 `tie_break`' '全部 override' '全部扇出，逐分支追踪' '不设扇出/深度上限' '每分支独立路径 visited'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
```

Expected: 无 `missing ...`。

---

## Task 8: 更新第十二章使用示例

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第十二章）

- [ ] **Step 1: 更新示例一第 3 步**

将：

```text
3. 阶段二：S1 在 system 域定位初始 impl `DemoSystemService`；对 `GetVersion` 等 stub 接口函数做单线路追踪，最终落到 vendor 的 `DemoService::GetVersion`；写 `02_impl.json`（含 `initial_impl_*`、`trace`、`final_impl_*`、`impl_classes`）。
```

替换为：

```text
3. 阶段二：S1 在 system 域定位初始 impl `DemoSystemService`；对 `GetVersion` 等 stub 接口函数先做调用点单线路选择，再对动态目标做目标级扇出追踪，最终落到 vendor 的 `DemoService::GetVersion`；写 `02_impl.json`（含 `initial_impl_*`、`final_impl_functions`、`unresolved_branches`、`impl_classes`）。
```

- [ ] **Step 2: 更新示例二**

将：

```text
初始 impl 在 vendor 且方法定义也在该初始 impl 类中时，单线路追踪不产生额外中间层；`trace = []`，`final_impl_*` 直接取初始 impl 方法，`initial_impl_class = final_impl_class`。若方法定义在 vendor 基类，则 `final_impl_class` 为该 vendor 基类。输出 JSON 的 `impl_classes` 只包含最终 vendor 类。
```

替换为：

```text
初始 impl 在 vendor 且方法定义也在该初始 impl 类中时，不产生额外中间层；`final_impl_functions` 只有一个元素，其 `trace = []`、`target_evidence = []`，`initial_impl_class` 与该元素的 `impl_class` 相同。若方法定义在 vendor 基类，则 `impl_class` 为该 vendor 基类。输出 JSON 的 `impl_classes` 只包含最终 vendor 类。
```

- [ ] **Step 3: 追加示例四、示例五**

在示例三末尾之后追加：

````markdown
### 示例四：动态目标扇出

`DemoSystemService::VendorOpen` 内同时调用 `CheckUpdate()` 与 `demoHalInf.demoHalOpen(...)`；打分选中后者。`demoHalInf.demoHalOpen` 由 `dlsym(handle, "demo_hal_open" + variant)` 赋值，`variant` 运行时决定，按 `demo_hal_open*` 匹配到 `demo_hal_open_a` 与 `demo_hal_open_b` 两个实现。

1. 目标级扇出：两个实现各建一条分支，分别追到 `DemoVendorA::HalOpen` 与 `DemoVendorB::HalOpen`。
2. `02_impl.json` 的该映射记录含两个 `final_impl_functions` 元素，各自带分支 trace 与 `target_evidence`。
3. 阶段三对同一 cmdId 生成两条 case；最终 JSON 并列输出两个最终接口。

### 示例五：调用点并列取首

若 `CheckUpdate()` 与 `demoHalInf.demoHalOpen(...)` 分数相同，按当前函数体内源码顺序取第一个，并在 trace 中记录：

```json
{
  "tie_break": {
    "candidates": [
      {"call": "CheckUpdate()", "score": 5},
      {"call": "demoHalInf.demoHalOpen(...)", "score": 5}
    ],
    "chosen": "CheckUpdate()",
    "reason": "source_order"
  }
}
```

追踪继续，不中止、不判并列歧义。
````

- [ ] **Step 4: 验证第十二章**

Run:

```bash
for t in '示例四：动态目标扇出' '示例五：调用点并列取首' '目标级扇出追踪' 'source_order'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
grep -q 'trace_ambiguous' HDF-api-finder/SKILL.md && echo "trace_ambiguous=present" || echo "trace_ambiguous=absent"
```

Expected: 无 `missing ...`；`trace_ambiguous=absent`。

---

## Task 9: 结构与禁用内容验证

**Files:**
- Verify: `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 章节数量与顺序**

Run:

```bash
grep -c '^## ' HDF-api-finder/SKILL.md
grep -nE '^## (一|二|三|四|五|六|七|八|九|十|十一|十二)、' HDF-api-finder/SKILL.md
```

Expected: 第一行 `12`；第二个命令输出 12 行，顺序为一至十二。

- [ ] **Step 2: frontmatter 与编码**

Run:

```bash
head -5 HDF-api-finder/SKILL.md
file HDF-api-finder/SKILL.md
```

Expected: 前 3 行为 `---`、`name: HDF-api-finder`、`description: ...`；UTF-8 Unicode text。

- [ ] **Step 3: 全部 JSON 可解析并检查新 schema**

Run:

```bash
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
objs = [json.loads(b) for b in blocks]
print(f"json_blocks={len(blocks)}")
final_entries = 0
impl_records = 0
case_items = 0
for obj in objs:
    if not isinstance(obj, dict) or 'entries' not in obj:
        continue
    for e in obj['entries']:
        if not isinstance(e, dict):
            continue
        if 'on_remote_request' in e and 'cases' in e:
            final_entries += 1
            assert 'impl_classes' in e, 'final entry missing impl_classes'
        for rec in e.get('stub_function_map', []):
            impl_records += 1
            assert 'final_impl_functions' in rec, 'mapping missing final_impl_functions'
            assert 'final_impl_function' not in rec, 'old final_impl_function remains'
            assert 'trace' not in rec, 'top-level trace remains'
            for item in rec['final_impl_functions']:
                for key in ('function', 'file', 'line', 'impl_class', 'impl_class_file', 'trace', 'target_evidence'):
                    assert key in item, f'missing {key}'
        for case in e.get('cases', []):
            for item in case.get('impl_interfaces', []):
                case_items += 1
                assert item.get('evidence_chain'), 'missing per-impl evidence_chain'
            assert 'evidence_chain' not in case, 'case-level evidence_chain remains'
print(f"final_entries={final_entries} impl_records={impl_records} case_items={case_items}")
assert final_entries >= 2
assert impl_records >= 1
assert case_items >= 2
print("json_schema_ok")
PY
```

Expected: `json_blocks>=8`、`final_entries>=2`、`impl_records>=1`、`case_items>=2`、`json_schema_ok`。

- [ ] **Step 4: 禁用字符串、旧术语与占位符**

Run:

```bash
grep -nE 'codegraph|hal-service-api-finder|ar_service|ActivityRecognition|ActivityInterfaceStub|ArService|CMD_ACTIVITY_INTERFACE_GET_VERSION|NfcVendorAdaptions|nfcHalInf|phNxpNciHal_open' HDF-api-finder/SKILL.md || echo "forbidden_strings=none"
grep -nE 'TBD|TODO|待定|FIXME|XXX' HDF-api-finder/SKILL.md || echo "placeholders=none"
grep -n 'trace_ambiguous' HDF-api-finder/SKILL.md || echo "trace_ambiguous=absent"
grep -n '单线路追踪' HDF-api-finder/SKILL.md || echo "old_term=absent"
grep -n '"final_impl_function"\|"final_impl_file"\|"final_impl_class"' HDF-api-finder/SKILL.md || echo "old_fields=absent"
```

Expected: `forbidden_strings=none`、`placeholders=none`、`trace_ambiguous=absent`、`old_term=absent`、`old_fields=absent`。

- [ ] **Step 5: 新字段/新 unresolved/错误码覆盖**

Run:

```bash
for t in final_impl_functions unresolved_branches target_evidence tie_break target_not_enumerable '调用点选择' '目标级扇出' '追踪分支'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
for c in usage_error invalid_project_dir conf_not_found invalid_conf output_dir_failed empty_scope entry_invalid entry_not_found impl_not_found final_impl_not_found snapshot_write_failed output_write_failed self_check_failed internal_error; do
  grep -q "$c" HDF-api-finder/SKILL.md || echo "missing $c"
done
```

Expected: 无任何输出。

- [ ] **Step 6: 记录大小**

Run:

```bash
wc -l HDF-api-finder/SKILL.md
wc -c HDF-api-finder/SKILL.md
```

Expected: 记录行数与字节数，供完成报告使用。

---

## Task 10: 流程一致性验证

**Files:**
- Verify: `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 阶段边界**

Run:

```bash
grep -nE '^## (五|六|七|八)、' HDF-api-finder/SKILL.md
grep -q '阶段二做三件事' HDF-api-finder/SKILL.md && echo "phase2_scope=present"
grep -q '阶段三只做一件事' HDF-api-finder/SKILL.md && echo "phase3_scope=present"
grep -q '阶段三不解析 impl 方法定义' HDF-api-finder/SKILL.md && echo "phase3_lookup=present"
```

Expected: 五→六→七→八顺序；3 个 `present`。

- [ ] **Step 2: 单线路与扇出规则**

Run:

```bash
grep -q '调用点选择保持单线路' HDF-api-finder/SKILL.md && echo "call_site_single=present"
grep -q '并列按源码顺序取第一个' HDF-api-finder/SKILL.md && echo "tiebreak=present"
grep -q '目标级扇出' HDF-api-finder/SKILL.md && echo "fanout=present"
grep -q '不设深度上限，不设扇出上限' HDF-api-finder/SKILL.md && echo "no_limit=present"
```

Expected: 4 个 `present`。

- [ ] **Step 3: 目标枚举与分支隔离**

Run:

```bash
grep -q 'vendor + system 域内全部 override' HDF-api-finder/SKILL.md && echo "virtual_fanout=present"
grep -q 'dlsym' HDF-api-finder/SKILL.md && echo "dlsym=present"
grep -q 'target_not_enumerable' HDF-api-finder/SKILL.md && echo "not_enumerable=present"
grep -q '每分支独立' HDF-api-finder/SKILL.md && echo "branch_visited=present"
grep -q 'unresolved_branches' HDF-api-finder/SKILL.md && echo "branch_failure=present"
```

Expected: 5 个 `present`。

- [ ] **Step 4: 输出与状态流**

Run:

```bash
grep -q '"impl_classes"' HDF-api-finder/SKILL.md && echo "impl_classes=present"
grep -q '每个 `status = "done"` 的用例至少一个 `impl_interfaces` 元素' HDF-api-finder/SKILL.md && echo "selfcheck=present"
grep -q '动态扇出分支数' HDF-api-finder/SKILL.md && echo "report=present"
grep -q '不得存在 `pending`' HDF-api-finder/SKILL.md && echo "pending_closed=present"
grep -q 'unresolved 不进入最终 JSON' HDF-api-finder/SKILL.md && echo "unresolved_filter=present"
```

Expected: 5 个 `present`。

- [ ] **Step 5: 发现缺口时修复并重跑**

任一检查输出 `missing` 或缺少 `present` → 回到对应任务补充，再重跑 Task 9 与 Task 10，直到全部通过。

---

## Task 11: 最终通读与交付报告

**Files:**
- Verify: `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 分段通读全文**

Run:

```bash
wc -l HDF-api-finder/SKILL.md
```

Expected: 记录行数。分 3 段 Read 全文（每段不超过 300 行），检查：

- 12 个章节齐全、顺序正确；
- 调用点选择 / 目标级扇出 / 追踪分支 / tie_break 术语一致；
- 阶段二与阶段三边界清晰，阶段三只遍历 `final_impl_functions`；
- `02_impl.json` 映射记录无顶层 `trace`、无 `final_impl_function` 单值字段；
- `03_cases.json` 的 `impl_interfaces` 元素自带 `evidence_chain`，case 级无 `evidence_chain`；
- 示例全部使用中性 demo 命名；
- 无 `trace_ambiguous`、无旧术语 `单线路追踪`、无代码图工具名称、无参考 skill 名称。

- [ ] **Step 2: 最终结构复检**

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
grep -n '单线路追踪' HDF-api-finder/SKILL.md || echo "old_term=absent"
```

Expected: `12`、`json_valid=ok`、`forbidden_strings=none`、`trace_ambiguous=absent`、`old_term=absent`。

- [ ] **Step 3: 交付报告**

报告：

- SKILL.md 绝对路径；
- 行数与字节数；
- 章节数（12）；
- JSON 代码块数量；
- 新字段/新 unresolved 检查结果；
- 明确声明：仅验证流程正确性，未验证真实项目中的接口识别准确性。

---

## Task 12: 提交 SKILL.md

**Files:**
- Modify: `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 检查 diff**

Run:

```bash
git status --short
git diff --stat
```

Expected: 仅 `HDF-api-finder/SKILL.md` 有未提交修改（设计与计划已提交）。

- [ ] **Step 2: 提交 SKILL.md**

Run:

```bash
git add HDF-api-finder/SKILL.md
git -c user.name='siakam43' -c user.email='429872468@qq.com' commit -m "$(cat <<'EOF'
docs(HDF-api-finder): support dynamic target fanout

Keep single-path selection at the call-site layer and fan out dynamically
determined call targets (function pointers, dlsym symbols, multiple virtual
overrides) to all statically enumerable implementations. Record multiple
final impl functions per mapping and emit them as parallel cases.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

Expected: 提交成功，仅包含 `HDF-api-finder/SKILL.md`。

- [ ] **Step 3: 验证提交**

Run:

```bash
git log -1 --format='%h | %an <%ae> | %s'
git status --short
```

Expected: 作者为 `siakam43 <429872468@qq.com>`；工作区干净（除未跟踪文件外）。

- [ ] **Step 4: 不推送**

Run:

```bash
git status -sb
```

Expected: 记录 `main` 领先 `origin/main` 的提交数；本次不执行 `git push`，等待用户授权。

---

## 计划自查

- 规格覆盖：增量设计 §3 对应 Task 2；§4、§5 对应 Task 3；§6 对应 Task 4；§7、§8 对应 Task 5；§9、§10 对应 Task 6；§10.2 对应 Task 7；§11 对应 Task 8；§12 的验证方式对应 Task 9–11；提交对应 Task 12。
- 占位符：无 TBD/TODO；新 JSON schema 与替换文本已完整给出；执行时以增量设计为权威来源。
- 类型/字段一致性：`final_impl_functions`、`unresolved_branches`、`target_evidence`、`tie_break`、`target_not_enumerable`、`impl_interfaces[].evidence_chain` 在 Task 3–10 中一致使用；旧字段 `final_impl_function`、`final_impl_file`、`final_impl_class`、`trace_ambiguous`、`单线路追踪` 均有清理与验证步骤。
