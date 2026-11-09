# HDF-api-finder vendor 最终自由函数接口设计（增量）

- 日期：2026-11-09
- 状态：已实现，待用户验证
- 目标文件：`HDF-api-finder/SKILL.md`
- 基础设计：`docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md`
- 前置增量：`docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md`、`docs/superpowers/specs/2026-11-04-hdf-api-finder-dynamic-target-fanout-design.md`
- 本设计为增量设计；与基础设计及前置增量冲突处，以本设计为准。

## 1. 背景与目标

现行 6.5 补充规则规定"vendor free function 只是中间 helper，不算最终接口"，并要求最终接口必须是 vendor 类成员函数。实际测试发现，vendor 侧存在没有类承载的 C/C++ 自由函数接口（C 风格 HAL API、命名空间级 API、`extern "C"` 导出函数等）。旧规则下这类函数只会被当作中间层继续追踪：

- 若其后没有 vendor 类成员函数，分支以 `trace_no_candidate` 或 `final_impl_not_found` 终止，真实接口被漏报；
- 即便后续有函数，报告的是更深层的实现，而不是实际对外暴露的自由函数接口。

目标：删除该规则，把定义在 vendor 域的自由函数与 vendor 类成员函数同等视为最终接口；不改变最终 JSON schema、域边界、追踪与扇出规则、阶段三查表流程。

## 2. 术语

| 用词 | 含义 |
|------|------|
| vendor 自由函数 | 定义在 vendor 域文件内、不属于任何类的函数；包含命名空间级函数、`static` 函数、匿名命名空间函数、`extern "C"` 函数 |
| 最终接口函数 | vendor 类成员函数（类完整定义与函数定义都在 vendor）或 vendor 自由函数（函数定义在 vendor） |
| 最终自由函数接口 | 以 vendor 自由函数形式存在的最终接口函数 |
| 最终 impl 类 | 类成员形式的最终接口函数所属的 vendor 类；最终自由函数接口不属于任何类 |
| 中间层 | system 域函数、非 vendor 类成员函数、vendor 类但类定义或函数定义不全在 vendor 的成员函数 |

## 3. 最终接口判定变更

### 3.1 6.5 步骤 6：最终接口

满足任一条件即为最终接口，记录到 `final_impl_functions`（含该分支 `trace` 与 `target_evidence`），分支结束：

1. **类成员最终接口**：vendor 类成员函数，且类定义与函数定义都在 vendor（现状不变）。
2. **最终自由函数接口**：vendor 自由函数，函数定义所在文件属于 vendor 域（新增）。

### 3.2 6.5 步骤 6：中间层

中间层收窄为：

- system 域函数；
- 非 vendor 类成员函数（所属类不在 vendor）；
- vendor 类但类定义或函数定义不全在 vendor 的成员函数。

vendor 自由函数从中间层列表中删除。

### 3.3 判定细节

- 到达任一最终接口即终止当前分支；自由函数与类成员一致，不继续下钻，不做双报。
- "vendor 自由函数"只按定义位置判定：定义文件属于 vendor 域即可，包括 `static`、匿名命名空间、`extern "C"` 函数；不做链接属性、头文件声明或命名启发式过滤。
- 仅声明在 vendor、定义在 system 的自由函数按定义位置归为 system 函数，仍是中间层。
- 同一文件同时属于 vendor 与 system 域时，自由函数按 vendor 归属判定为最终接口（沿用现有域归属规则）。
- 6.4 不变：初始 impl 始终是类；直通路径只产生类成员最终接口，不会直接产生最终自由函数接口。
- 去重、visited、分支隔离、不设深度/扇出上限、目标级扇出规则全部不变；最终自由函数接口按 `(函数限定名, 文件, 行)` 去重，函数名使用命名空间限定的定义处名称。

### 3.4 6.5 补充规则

删除"vendor free function 只是中间 helper，不算最终接口"。

替换为：

- vendor 自由函数与 vendor 类成员函数同等作为最终接口，到达即终止该分支。
- 最终接口函数定义必须在 vendor；若为类成员函数，其类定义也必须在 vendor。

