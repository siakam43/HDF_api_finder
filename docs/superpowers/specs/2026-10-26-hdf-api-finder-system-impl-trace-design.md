# HDF-api-finder system 初始 impl 追踪设计（增量）

- 日期：2026-10-26
- 状态：设计已确认，待实现
- 目标文件：`HDF-api-finder/SKILL.md`
- 基础设计：`docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md`
- 本设计为增量设计；与基础设计冲突处，以本设计为准。

## 1. 背景与目标

基础设计只允许 impl 类定义在 vendor 域，阶段三要求 impl 方法定义在 vendor，否则记 `impl_scope`。

新需求：

- 与 stub 直接绑定的 impl 类（下称**初始 impl 类**）允许定义在 vendor 或 system。
- 若初始 impl 在 system，它只是数据透传层，其接口不是最终接口；必须继续追踪，直到调用到 vendor 域内实现的函数。
- **最终接口所属的类**才是最终 impl 类；最终 impl 类和最终接口函数都必须在 vendor 域。
- system 初始 impl 和 system 中间层只作为追踪链路，不进入最终接口输出。

目标：在不改变阶段一、conf、scope、输出路径和搜索纪律的前提下，扩展阶段二与阶段三，使 system 初始 impl 能被正确下钻到 vendor 最终接口。

## 2. 术语

| 用词 | 含义 |
|------|------|
| 初始 impl 类 | 直接与 stub 绑定的类，允许定义在 vendor 或 system |
| 中间 impl 类 | 初始 impl 在 system 时的透传类；其接口方法不是最终接口 |
| 最终 impl 类 | 最终接口函数所属的 vendor 类，类定义与函数定义都必须在 vendor |
| 最终接口函数 | vendor 类成员函数，从 stub 接口经初始 impl / 中间 impl 最终到达 |
| 单线路追踪 | system 透传层每层只选择一个最可能的下一跳，直到 vendor 类成员函数 |
| 映射记录 | `stub_function_map` 中的一条记录，对应一个 `(stub 接口函数, 初始 impl 方法)` 对 |

## 3. 阶段二变更

### 3.1 分析范围收窄

基础设计 6.1 不变：

- 读取 `OnRemoteRequest` 完整函数体，只收集其中被直接调用的 stub 成员函数。
- 排除 `IPCObjectStub::OnRemoteRequest`、基类框架调用、日志/断言/纯校验、构造/析构/运算符、free 函数、`GetDescriptor`。
- 某个 stub 接口函数在 stub 域内找不到完整定义时，跳过其函数体特征；阶段三/映射记录记 `stub_interface_scope`。
- `OnRemoteRequest` 没有调用任何 stub 成员函数时，S1/S2 无法打分，直接进入 S3 兜底。

### 3.2 stub 函数 → 初始 impl 方法映射

对每个 stub 接口函数：

1. 读取完整函数体，收集对 impl 成员的调用：
   - `impl_->Foo(...)`、`this->impl_->Foo(...)`；
   - 局部别名：`auto p = impl_; p->Foo(...)`；
   - 其他可静态确定的等价形式。
2. 每个不同的 impl 方法调用形成一条映射记录：
   - `stub_interface_function` / `stub_interface_file` / `stub_interface_line`；
   - `initial_impl_function` 记方法名，后续解析到具体类与文件；
   - 一个 stub 函数调用多个 impl 方法时，产生多条映射记录。
3. 没有 impl 调用 → 映射记录 `status = "unresolved"`，`unresolved_reason = impl_call_not_found`。
4. stub 函数定义不在 stub 域 → 映射记录 `status = "unresolved"`，`unresolved_reason = stub_interface_scope`。
5. 映射记录是后续单线路追踪和阶段三用例映射的基本单位。

### 3.3 初始 impl 定位

按 S1 → S2 → S3 顺序，命中唯一结果即停。

#### S1 特征匹配（首选）

在 vendor + system 两域统一找候选初始 impl 类：

- 实现接口类型的类（直接或间接继承接口）；
- 定义与 stub 接口方法集合一一对应的类；
- 继承 stub 类的实现类。

打分维度，从高到低：

1. impl 方法名覆盖率；
2. 接口继承关系；
3. 同命名空间前缀；
4. 类名相似度（与 S2 命名候选的匹配程度，弱信号）。

