# HDF-api-finder 动态目标扇出设计（增量）

- 日期：2026-11-04
- 状态：设计已确认，待实现
- 目标文件：`HDF-api-finder/SKILL.md`
- 基础设计：`docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md`
- 前置增量：`docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md`
- 本设计为增量设计；与基础设计及前置增量冲突处，以本设计为准。

## 1. 背景与目标

前置增量规定单线路追踪：system 透传层每层只选最可能的下一跳，打分并列记 `trace_ambiguous`，停止该映射。

实践表明该规则过强：**调用点选择**与**调用目标解析**是两个不同层级。

- 调用点层：一个函数体内可能有多个调用表达式（业务调用与日志/校验/辅助调用并存）。继续用多信号打分选唯一，保持单线路。
- 目标层：被选中调用表达式的目标可能由运行时决定（函数指针、回调、`dlsym` 符号、多个虚函数 override）。此时每个静态可枚举的目标都是合理实现，应全部分支追踪，不得判 ambiguous 终止。

示例：中间函数 `VendorOpen` 内同时调用 `CheckFirmwareUpdate()` 与 `halInf.halOpen(...)`；打分选中 `halInf.halOpen`（单线路）。但 `halInf.halOpen` 由 `dlsym(handle, "hal_open" + suffix)` 按运行时变量赋值，scope 内存在多个 `hal_open*` 实现。旧规则在此判并列并中止；新规则应把全部匹配实现作为下一跳分支继续追踪，最终接口并列输出。

目标：在不改变域约束、最终接口约束、输出路径与阶段三基本流程的前提下，扩展阶段二追踪，使一个 `(stub 接口函数, 初始 impl 方法)` 对可解析出多个最终 vendor 接口，并在最终 JSON 中并列输出。

## 2. 术语

| 用词 | 含义 |
|------|------|
| 调用点候选 | 当前函数体内一个可能作为下一跳的调用表达式（成员调用、虚调用、free 调用、函数指针/回调调用） |
| 调用点选择 | 对调用点候选多信号打分，选出唯一调用表达式；并列时按源码顺序取第一个 |
| 目标集合 | 被选中调用表达式运行时可指向的全部函数定义，vendor+system 域内静态可枚举 |
| 目标级扇出 | 目标集合大小大于 1 时，每个目标作为独立下一跳分支继续追踪 |
| 追踪分支 | 从某个目标出发的一条独立路径，拥有独立 trace、visited 与终止状态 |
| tie_break | 调用点打分并列时按源码顺序取第一个的记录 |

## 3. 核心原则变更

- **单线路重新定义**：单线路只约束调用点选择。调用目标动态不唯一时（函数指针、回调、`dlsym` 符号、多个 virtual override），必须全部扇出，不得判 `trace_ambiguous`，不得只挑一个。
- **并列不再中止**：调用点打分并列时，按当前函数体内源码顺序（先文件行号、后列号）取第一个，并在 trace 中记录 `tie_break`（候选、分数、选中项、原因 `source_order`）。`trace_ambiguous` 从 unresolved 类型中移除。
- **分支失败隔离**：单个扇出分支失败只记 unresolved，不中止同映射的其他分支。
- **域规则不变**：目标枚举只在 vendor+system 域进行，stub 域不参与；最终接口仍必须是 vendor 类成员函数，且类完整定义与函数定义都在 vendor。
- **scope 仍是硬边界**：赋值点搜索与符号匹配只使用 scope 清单中的文件，禁止全仓搜索。

## 4. 阶段二变更

基础设计的 6.1–6.3 不变；6.4 仅调整最终接口的写入方式（改为 `final_impl_functions` 数组）。本设计替换前置增量的 3.5（单线路追踪），并同步调整 3.6（02_impl.json）与 3.7（入口状态）。

### 4.1 追踪入口

对每条映射记录的当前中间函数执行以下循环。初始 impl 方法本身是最终接口时，沿用 6.4 第 4 步，不进入追踪。

### 4.2 收集调用点候选

完整读取当前函数定义：

1. 定义不在 scope 或无法唯一确定 → 当前分支 `definition_not_found`，终止。
2. 收集函数体内的候选调用：成员调用、虚函数调用、free 函数调用、可识别为函数指针/回调的调用（目标集合按 §4.3 解析）。
3. 排除日志、断言、纯校验、构造、析构、运算符、框架 sbuf-parcel 转换等非业务调用。