## 4. 阶段二变更

### 4.1 6.4 初始 impl 方法解析

不变。初始 impl 是类，直通路径只处理类成员函数。

### 4.2 6.5 目标级扇出追踪

仅步骤 6 的最终/中间层分类与补充规则变化（见第 3 节）；步骤 1–5、7–10 不变。目标集合中的 vendor 自由函数各自作为独立分支终点，与类成员一样参与分支排序与去重。

### 4.3 6.6 02_impl.json

- `final_impl_functions[]` 元素字段集合不变。最终自由函数接口的 `impl_class` 与 `impl_class_file` 固定为 `null`；`function` 记录命名空间限定的定义处名称，`file`、`line`、`trace`、`target_evidence` 照常。
- `final_impl_functions` 只收录已解析最终接口，`null` 不会与"未解析"混淆。
- 入口级 `impl_classes` 仍只收录类成员最终接口所属的 vendor 类，按 `(impl_class, impl_file)` 去重排序；允许为空（全部最终接口均为自由函数时）。
- 字段规则补充：最终自由函数接口不产生 `impl_classes` 条目。

### 4.4 6.7 入口状态与失败处理

不变。`final_impl_not_found` 的触发条件仍是没有任何最终接口，只是最终接口范围扩大。

## 5. 阶段三与中间结果

### 5.1 7.2 用例映射

不变。阶段三只消费 `final_impl_functions` 的 `function` / `file` / `line`，不区分类成员与自由函数。

### 5.2 7.5 03_cases.json

- `impl_interfaces[]` 元素允许 `impl_class` 与 `impl_class_file` 为 `null`，其余字段照常。
- `evidence_chain` 末尾使用自由函数的命名空间限定名。
- 补充一条字段说明；其余规则不变。

### 5.3 7.6 unresolved 类型

- `final_impl_not_found` 含义改为：追踪耗尽仍没有 vendor 域最终接口函数（类成员函数或自由函数）。
- 其他 unresolved 类型不变。

## 6. 最终 JSON

- Schema 完全不变，不新增任何字段；`不添加示例之外的字段`约束不变。
- 最终自由函数接口只通过 case 的 `impl_interface_function` / `impl_interface_file` 体现。
- 入口级 `impl_classes` 仍只收录类成员最终接口所属的 vendor 类；允许为空。入口只要有可解析 case，即使 `impl_classes` 为空也照常输出。
- 8.1 规则 5 改为：`impl_classes` 只收录类成员最终接口所属的最终 vendor 类，按 `(impl_class, impl_file)` 去重排序；允许为空。
- 8.1 其余组装、去重、排序、字段集合规则不变。

## 7. 自检与完成报告

### 7.1 8.2 自检

- 第 4 项改为：每个 case 的最终接口若为类成员函数，其 `(impl_class, impl_file)` 必须在 `impl_classes` 中；全部最终接口均为自由函数时 `impl_classes` 允许为空，入口仍必须保留。删除"impl_classes 非空"硬要求。
- 第 5 项改为：每条最终 case 的 stub 接口函数定义在 stub 域且文件在 `scope_stub_files.txt` 内；impl 接口函数定义在 vendor 域、文件在 `scope_vendor_files.txt` 内；若 impl 接口函数为类成员函数，所属类也在 vendor 域。
- 第 12 项改为：每个最终接口函数定义在 vendor、文件在 `scope_vendor_files.txt` 内；若为类成员函数，所属类也在 vendor。
- 其余自检项不变。第 6 项"完整定义、定义处作用域限定名"对自由函数同样适用。

### 7.2 完成报告

第二节完成报告与 8.3 的完成报告：

- 增加"最终自由函数接口数"；
- "最终 impl 类数量"明确为"类成员最终接口所属的 vendor 类数量"，自由函数不计入。

## 8. 措辞同步