若 impl 方法名集合为空，退化为用 stub 接口函数名做覆盖度匹配。

唯一最高分候选 → 选定，`initial_impl_domain` 取 `vendor` 或 `system`，`initial_strategy = "feature_match"`。最高分并列 → 进入 S2；仍无法唯一确定 → 进入 S3。

#### S2 命名启发式（辅助排序）

用基础设计 6.2 的命名候选对候选类排序，只用于打破 S1 平局，不单独作为定论。S1 无候选时 S2 无对象可排序，直接进入 S3。S2 选出唯一候选 → `initial_strategy = "naming"`；仍并列 → S3。

#### S3 绑定点 + 注册流程（最后，合并为一档）

仅当 S1/S2 无法唯一确定时执行：

- 在 vendor + system 两域搜索；绝不在 stub 域搜索绑定点。
- 搜索 `GetOrNewObject(<impl>, <interface>::GetDescriptor())`、`new <StubClass>(<impl>)`、`std::make_shared<StubClass>(<impl>)`、`SetImpl(<impl>)`、`impl_ = <impl>`、`host->stub = ...`。
- 对 `<impl>` 表达式回溯：变量声明与初始化、`new XxxService()`、工厂/单例返回、强制转换后的真实对象。
- 若绑定点提供接口描述符，用其校验候选类。
- 搜索 Bind/Init/OnStart/Register/动态注册；visited（键为 `(文件, 函数签名行)`）防环。
- 唯一候选 → `initial_strategy = "binding"` 或 `"registration"`；仍并列 → `impl_ambiguous`；无候选 → `impl_not_found`。

#### 初始 impl 约束

- 初始 impl 类必须是 vendor 或 system 域内的完整类定义，前置声明不算。
- `initial_impl_domain` 只能是 `vendor` 或 `system`。
- 同一 stub 类被多个入口引用时，按 `(stub 类限定名, 定义文件, 定义行)` 缓存复用初始 impl 解析结果。

### 3.4 初始 impl 方法解析

对每条映射记录：

1. 在 `initial_impl_class` 及其 vendor/system 基类中解析 `initial_impl_function` 定义。
2. 方法定义不存在 → 映射记录 unresolved，`impl_method_not_found`。
3. 重载无法按实参唯一确定 → `impl_overload_ambiguous`。
4. 方法定义在 vendor 类成员函数中，且该类的完整定义也在 vendor → **直接作为最终接口**：
   - `final_impl_function` / `final_impl_file` / `final_impl_line` = 该方法；
   - `final_impl_class` / `final_impl_class_file` = 所属 vendor 类；
   - `trace = []`（初始 impl 方法就是最终方法；具体见 3.6 说明）。
5. 其他情况 → 该方法为中间层，进入 3.5 单线路追踪：
   - 定义在 system；
   - vendor free function；
   - 所属类定义不在 vendor；
   - 初始 impl 在 vendor 但方法继承自 system 基类。

### 3.5 单线路追踪

从当前中间函数出发，循环执行：

1. 完整读取当前函数定义；定义不在 scope 内或无法唯一确定 → `definition_not_found`，停止该映射。
2. 收集函数体内的候选调用：
   - 成员调用、虚函数调用、free 函数调用、函数指针/回调中可静态确定的目标；
   - 排除日志/断言/纯校验/构造/析构/运算符/框架 sbuf-parcel 转换等非业务调用。
3. 对每个候选解析目标定义：
   - 若目标是 system 类虚函数，且 vendor 内存在唯一 override → 以该 vendor override 为候选目标；多个 override → 该候选记 `impl_overload_ambiguous`，不参与打分；无 override → 按 system 静态定义。
   - 目标定义不在 scope 或无法唯一确定 → 该候选记 `definition_not_found`，不参与打分。
4. 多信号打分，选择“最可能的实际接口”：
   1. 实参包含 stub 接口传入的 data/reply 或 data/reply 派生值；
   2. 方法名与上一层接口函数相同，或去掉命名空间/类限定后一个是另一个的前缀/后缀扩展且多出部分不超过 8 个字符；
   3. 接收者是下一层服务对象，而不是日志/校验/框架 helper；
   4. 返回值被返回，或用于构造 reply；
   5. 命名空间/接口类型匹配。