### 4.3 解析每个调用点的目标集合

对每个调用点候选解析其目标集合。目标集合元素为 `(函数限定名, 定义文件, 定义行)`。

**A. 直接调用**（静态可确定的成员调用或 free 调用）：

- 定义唯一 → 目标集合为该定义。
- 重载无法按实参唯一确定 → 该调用点记 `impl_overload_ambiguous`，不参与打分。
- 目标定义不在 scope → 该调用点记 `definition_not_found`，不参与打分。

**B. 虚函数调用**：

- 以静态类型上的方法声明为起点，收集 vendor+system 域内全部 override 作为目标。
- 静态类型上的声明方法非纯虚且 scope 内有函数体时，静态定义本身也计入目标集合。
- 无 override 且无函数体 → 该调用点记 `definition_not_found`，不参与打分。
- 每个目标附带 `virtual_override` 或 `base_implementation` 证据。

**C. 函数指针 / 回调 / 成员函数指针调用**：

1. 识别被调用指针表达式：局部/全局变量、对象成员、结构体成员（`cb`、`obj.cb`、`ptr->cb`）。
2. 在 vendor+system 域搜索赋值点：
   - `<ptr> = &F`、`<ptr> = F`；
   - `<ptr> = &C::M`；
   - `<ptr> = reinterpret_cast<...>(dlsym(...))` 或等价转换形式；
   - 注册点对指针的直接赋值（如 `sCallbacks.onOpen = OnOpen;`）。
3. 赋值点匹配顺序：先按完整指针表达式（含接收者文本）匹配；无命中时按成员/变量名匹配；在 `target_evidence.lookup` 记录 `full_expression` 或 `member_name`。
4. `dlsym` 第二参数按字符串表达式解析：
   - 字面量段固定；
   - 运行时拼接段先尝试在 scope 内枚举常量值（字面量、`const`、`constexpr`、枚举、宏、多点赋值），可枚举则生成精确符号名；
   - 不可枚举的运行时段视为通配；
   - 按**非限定函数名**在 vendor+system 域匹配有函数体的函数定义（free function 或成员函数），全部命中加入目标集合。
   - 字符串表达式完全由运行时段构成、没有可匹配的字面量段，且运行时段不可枚举 → 目标集合为空，记 `target_not_enumerable`。
5. 直接赋值目标与符号匹配目标按 `(函数, 文件, 行)` 去重。
6. 同名重载或同符号多定义全部保留为目标，不按名称猜测唯一。
7. 找不到任何赋值点 → 目标集合为空，该调用点在候选评估中记 `target_not_enumerable`（写入候选证据），不参与打分。
8. 个别赋值点 RHS 无法解析 → 该 RHS 在候选评估中记 `definition_not_found` 或 `target_not_enumerable` 子项，不影响其他 RHS。

**D. 目标集合为空**：该调用点不参与打分；全部调用点的目标集合都为空 → 按 §4.5.5 的空目标集规则终止（`target_not_enumerable` 或 `trace_no_candidate`）。

### 4.4 调用点打分与选择

五信号不变，按调用表达式评分：

1. 实参包含 stub 传入的 data/reply 或派生值；
2. 方法名相同，或去掉命名空间/类限定后是前缀/后缀扩展且多出部分 ≤ 8 字符；
3. 接收者是下一层服务对象，不是日志/校验/框架 helper；
4. 返回值被返回或用于构造 reply；
5. 命名空间/接口类型匹配。

data/reply 派生值只做轻量识别：直接参数、简单别名、`&`/`*`、成员访问、`ReadXxx()` 返回值、`SbufToParcel` 后的 parcel；不做跨函数污点传播。

- 名称与命名空间信号按目标集合中最匹配的目标计算；其余信号按调用表达式本身计算。
- 唯一最高分 → 选中。
- 最高分并列 → 按源码顺序取第一个，记录 `tie_break`。
- 无有效候选 → 当前分支 `trace_no_candidate`，终止。

### 4.5 分支追踪

对选中调用点的目标集合逐目标建立分支：