| 位置 | 改动 |
|------|------|
| 核心原则 3 | 最终接口函数必须定义在 vendor 域；若为类成员函数，其类定义也必须在 vendor 域；system 域函数只作中间层 |
| 核心原则 10 | 最终接口改为 vendor 类成员函数（类定义也在 vendor）或 vendor 自由函数；中间层列表删除 vendor free function |
| 域归属判定 | 同核心原则 3 口径 |
| 6.5 步骤 6 | 最终分支增加 vendor 自由函数；中间层删除 vendor free function |
| 6.5 补充 | 删除旧规则，写入第 3.4 节新规则 |
| 6.6 字段规则 | 增加 `impl_class` / `impl_class_file` 为 null 的语义；`impl_classes` 允许为空 |
| 7.5 | 增加类字段可为 null 的说明 |
| 7.6 | `final_impl_not_found` 含义更新 |
| 8.1 规则 5 | `impl_classes` 允许为空 |
| 8.2 自检 4/5/12 | 按第 7.1 节放宽 |
| 第二节完成报告、8.3 | 增加最终自由函数接口数 |
| 约束规则 4/5 | 最终接口改为定义在 vendor 的类成员函数或自由函数；类成员函数还要求类定义在 vendor |
| 术语表 | 重定义最终 impl 类、impl 接口函数；新增最终接口函数、最终自由函数接口 |
| 抗理性化表 | 反转旧行；更新"system impl"、"直到 vendor 类成员"两句；新增"自由函数没有类，跳过不报"反面项 |
| 使用示例 | 新增示例六（vendor 自由函数接口） |

## 9. 示例

### 9.1 示例六：vendor 自由函数接口

`DemoSystemService::VendorOpen` 调用 vendor 自由函数 `demo::hal_open(...)`；该函数定义在 `/abs/vendor/demo_hal.cpp`，不属于任何类。`demo::hal_open` 即最终接口，分支在该函数终止。

- `02_impl.json` 的 `final_impl_functions` 元素：`function = "demo::hal_open"`，`impl_class = null`，`impl_class_file = null`，`trace` 含 `DemoSystemService::VendorOpen`。
- 最终 JSON 对该 cmdId 输出 case：`impl_interface_function = "demo::hal_open"`，`impl_interface_file = "/abs/vendor/demo_hal.cpp"`。
- 该入口没有类成员最终接口时，`impl_classes` 为空数组，入口仍正常输出。

### 9.2 存量示例

示例一、二、四、五不涉及最终自由函数接口，保持不变。

## 10. 不涉及

- 6.1 与 7.2 中"排除 free 函数"是 stub 侧调用识别规则，与 vendor 最终接口无关，保持原样。
- 6.3 初始 impl 类约束、6.4 直通路径、6.7 入口状态、阶段三查表流程不变。
- 最终 JSON 字段集合不变。
- 域归属判定、目标级扇出、visited、去重、不设深度/扇出上限规则不变。
- 九、权限表不变。
- 历史 spec/plan 不改写。
- 无代码改动、无新增依赖。

## 11. 影响范围

`HDF-api-finder/SKILL.md` 共涉及以下小节：核心原则 3/10、域归属判定、6.5、6.6、7.5、7.6、8.1、8.2、8.3、第二节完成报告、约束规则 4/5、术语表、抗理性化表、使用示例。

## 12. 验证方式

该 skill 是纯 markdown 提示词，仓库无自动化测试。验证分五层：

1. 旧措辞清零：`SKILL.md` 中不再存在旧规则句与"最终接口必须是 vendor 类成员函数"等表述（历史文档在 `docs/` 下，不在验证范围）。
2. 新措辞到位：6.5 最终/中间层、6.6/7.5 字段说明、8.1/8.2、术语表、抗理性化表、示例六均体现自由函数口径。
3. 一致性审计：逐条核对所有"最终接口""impl_classes""所属类"出现处；确认没有误改 6.1/7.2 的 stub 侧 free function 排除规则。
4. Schema 审计：比对最终 JSON 示例字段集合，确认零新增字段；存在"`impl_classes` 允许为空"的表述。
5. 端到端回归（建议）：在触发该问题的真实 HDF 项目上运行 `/HDF-api-finder`，确认自由函数接口进入 `_api.json`、`impl_classes` 为空时自检通过。仓库内无测试 fixture，此步骤需要用户环境配合。