5. 唯一最高分 → 下一跳；最高分并列 → `trace_ambiguous`，停止该映射；无候选 → `trace_no_candidate`，停止该映射。
6. 下一跳分类：
   - **最终接口**：vendor 类成员函数，且类完整定义与函数定义都在 vendor → 停止追踪，记录 `final_impl_*` 和 `final_impl_class*`。
   - **中间层**：system 函数、vendor free function、所属类不在 vendor 的成员函数、vendor 类但函数/类定义不在 vendor 的成员函数 → 追加到 `trace`，继续循环。
7. visited 集合键为 `(文件, 函数签名行)`；重复命中 → `cycle`，停止该映射。
8. 不设深度上限；直到最终接口或明确 unresolved（`trace_ambiguous` / `trace_no_candidate` / `definition_not_found` / `cycle`）。
9. 追踪耗尽仍没有最终接口，且未命中上述其他 unresolved 原因 → `final_impl_not_found`。

补充：

- vendor free function 只是中间 helper，不算最终接口；继续追踪到 vendor 类成员函数。
- 最终 impl 类定义与最终接口函数定义都必须在 vendor；只满足其一时继续追踪或记 unresolved。
- 单线路追踪只选择一个最可能的下一跳，不做全调用图扇出；证据不足时记 unresolved，不猜测。