1. 目标集合大小 1 → 单分支，正常下一跳。
2. 目标集合大小大于 1 → 目标级扇出：每个目标创建一条独立分支，继承当前分支的 trace 与 visited 副本；可递归扇出。
3. 目标排序：按 `(定义文件, 定义行)` 升序，保证结果确定性。
4. 目标分类：
   - **最终接口**：vendor 类成员函数，且类完整定义与函数定义都在 vendor → 记录到本映射的最终接口集合，字段含函数、文件、行、所属类、类文件、分支 trace、`target_evidence`；分支结束。
   - **中间层**：其他情况（system 函数、vendor free function、所属类不在 vendor、vendor 类但函数/类定义不在 vendor）→ 分支 trace 追加该目标与调用证据，继续 4.2。
5. 分支终止原因：`definition_not_found`、`target_not_enumerable`、`trace_no_candidate`、`cycle`。失败分支记入该映射的 `unresolved_branches`，其他分支继续。全部调用点的目标集合都为空时，若存在 `target_not_enumerable` 拒绝记录，当前分支记 `target_not_enumerable`，否则记 `trace_no_candidate`。`final_impl_not_found` 是映射级汇总原因，不作为分支终止原因。
6. visited 每分支独立，键为 `(文件, 函数签名行)`，只含当前分支祖先；同一分支重复命中原函数 → `cycle`。分支间不共享 visited，DAG 收敛不视为环。
7. 不设深度上限，不设扇出上限。

### 4.6 分支结果合并

1. 最终接口按 `(函数, 文件, 行)` 去重；分支按目标解析的深度优先遍历顺序排序；同一最终接口来自多条分支时，保留排序后第一条分支的 trace，`target_evidence` 合并去重。
2. `final_impl_functions` 非空 → 映射记录 `status = "resolved"`。
3. 失败分支全部记入 `unresolved_branches`；部分失败不影响 resolved 结论。
4. 进入 4.5 追踪的映射，全部分支失败或目标集合始终为空 → `status = "unresolved"`，`unresolved_reason` 取排序后第一条失败分支的原因；若没有任何失败分支记录，记为 `final_impl_not_found`；6.2/6.4 已设置的 `unresolved_reason` 保持不变。
5. `tie_break` 记录在发生并列选择的 trace 条目中。
6. 入口 `status` 语义不变：至少一条映射记录 resolved 即为 resolved，否则 unresolved。
7. `unresolved_branches` 按 `(kind, detail)` 去重，`tried` 合并去重。

## 5. 02_impl.json 变更

映射记录保持 `(stub 接口函数, 初始 impl 方法)` 一对一条，把单一 `final_impl_*` 改为数组：

```json
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
          "lookup": "full_expression",
          "pattern": "demo_hal_open*"
        }
      ]
    }
  ],
  "unresolved_branches": [],
  "status": "resolved",
  "unresolved_reason": null
}
```

字段规则：

- `final_impl_functions` 每个元素对应一条最终接口（追踪分支或 6.4 直通映射）；元素自带该分支的 `trace`（含初始 impl 方法，不含最终函数）与 `target_evidence`；初始 impl 方法本身是最终接口时 `trace = []`。
- `target_evidence` 汇总该分支上每次目标级选择产生的证据；直接单目标且无动态选择时可为空数组。
- `target_evidence.match` 取值：`direct_assign` / `dlsym_exact` / `dlsym_glob` / `virtual_override` / `base_implementation`。
- `dlsym_glob` 目标必须记录 `pattern`；`dlsym_exact` 在 `pattern` 记录精确符号名。
- `target_evidence.lookup` 用于赋值点匹配得到的目标（`direct_assign` / `dlsym_exact` / `dlsym_glob`），取值 `full_expression` / `member_name`；`virtual_override` / `base_implementation` 记录声明或定义位置与表达式。
- `tie_break` 为可选字段，仅出现在发生并列选择的 trace 条目；`candidates` 记录参与并列的调用与分数，`chosen` 记录选中项，`reason` 固定为 `source_order`。
- `unresolved_branches` 记录失败分支：`kind` / `detail` / `tried` / 分支 `trace`；按 `(kind, detail)` 去重，`tried` 合并去重，保留排序后第一条分支的 trace。
- `status = "resolved"` 当且仅当 `final_impl_functions` 非空；`unresolved_reason` 仅在 unresolved 时非 null：进入 6.5 的映射取排序后第一条失败分支的原因，若没有任何失败分支记录则记 `final_impl_not_found`；6.2/6.4 已设置的 `unresolved_reason` 保持不变。
- 元素按 `(函数, 文件, 行)` 去重，`target_evidence` 合并去重。
- 入口级 `impl_classes` 语义不变：全部最终 vendor 类按 `(impl_class, impl_file)` 去重排序。
- 写入失败 → `error.json`（`phase=resolve_impl`，`code=snapshot_write_failed`），停止。

## 6. 阶段三变更

### 6.1 查表映射

对每个用例识别出的每个 stub 接口函数：

1. 在 `02_impl.json` 的 `stub_function_map` 中查该 stub 函数对应的全部映射记录。
2. 对每条记录遍历 `final_impl_functions`，每个元素生成一条 `(stub 接口函数, 最终 impl 函数)` 对。
3. `impl_interfaces[]` 每个元素的 `evidence_chain` 为 `[stub 接口函数] + 该元素 trace + [该元素最终函数]`。
4. 映射记录无最终接口 → 把 `unresolved_reason` 写入用例 unresolved。
5. 映射记录有部分失败分支 → 成功分支照常生成对应对；失败分支写入用例 `unresolved`；用例 `status = "done"`。
6. 一个 stub 函数对应多条 resolved 映射记录 → 全部展开，生成多对。

### 6.2 03_cases.json 变更

`impl_interfaces[]` 本就支持多项；每项新增自己的 `evidence_chain`，case 级 `evidence_chain` 移除，避免多条分支共用一条链产生歧义：

```json
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
  }
]
```

- `status = "done"` 的用例至少一个 `impl_interfaces` 元素；`unresolved` 字段可为 null，也可记录部分失败分支。
- `status = "unresolved"` 的用例 `impl_interfaces` 为空，`unresolved` 非空。
- 写穿与防遗忘规则不变。

## 7. 最终 JSON

结构不变：入口含 `impl_classes` 与 `cases`。同一 cmdId 命中多个最终接口时，按现有规则输出多条 case；六元组去重与排序规则不变。最终 JSON 不输出 trace、tie_break、unresolved。

## 8. 自检与完成报告

### 8.1 新增自检项

1. 每条 `status = "resolved"` 的映射记录至少一个 `final_impl_functions` 元素。
2. 每个最终接口函数定义在 vendor、文件在 `scope_vendor_files.txt` 内，所属类也在 vendor。
3. `final_impl_functions` 已按 `(函数, 文件, 行)` 去重；`target_evidence` 已合并去重。
4. 每个 `status = "done"` 的用例至少一个 `impl_interfaces` 元素，每个元素 `evidence_chain` 非空。
5. 部分失败分支只出现在 unresolved 记录中，不得混入最终 JSON。

原自检第 1–10 项继续适用。

### 8.2 完成报告新增

- 动态扇出分支数：成功到达最终接口的分支数（去重前）/ 记录在 `unresolved_branches` 的失败分支数；
- 最终接口数：`02_impl.json` 中 `final_impl_functions` 元素总数（去重后）。

原有报告项不变。

## 9. unresolved 与错误码

### 9.1 unresolved 类型调整

| 变更 | kind | 级别 | 含义 |
|------|------|------|------|
| 移除 | `trace_ambiguous` | 映射 | 调用点并列改为源码顺序取第一个，不再产生此类型 |
| 新增 | `target_not_enumerable` | 映射/分支 | 选中调用点的动态目标无法静态枚举：无赋值点、`dlsym` 表达式完全由运行时段构成且不可枚举、符号模式无字面量段等 |
| 保留 | `impl_overload_ambiguous` | 映射 | 仅指重载无法按实参唯一确定；扇出不覆盖重载歧义 |

其余 kind（`impl_not_found`、`impl_ambiguous`、`dispatch_form_unsupported`、`stub_interface_not_found`、`stub_interface_scope`、`impl_call_not_found`、`impl_method_not_found`、`trace_no_candidate`、`final_impl_not_found`、`cmd_value`、`definition_not_found`、`cycle`）不变。

每条 unresolved 记录 `kind`、`detail`、`tried`；同一 `(kind, detail)` 去重，`tried` 合并去重。

### 9.2 错误码

全局错误码不变。所有入口都没有最终接口时仍按 `final_impl_not_found` 终止。

### 9.3 部分失败

- 单个扇出分支失败：记 `unresolved_branches`，其他分支继续。
- 单条映射记录全部分支失败：映射 unresolved，继续其他映射。
- 单个入口全部映射失败：入口 unresolved，继续其他入口。
- 单个用例失败：记 unresolved，继续其他用例。
- 全局不可恢复错误：写 `error.json`，停止。

## 10. 约束与抗理性化