### 3.6 02_impl.json 扩展

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
          "stub_interface_function": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::GetVersion",
          "stub_interface_file": "/abs/stub/demo_interface_stub.cpp",
          "stub_interface_line": 60,
          "initial_impl_function": "OHOS::HDI::Demo::V1_0::DemoSystemService::GetVersion",
          "initial_impl_file": "/abs/system/demo_system_service.cpp",
          "initial_impl_line": 40,
          "trace": [
            {
              "function": "OHOS::HDI::Demo::V1_0::DemoSystemService::GetVersion",
              "file": "/abs/system/demo_system_service.cpp",
              "line": 40,
              "evidence": "DemoService::GetInstance()->GetVersion()"
            }
          ],
          "final_impl_function": "OHOS::HDI::Demo::V1_0::DemoService::GetVersion",
          "final_impl_file": "/abs/vendor/demo_service.cpp",
          "final_impl_line": 80,
          "final_impl_class": "OHOS::HDI::Demo::V1_0::DemoService",
          "final_impl_class_file": "/abs/vendor/demo_service.h",
          "status": "resolved",
          "unresolved_reason": null
        }
      ],
      "impl_classes": [
        {
          "impl_class": "OHOS::HDI::Demo::V1_0::DemoService",
          "impl_file": "/abs/vendor/demo_service.h"
        }
      ],
      "status": "resolved",
      "unresolved_reason": null
    }
  ],
  "unresolved": []
}
```

字段说明：

- `initial_impl_class` / `initial_impl_file` / `initial_impl_domain` / `initial_strategy` / `initial_evidence` 记录初始 impl。
- `stub_function_map` 按 `(stub 接口函数, 初始 impl 方法)` 对记录；一个 stub 函数调用多个 impl 方法时有多条记录。
- `trace` 记录从初始 impl 方法到最终接口之前的完整追踪链（含初始 impl 方法；不含最终接口函数）。初始 impl 方法本身是最终接口时，`trace = []`。
- `final_impl_function` / `final_impl_file` / `final_impl_line` / `final_impl_class` / `final_impl_class_file` 记录最终 vendor 接口。
- `impl_classes` 是该入口全部 resolved 映射记录命中的最终 vendor 类，按 `(impl_class, impl_file)` 去重。
- 映射记录 `status` 取 `resolved` / `unresolved`；`unresolved_reason` 为 unresolved 记录或 null。
- 入口 `status` 为 `resolved` 当且仅当至少一条映射记录 resolved；否则 `unresolved`。
- 写入失败 → `error.json`（`phase=resolve_impl`，`code=snapshot_write_failed`），停止。

### 3.7 入口状态与致命失败

- 单个映射记录失败：记 unresolved，继续其他映射记录。
- 单个入口全部映射失败：入口 `status = "unresolved"`，继续其他入口。
- 所有入口 `status` 均非 `resolved`：
  - 所有入口都没有初始 impl → `error.json`（`phase=resolve_impl`，`code=impl_not_found`）；
  - 至少一个入口有初始 impl，但没有任何入口解析出最终接口 → `error.json`（`phase=resolve_impl`，`code=final_impl_not_found`）。
- 至少一个入口 resolved → 继续阶段三；失败入口不进入最终 JSON，其 unresolved 计入报告。

## 4. 阶段三变更

### 4.1 预规划

基础设计 7.1 不变：枚举全部 cmdId 用例，写入 `03_cases.json`，状态 `pending`。

### 4.2 用例映射

对每个用例：

1. 读取完整分支代码，识别 stub 接口函数调用（基础设计 7.2 的 stub 调用识别规则不变）。
2. 0 个 stub 成员调用 → `stub_interface_not_found`。
3. 多个 stub 调用 → 每个分别形成映射。
4. 对每个 stub 接口函数：
   - 在 `02_impl.json` 的 `stub_function_map` 中查该 stub 函数对应的全部映射记录。
   - 映射记录 resolved → 用 `final_impl_function` / `final_impl_file` / `final_impl_class` / `final_impl_class_file` 生成结果；`evidence_chain` 记录 `[stub 调用] + trace + [final_impl_function]`。
   - 映射记录 unresolved → 把该记录的 `unresolved_reason` 写入用例 unresolved。
   - 一个 stub 函数对应多条 resolved 映射记录 → 生成多条 `(stub, final impl)` 对。
5. 阶段三不再解析 impl 方法定义，不再读取 system 调用链，不再使用 `impl_scope`。

### 4.3 03_cases.json 扩展

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
          "case_expr": "CMD_DEMO_GET_VERSION",
          "cmd_id_macro": "CMD_DEMO_GET_VERSION",
          "cmd_id_value": 0,
          "outer_cmd_macro": null,
          "outer_cmd_value": null,
          "stub_interface_function": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::GetVersion",
          "stub_interface_file": "/abs/stub/demo_interface_stub.cpp",
          "stub_interface_line": 60,
          "impl_interfaces": [
            {
              "function": "OHOS::HDI::Demo::V1_0::DemoService::GetVersion",
              "file": "/abs/vendor/demo_service.cpp",
              "line": 80,
              "impl_class": "OHOS::HDI::Demo::V1_0::DemoService",
              "impl_class_file": "/abs/vendor/demo_service.h",
              "evidence": "DemoService::GetInstance()->GetVersion()"
            }
          ],
          "evidence_chain": [
            "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::GetVersion",
            "OHOS::HDI::Demo::V1_0::DemoSystemService::GetVersion",
            "OHOS::HDI::Demo::V1_0::DemoService::GetVersion"
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

- `impl_interfaces` 只记录最终 vendor 函数；新增 `impl_class` / `impl_class_file`。
- `evidence_chain` 包含 stub 接口函数、初始/中间 impl、最终函数。
- unresolved 用例的 `impl_interfaces` 为空，`unresolved` 记录原因。

### 4.4 写穿

基础设计 7.4 不变：预规划后写 `03_cases.json`，每完成一个用例更新 status 与结果并落盘；全部处理完后不得存在 `pending`。

## 5. 最终 JSON 变更

入口级 `impl_class` / `impl_file` 替换为 `impl_classes` 数组：

```json
{
  "service_name": "demo_service",
  "entries": [
    {
      "on_remote_request": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest",
      "on_remote_request_file": "/abs/stub/demo_interface_stub.cpp",
      "on_remote_request_line": 45,
      "stub_class": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub",
      "impl_classes": [
        {
          "impl_class": "OHOS::HDI::Demo::V1_0::DemoService",
          "impl_file": "/abs/vendor/demo_service.h"
        }
      ],
      "cases": [
        {
          "cmd_id_macro": "CMD_DEMO_GET_VERSION",
          "cmd_id_value": 0,
          "stub_interface_function": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::GetVersion",
          "stub_interface_file": "/abs/stub/demo_interface_stub.cpp",
          "impl_interface_function": "OHOS::HDI::Demo::V1_0::DemoService::GetVersion",
          "impl_interface_file": "/abs/vendor/demo_service.cpp"
        }
      ]
    }
  ]
}
```

规则：

- `impl_classes` 只收录已解析用例命中的最终 vendor 类，按 `(impl_class, impl_file)` 去重，按 `impl_class` → `impl_file` 排序。
- 入口级不再输出 `impl_class` / `impl_file`；case 字段不变。
- 没有可解析用例的入口不进入最终 JSON；全部为空时写 `"entries": []`。
- 自检增加：
  - 每个有 case 的入口 `impl_classes` 非空；
  - 每个 case 的最终 impl 函数所属类（可从 `02_impl.json` 的映射或函数限定名得到）在 `impl_classes` 中；
  - `impl_classes` 无重复、已排序。
- 完成报告增加“初始 impl 类数量 / 最终 impl 类数量”。

## 6. unresolved 与错误码

### 6.1 unresolved 类型调整

保留：

| kind | 级别 | 含义 |
|------|------|------|
| `impl_not_found` | 入口 | 找不到初始 impl 类 |
| `impl_ambiguous` | 入口 | 多个同等置信度初始 impl 候选 |
| `dispatch_form_unsupported` | 入口 | 分发结构无法静态枚举 |
| `stub_interface_not_found` | 用例 | 分支内没有可识别的 stub 成员调用 |
| `stub_interface_scope` | 映射/用例 | stub 接口函数不在 stub 域，或 helper 在 stub 域外 |
| `impl_call_not_found` | 映射 | stub 接口函数内找不到 impl 调用 |
| `impl_method_not_found` | 映射 | 初始 impl 类及 vendor/system 基类中没有对应方法定义 |
| `impl_overload_ambiguous` | 映射 | 方法/B 层调用重载无法按实参唯一确定 |
| `cmd_value` | 用例 | cmdId 值或宏表达式无法解析 |
| `definition_not_found` | 映射/用例 | 函数定义不在 scope 内或无法唯一确定 |
| `cycle` | 映射/用例 | 追踪路径形成真环 |

新增：

| kind | 级别 | 含义 |
|------|------|------|
| `trace_ambiguous` | 映射 | system 层多个候选同等置信度，无法唯一选择 |
| `trace_no_candidate` | 映射 | system 层没有可信的下一跳候选 |
| `final_impl_not_found` | 映射 | 追踪耗尽仍没有 vendor 类成员函数作为最终接口 |

移除：`impl_scope`（system 方法现在是合法中间层）。

说明：`impl_not_found` 与 `final_impl_not_found` 既可作 unresolved kind（映射/入口级），也可作所有入口失败时的致命错误码；按上下文区分。

每条 unresolved 记录 `kind`、`detail`、`tried`；同一 `(kind, detail)` 去重，`tried` 合并去重。

### 6.2 错误码调整

| 错误码 | 触发条件 | 处理 |
|--------|----------|------|
| `impl_not_found` | 所有入口都没有初始 impl 类 | 写 error.json（phase=resolve_impl），结束 |
| `final_impl_not_found` | 至少一个入口有初始 impl，但所有入口都没有解析出 vendor 最终接口 | 写 error.json（phase=resolve_impl），结束 |

其余错误码不变：`usage_error`、`invalid_project_dir`、`conf_not_found`、`invalid_conf`、`output_dir_failed`、`empty_scope`、`entry_invalid`、`entry_not_found`、`snapshot_write_failed`、`output_write_failed`、`self_check_failed`、`internal_error`。

### 6.3 部分失败

- 单个映射记录失败：记 unresolved，继续其他映射。
- 单个入口全部映射失败：入口 unresolved，继续其他入口。
- 单个用例失败：记 unresolved，继续其他用例。
- 单个 cmdId 值解析失败：值记 null，映射仍输出，记 `cmd_value` unresolved。
- 全局不可恢复错误：写 `error.json`，停止。

## 7. 约束与抗理性化

### 7.1 核心原则/约束调整

- 入口只在 stub 域；初始 impl 可在 vendor 或 system；最终 impl 类与最终接口函数必须在 vendor；system 只是中间层。
- S1/S2/S3 在 vendor + system 统一搜索；S3 绝不在 stub 域搜索。
- 单线路、多信号打分；并列记 `trace_ambiguous`，不猜测。
- 最终接口必须是 vendor 类成员函数；vendor free function、system 函数、非 vendor 类成员函数都是中间层。
- 追踪不设深度上限；visited 防环；真环记 `cycle`。
- 只分析 OnRemoteRequest 中实际调用的 stub 成员函数。
- 不臆造：无法确定写 unresolved，记录 detail 与 tried。
- 输出路径绝对路径；函数名带作用域限定；重载以 `(文件, 定义行)` 区分。
- 关键文件存在性双方法确认，禁止 `test -f`。
- 全中文提示词，术语一致。
- 不使用任何代码图/MCP 代码探索工具；不体现与其他 skill 的关联性。
- 示例使用中性 demo 命名。
- 每次运行先备份旧输出，从零分析；不做自动恢复。

### 7.2 抗理性化检查新增

| 想法 | 现实 |
|------|------|
| “system impl 就是最终接口” | 不是；system 只是中间层，必须追到 vendor 类成员函数。 |
| “system 层有多个候选，随便选一个” | 多信号打分；并列记 `trace_ambiguous`。 |
| “追踪太深，先停在这里” | 不设深度上限，直到 vendor 类成员或明确终止条件。 |
| “vendor free function 也算接口” | 不算；继续追到 vendor 类成员函数。 |
| “初始 impl 只能在 vendor” | 初始 impl 可在 vendor/system；最终接口必须在 vendor。 |
| “虚函数按静态类型继续” | vendor 内存在唯一 override 时优先该 override。 |
| “impl 在 system 就记 impl_scope” | `impl_scope` 已移除；system 是合法中间层，继续追踪。 |
| “阶段三再解析 impl 方法” | 阶段二已完成初始 impl → 最终接口映射；阶段三只查表。 |

## 8. 示例

### 8.1 system 初始 impl

```text
/HDF-api-finder /srv/demo/project /srv/demo/conf.json
```

1. 初始化并生成 scope。
2. 阶段一定位 `OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest`。
3. 阶段二 S1 在 system 域定位初始 impl `DemoSystemService`；对 `GetVersion` 等 stub 接口函数做单线路追踪，最终落到 vendor 的 `DemoService::GetVersion`；写 `02_impl.json`。
4. 阶段三枚举 `CMD_DEMO_GET_VERSION` 等用例，从 `stub_function_map` 取 `DemoService::GetVersion`；写 `03_cases.json`。
5. 输出最终 JSON，入口级使用 `impl_classes` 数组。
6. 完成报告增加初始 impl 类数量与最终 impl 类数量。

### 8.2 vendor 初始 impl（兼容场景）

初始 impl 在 vendor 且方法定义也在 vendor 时，单线路追踪不产生额外中间层；`trace = []`，`final_impl_*` 直接取初始 impl 方法，`initial_impl_class = final_impl_class`。

## 9. 影响范围

- 新建本增量设计文档。
- 在基础设计 `2026-10-26-hdf-api-finder-design.md` 顶部加指向本设计的说明；冲突处以本设计为准。
- 修改 `HDF-api-finder/SKILL.md`：
  - 第一章核心原则：域边界、单线路追踪、最终接口规则；
  - 第二章使用方法与输出：最终 JSON 的 `impl_classes`；
  - 第三章域归属判定：区分初始 impl（vendor/system）与最终 impl（vendor）；
  - 第六章阶段二：初始 impl 双域定位、stub 函数映射、单线路追踪、02_impl 扩展、错误处理；
  - 第七章阶段三：查表映射、03_cases 扩展；
  - 第八章输出与自检：`impl_classes`、自检、完成报告；
  - 第十章约束规则、错误码、术语表；
  - 第十一章抗理性化检查；
  - 第十二章使用示例。
- 新建实现计划：`docs/superpowers/plans/2026-10-26-hdf-api-finder-system-impl-trace.md`；旧计划保留为历史。
- 验证仍只覆盖流程正确性，不验证真实项目中接口识别准确性。

## 10. 不涉及

- 不改变阶段一、conf 字段、scope 生成、搜索纪律、输出路径、原子写出。
- 不引入完整污点传播；单线路追踪只使用调用关系与 data/reply 透传信号。
- 不实现自动恢复。
- 不调用 sub-agent。
- 不使用任何代码图/MCP 代码探索工具。
- 不要求自动验证真实项目中的接口识别正确性。