### 10.1 约束规则调整

- 单线路只约束调用点选择；并列按源码顺序取第一个并记录 `tie_break`。
- 目标动态不唯一时必须全部扇出；不设扇出/深度上限。
- 目标枚举只在 vendor+system 域；stub 域不参与；最终接口必须在 vendor。
- 每分支独立 visited；分支间不共享。
- 分支失败隔离；不静默丢弃。
- 其余约束（scope 硬边界、主 agent、不臆造、绝对路径、双方法存在性检查、中文术语一致、无代码图工具、示例中性命名、从零分析、无自动恢复）不变。

### 10.2 抗理性化检查新增

| 想法 | 现实 |
|------|------|
| “打分并列就 trace_ambiguous 停止” | 源码顺序取第一个，记录 `tie_break` 证据。 |
| “动态目标有多个实现，判 ambiguous” | 全部扇出，逐分支追踪。 |
| “多个 override 只挑一个” | 全部 override（及可实例化基类实现）作为分支。 |
| “dlsym 带运行时变量就放弃” | 字面量段通配匹配；变量值可静态枚举时精确匹配。 |
| “扇出太多，只追前 N 个” | 不设扇出/深度上限；质量优先。 |
| “分支失败就整体失败” | 单分支失败记 unresolved，其他分支继续。 |
| “所有分支共用一个 visited” | 每分支独立路径 visited，避免 DAG 收敛被误判为环。 |
| “回调注册要展开注册函数内部” | 只认直接赋值点；不展开跨函数指针写入。 |

## 11. 示例

中性 demo：system 初始 impl 方法 `DemoSystemService::VendorOpen` 内调用 `demoHalInf.demoHalOpen`；该指针由 `dlsym(handle, "demo_hal_open" + variant)` 赋值；scope 内存在 `demo_hal_open_a` 与 `demo_hal_open_b` 两个实现，分别调用 vendor 类成员 `DemoVendorA::HalOpen` 与 `DemoVendorB::HalOpen`。

1. 调用点候选为 `CheckUpdate()` 与 `demoHalInf.demoHalOpen(...)`；打分选中后者。
2. 目标解析：赋值点命中 `dlsym`，运行时变量 `variant` 不可枚举，按 `demo_hal_open*` 通配匹配两个实现。
3. 目标级扇出：两条分支分别追到 `DemoVendorA::HalOpen` 与 `DemoVendorB::HalOpen`。
4. `02_impl.json` 的该映射记录含两个 `final_impl_functions` 元素。
5. 阶段三对同一 cmdId 生成两条 case；最终 JSON 并列输出。

并列场景示例：若 `CheckUpdate()` 与 `demoHalInf.demoHalOpen(...)` 分数相同，按源码顺序取第一个并在 trace 中记录 `tie_break`，继续追踪不中止。

## 12. 影响范围

- 新建本增量设计文档。
- 在基础设计与前置增量设计顶部追加指向本设计的说明；冲突处以本设计为准。
- 修改 `HDF-api-finder/SKILL.md`：
  - 第一章核心原则：第 5、9、10 条；
  - 第六章阶段二：引言、6.4 第 4–5 步（改为数组写入）、6.5 新算法、6.6 02_impl 结构、6.7 入口状态；
  - 第七章阶段三：7.2 查表映射、7.5 示例、7.6 unresolved 表；
  - 第八章输出与自检：自检项、完成报告；
  - 第十章约束规则、错误码、术语表；
  - 第十一章抗理性化检查；
  - 第十二章新增动态目标扇出示例与并列取首示例。
- 新建实现计划：`docs/superpowers/plans/2026-11-04-hdf-api-finder-dynamic-target-fanout.md`。
- 验证沿用既有约定：只验证结构、术语一致性、流程闭合与 JSON 可解析；不验证真实项目接口识别准确性。

## 13. 不涉及

- 不改变阶段一、conf 字段、scope 生成、搜索纪律、输出路径、原子写出。
- 不做完整污点传播或跨函数指针流分析；回调注册只认直接赋值点。
- 不展开“注册函数内部写指针”的跨函数流程。
- 不分析 `std::function`、lambda、闭包等无法静态落到函数定义的动态目标。
- 不引入 sub-agent、代码图/MCP 工具，不实现自动恢复。
- 不改变“最终接口必须是 vendor 类成员函数”的约束。
- 不验证真实项目中接口识别的语义正确性。
