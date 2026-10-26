---
name: HDF-api-finder
description: 当用户调用 /HDF-api-finder <project_dir> <conf>，或要求在 OpenHarmony/HDF 层 C/C++ 代码中识别指定 service 对外提供的全部接口函数时使用。根据 OnRemoteRequest 入口、cmdId 分发和 stub→impl 调用关系，输出按入口分组的 stub/impl 接口 JSON 映射。
---

# HDF-api-finder

根据 HDF/HAL 层 service 的 IPC stub 实现，识别指定 service 对外提供的全部接口函数，输出 cmdId 到 stub 接口函数、impl 接口函数的映射，用于安全攻击面分析。

## 一、核心原则

1. **分析质量优先。** 宁可慢，不可草率。不设深度上限、不设扇出上限；所有静态可确定的 cmdId 用例全部追踪，环检测是唯一的循环保护。不得因任务长而跳读、合并或提前退出。
2. **scope 是硬边界。** 所有搜索只使用 `<out>/scope_files.txt` 或分类清单；禁止 `grep -rn <project_dir>`、`rg <project_dir>`、`find <project_dir> -exec grep` 等全仓搜索。
3. **域边界。** 入口只在 stub 域搜索；impl 接口必须定义在 vendor 域；system 域函数只作辅助追踪，不能作为 stub 接口或 impl 接口输出。
4. **只分析实际调用的 stub 函数。** 只分析 OnRemoteRequest 中直接调用的 stub 成员函数，不扫描 stub 类的其他成员函数。
5. **证据优先，不臆造。** 每个入口、每个 cmdId 值、每个 stub/impl 对应关系都要有代码证据；无法确定的分析项写入 unresolved，记录 `detail` 与 `tried`，不得猜测。
6. **全部由主 agent 完成。** 不调用 sub-agent，不把分析拆给子代理。
7. **每次从零分析。** 先备份旧输出，再从零分析；不做自动恢复，不复用历史结果。
8. **函数名带作用域。** C++ 函数名使用定义处作用域限定名；重载以 `(文件, 定义行)` 区分，限定名相同不得合并。
9. **impl 定位顺序。** S1 特征匹配 → S2 命名启发式 → S3 绑定点与注册流程；S3 绝不在 stub 域搜索。
10. **中文与独立性。** 全中文提示词，同一概念用词前后一致；正文不得出现任何代码图/MCP 代码探索工具，不体现与其他 skill 的关联性。

## 二、使用方法与输出

### 使用方法

```text
/HDF-api-finder <project_dir> <conf>
```

- 两个参数均必填，不能为空；缺失任意一个、为空或仅含空白字符、或参数多于两个 → `usage_error`，打印用法并结束。
- `<project_dir>` 必须是已存在的目录；相对路径转换为绝对路径后使用。
- `<conf>` 是 JSON 配置文件；相对路径先按当前工作目录解析，找不到再按 `<project_dir>` 解析。
- conf 校验失败时报告具体字段与原因，不创建输出目录。

### conf JSON

```json
{
  "service_name": "demo_service",
  "stub_code": ["/abs/stub"],
  "vendor_code": ["/abs/vendor"],
  "system_code": ["/abs/system"],
  "entry": "/abs/stub/demo_interface_stub.cpp:123"
}
```

- 必填字段：`service_name`、`stub_code`、`vendor_code`。
- 可选字段：`system_code`（默认空数组）、`entry`（缺省、`null`、空串表示未指定）。
- `service_name` 非空，仅允许字母、数字、下划线、点、连字符。
- `stub_code`、`vendor_code`、`system_code` 的每一项都是非空绝对路径；可指向目录或文件，目录递归展开。
- 未知字段 → `invalid_conf`。
- `entry` 必须是 `绝对文件路径:正整数行号`；格式不合法 → `invalid_conf`。文件是否存在、是否属于 stub 域、是否为合法 OnRemoteRequest 定义，由阶段一校验（`entry_invalid`）。

### 输出

最终输出：

```text
<project_dir>/.ethunter_out/HDF-api-finder/<service_name>/<service_name>_api.json
```

最终 JSON 按入口分组，只包含已成功解析 stub→impl 对应关系的用例：

```json
{
  "service_name": "demo_service",
  "entries": [
    {
      "on_remote_request": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest",
      "on_remote_request_file": "/abs/stub/demo_interface_stub.cpp",
      "on_remote_request_line": 45,
      "stub_class": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub",
      "impl_class": "OHOS::HDI::Demo::V1_0::DemoService",
      "impl_file": "/abs/vendor/demo_service.h",
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

- 不添加示例之外的字段；不生成 summary 报告。
- 没有可解析用例的入口不进入最终 JSON；全部入口都没有可解析用例时写 `"entries": []`。

### 完成报告

完成时报告：

- scope 各域文件数与总数；
- service_name；
- 入口来源（conf/discovered）与入口数量；
- stub 类数量、impl 类数量；
- cmdId 用例数（预规划用例数，不含 default）；
- 已解析映射数（最终 JSON 的 case 总数）；
- unresolved 数量；
- 最终输出文件绝对路径。

## 三、初始化与分析范围

### 参数解析

1. 校验两个参数均非空：`<project_dir>` 与 `<conf>` 缺失任意一个、为空或仅含空白字符、或参数多于两个 → `usage_error`，打印用法并结束。
2. 将 `<project_dir>` 解析为绝对路径：相对路径按当前工作目录展开并规范化，后续一律使用绝对路径。
3. `<project_dir>` 不存在或不是目录 → `invalid_project_dir`，结束，不创建任何输出。
4. `<conf>` 不存在 → `conf_not_found`，结束，不创建任何输出。
5. `<conf>` 解析为 JSON；解析失败、必填字段缺失或类型错误、路径非绝对、`service_name` 非法、`entry` 格式错误、存在未知字段 → `invalid_conf`，报告具体字段与原因，结束，不创建输出目录。

### conf 字段

| 字段 | 必填 | 类型 | 规则 |
|------|------|------|------|
| `service_name` | 是 | string | 非空；仅允许字母、数字、下划线、点、连字符 |
| `stub_code` | 是 | string[] | 非空数组；每项为非空绝对路径 |
| `vendor_code` | 是 | string[] | 非空数组；每项为非空绝对路径 |
| `system_code` | 否 | string[] | 默认空数组；每项为非空绝对路径 |
| `entry` | 否 | string | `绝对文件路径:正整数行号`；缺省、`null`、空串表示未指定 |

- 路径项可指向目录或文件：目录递归展开，文件直接加入对应域。
- `entry` 的格式在 conf 阶段校验；文件是否存在、是否属于 stub 域、是否落在合法 OnRemoteRequest 定义内，由阶段一校验（`entry_invalid`）。
- 源码扩展名：`.c .h .cpp .cxx .cc .hpp .hxx .hh .inl .ipp`。

### 输出目录初始化

- 输出根目录：`<out>` = `<project_dir>/.ethunter_out/HDF-api-finder/<service_name>`。
- 若 `<out>` 已存在：先删除旧 `<out>_bak`（若存在），再把 `<out>` 整体 `mv` 为 `<out>_bak`，新建空 `<out>`；不得直接删除旧结果。
- 若 `<out>` 不存在：保留现有 `<out>_bak`，新建 `<out>`。
- 备份或创建失败 → `output_dir_failed`，报告并结束。

### macro.json

- 路径：`<project_dir>/.ethunter_out/macro.json`。
- 按需查阅，不预先全量读取。
- 仅代表编译指令中 `-D` 等动态定义的宏，不是全部宏。
- 遇到 `#ifdef` / `#if`：先查代码，再查 `macro.json`；两处都没有定义，默认分析对应分支。
- `#ifdef` 无法判断时分析受保护分支；`#if/#else` 无法判断时两个分支都分析（宁可多分析，不可漏）。
- `macro.json` 不存在是正常情况，继续分析。

### 分析范围

- 分别展开 `stub_code`、`vendor_code`、`system_code` 中的全部源码文件。
- 路径规范化：统一分隔符为 `/`，转为绝对路径，去重，排序。
- 生成以下文件：

| 文件 | 内容 |
|------|------|
| `scope_files.txt` | 三类源码并集 |
| `scope_stub_files.txt` | stub 域文件清单 |
| `scope_vendor_files.txt` | vendor 域文件清单 |
| `scope_system_files.txt` | system 域文件清单；system_code 为空时不生成 |
| `scope_roots.json` | 三类根路径、文件条目、missing 列表 |

- 不存在的路径记入 `scope_roots.json` 的 `missing`，不中断分析。
- stub 域或 vendor 域展开后为空 → 写 `error.json`（`phase=init`，`code=empty_scope`），停止；system 域为空不构成失败。
- scope 是本次分析的硬边界：不得分析或搜索清单以外的源码文件。

### 域归属判定

- 文件路径以某个域根路径为前缀，或等于该域清单中的文件条目 → 属于该域。
- 允许一个文件同时属于多个域；判定时按具体规则取所需域。
- stub 接口函数必须定义在 stub 域；impl 接口函数必须定义在 vendor 域；system 域函数只作为辅助追踪对象，不能作为 stub 接口或 impl 接口输出。

### 搜索方法与纪律

所有阶段统一使用以下三种方法；后续只写方法名与参数，不再重复命令。其中 `<out>` = `<project_dir>/.ethunter_out/HDF-api-finder/<service_name>`。

```bash
# 1. 并集精确字符串搜索
xargs -r -a "<out>/scope_files.txt" -d '\n' grep -I -H -n -F "<pattern>"

# 2. 并集正则/词边界搜索
xargs -r -a "<out>/scope_files.txt" -d '\n' grep -I -H -n -E "<pattern>"

# 3. 指定域搜索（把 <domain_files> 换成 scope_stub_files.txt / scope_vendor_files.txt / scope_system_files.txt）
xargs -r -a "<out>/<domain_files>" -d '\n' grep -I -H -n -E "<pattern>"

# 4. 指定文件搜索
grep -I -H -n -E "<pattern>" "<file>"
```

全局纪律：

- 只搜索 scope 清单中的文件；禁止 `grep -rn <project_dir>`、`rg <project_dir>`、`find <project_dir> -exec grep` 等全仓搜索。
- `scope_system_files.txt` 不存在时（system_code 为空），不执行 system 域搜索。
- `grep -I` 保留，跳过二进制文件；不得用 `2>/dev/null` 隐藏错误；退出码 1/123 表示无匹配，不是失败。
- 路径变量一律使用双引号；`<project_dir>` 按 shell 规则转义。
- 不使用任何代码图/MCP 代码探索工具；使用 grep、find、Read，可选 ast-grep。

### 文件存在性检查

关键文件/目录的存在性判断一律双方法交叉确认，禁止 `test -f` / `[ -f ]`：

- 方法一：`ls <path>`。
- 方法二：`Read <path>`（目录返回目录类错误也视为存在）。

两次都确认存在 → 存在。两次都确认不存在 → 不存在。两次不一致 → 用 `find` 等第三种方法确认，取多数。两次都无法得出确定结论 → 立即停止并报告，不得假设缺失后继续。

## 四、中间结果记录

中间文件（均位于 `<out>`）：

| 文件 | 写入时机 |
|------|----------|
| `scope_files.txt` / `scope_stub_files.txt` / `scope_vendor_files.txt` / `scope_system_files.txt` / `scope_roots.json` | 初始化阶段 |
| `01_entry.json` | 阶段一结束 |
| `02_impl.json` | 阶段二结束 |
| `03_cases.json` | 预规划后首次写入，之后每完成一个用例更新一次 |
| `error.json` | 任一阶段失败时 |
| `<service_name>_api.json` | 输出阶段，原子写入 |

规则：

- `03_cases.json` 的 `status` 取值为 `pending` / `done` / `unresolved`；全部用例处理完后不得存在 `pending`。
- 长任务防遗忘依靠 `03_cases.json` 的用例清单与 `status` 写穿：每个用例处理前后核对清单，不得跳过 `pending` 项。
- 不做自动恢复机制；每次执行先备份旧输出，再从零分析。
- 中间文件只用于调试与人工查看，不得作为下次运行的输入。
- `error.json` 仅在失败时写入；写盘失败等全局错误按对应阶段处理。

## 五、阶段一：定位 OnRemoteRequest 入口

阶段一只做一件事：确定分析入口并校验，写出 `01_entry.json`。本阶段不分析 impl，不枚举 cmdId。

### 5.1 指定 entry

1. 解析 `<绝对文件路径>:<行号>`；文件必须存在于 stub 域，行号在该文件行数范围内。
2. 读取文件，确认行号落在某个 `OnRemoteRequest` 成员函数定义范围（签名行到配对闭合 `}`）内；归一化到签名行。
3. 校验函数与所属类：
   - 是成员函数定义，不是声明、调用、注释或字符串中的同名文本。
   - 所属类命名空间以 `OHOS::HDI` 开头。
   - 所属类的完整定义必须在 stub 域内；仅前置声明或定义在 stub 域外 → `entry_invalid`。
   - 所属类直接基类列表中包含 `IPCObjectStub`；基类名按最后一级标识符匹配，允许命名空间限定；不得是经 `IRemoteStub<T>` 等中间基类间接继承。
   - 参数恰好四个，按位置类型为：`uint32_t`、`MessageParcel&`、`MessageParcel&`、`MessageOption&`；允许空白差异；返回类型不作限制。
   - 参数名任意；第一个参数代表 cmdId，第二个代表 data，第三个代表 reply，第四个代表 option。
4. 任一条件不满足 → 写 `error.json`（`phase=locate_entry`，`code=entry_invalid`），停止，不写 `01_entry.json`。
5. 只分析该入口，不再自动发现其他入口。

### 5.2 未指定 entry

1. 在 `scope_stub_files.txt` 内搜索 `OnRemoteRequest`（正则/词边界）。
2. 对每个命中读取上下文，只接受完整成员函数定义；调用、声明、注释、字符串中的同名文本一律丢弃并记录原因。
3. 按 5.1 的类与方法规则校验；按 `(文件, 签名行)` 去重。
4. 合法入口数为 0 → 写 `error.json`（`phase=locate_entry`，`code=entry_not_found`），停止。
5. 合法入口数 ≥ 1 → 全部作为分析入口，逐个独立分析。

### 5.3 01_entry.json

```json
{
  "service_name": "demo_service",
  "scope": {
    "stub_file_count": 12,
    "vendor_file_count": 30,
    "system_file_count": 0,
    "total_file_count": 42,
    "scope_files": "scope_files.txt",
    "scope_stub_files": "scope_stub_files.txt",
    "scope_vendor_files": "scope_vendor_files.txt",
    "scope_system_files": null,
    "scope_roots": "scope_roots.json"
  },
  "entry_source": "discovered",
  "entries": [
    {
      "on_remote_request": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest",
      "file": "/abs/stub/demo_interface_stub.cpp",
      "line": 45,
      "stub_class": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub",
      "stub_class_file": "/abs/stub/demo_interface_stub.h",
      "stub_class_line": 20,
      "parent_class": "IPCObjectStub",
      "parameters": [
        {"name": "code", "type": "uint32_t", "role": "cmd_id"},
        {"name": "data", "type": "MessageParcel&", "role": "data"},
        {"name": "reply", "type": "MessageParcel&", "role": "reply"},
        {"name": "option", "type": "MessageOption&", "role": "option"}
      ],
      "evidence": "class DemoInterfaceStub : public IPCObjectStub { ... };"
    }
  ],
  "rejected_candidates": [
    {"file": "/abs/stub/other.cpp", "line": 88, "reason": "声明，不是定义"}
  ]
}
```

- `entry_source` 取 `conf` 或 `discovered`。
- 示例中的参数名仅作示例；实际按位置与类型识别，不要求名字为 `code` / `data`。
- `rejected_candidates` 记录被拒绝的命中及原因，供 `entry_invalid` / `entry_not_found` 的 evidence 使用。
- 写入失败 → 写 `error.json`（`phase=locate_entry`，`code=snapshot_write_failed`），停止。

## 六、阶段二：定位 impl 类

阶段二只做一件事：对每个入口解析 stub 类对应的 impl 类，写出 `02_impl.json`。本阶段不枚举 cmdId 用例。

### 6.1 分析范围收窄

- 读取 `OnRemoteRequest` 完整函数体，只收集其中被直接调用的 stub 成员函数作为 stub 接口函数集合。
- 排除：`IPCObjectStub::OnRemoteRequest`、基类框架调用、日志/断言/纯校验、构造/析构/运算符、free 函数、`GetDescriptor` 等非业务成员。
- 只读取上述 stub 接口函数的函数体，用于提取 impl 成员、接口类型和 impl 方法名；不扫描 stub 类的其他成员函数。
- 若某个 stub 接口函数在 stub 域内找不到完整定义，跳过其函数体特征；阶段三会记 `stub_interface_scope`，不影响其他函数。
- 若 `OnRemoteRequest` 中没有调用任何 stub 成员函数，stub 接口函数集合为空，S1/S2 无法打分，直接进入 S3 兜底。

### 6.2 特征收集

从 stub 类定义与已收窄的 stub 接口函数体中收集：

- impl 成员候选：类型为 `sptr<I...>` / `sptr<...>` 的成员，名称含 `impl` / `service`（不区分大小写）的成员，以及 stub 函数体中作为接收者的成员。
- 接口类型候选：impl 成员声明类型、stub 类命名、`GetDescriptor` 返回值、stub 函数签名中出现的接口类型。
- impl 方法名集合：stub 接口函数体中对 impl 成员的调用，如 `impl_->Foo(...)` 中的 `Foo`。
- 命名候选：stub 类名去掉 `Stub` / `InterfaceStub` 后的基名，尝试 `<Base>`、`<Base>Impl`、`<Base>Service`、`<Base>StubImpl`、`<Base>ImplService`。

### 6.3 impl 定位优先级

按以下顺序执行，命中唯一结果即停。

#### S1 特征匹配（首选）

在 vendor 域内找候选 impl 类，候选来源：

- 实现接口类型的类（直接或间接继承接口）；
- 定义与 stub 接口方法集合一一对应的类（方法名、参数个数、可匹配的签名）；
- 继承 stub 类的实现类（若存在这种形式）。

打分维度，从高到低：

1. impl 方法名覆盖率：stub 接口函数体中调用到的 impl 方法，在候选类中有对应定义的比例；
2. 接口继承关系：候选类是否实现接口类型；
3. 同命名空间前缀：与 stub 类命名空间前缀一致优先；
4. 类名相似度：与 S2 命名候选的匹配程度（弱信号）。

若 impl 方法名集合为空，退化为用 stub 接口函数名（去掉命名空间与类限定后的方法名）做覆盖度匹配。

唯一最高分候选 → 选定，`strategy = "feature_match"`。最高分并列 → 进入 S2；仍无法唯一确定 → 进入 S3。

#### S2 命名启发式（辅助排序）

用 6.2 的命名候选对候选类排序，只用于打破 S1 的平局，不单独作为定论。S1 无候选时 S2 无对象可排序，直接进入 S3。S2 选出唯一候选 → `strategy = "naming"`；仍并列 → 进入 S3。

#### S3 绑定点 + 注册流程（最后，合并为一档）

仅当 S1/S2 无法唯一确定时执行：

- 只在 vendor 域搜索，必要时扩展到 system 域；绝不在 stub 域搜索绑定点。
- 搜索绑定形式：
  - `GetOrNewObject(<impl>, <interface>::GetDescriptor())`；
  - `new <StubClass>(<impl>)` / `std::make_shared<StubClass>(<impl>)`；
  - `SetImpl(<impl>)` / `<stub>->SetImpl(<impl>)`；
  - `impl_ = <impl>` / `<stub>->impl_ = <impl>`；
  - `host->stub = ...`。
- 对 `<impl>` 表达式回溯：变量声明与初始化、`new XxxService()`、工厂/单例返回、强制转换后的真实对象。
- 若绑定点提供接口描述符（如 `IXxx::GetDescriptor()`），用其校验候选类是否实现对应接口。
- 搜索服务注册流程：Bind/Init/OnStart/Register/动态注册函数；沿 stub 与 impl 的创建、绑定关系解析。
- 搜索时必须维护 visited 集合（键为 `(文件, 函数签名行)`），防止循环。
- 得到唯一候选 → `strategy = "binding"` 或 `"registration"`；仍并列 → `impl_ambiguous`；无候选 → `impl_not_found`。

### 6.4 约束与失败处理

- impl 类必须是 vendor 域内的完整类定义（能看到类体），前置声明不算。
- stub 域、system 域中的类不能作为 impl 类。
- 同一 stub 类被多个入口引用时，按 `(stub 类限定名, 定义文件, 定义行)` 缓存复用解析结果。
- 单个入口失败：记入 `02_impl.json` 的 `unresolved`，`status = "unresolved"`，继续处理其他入口。
- 所有入口的 `status` 均非 `resolved`（包括 `impl_not_found` 与 `impl_ambiguous`）→ 写 `error.json`（`phase=resolve_impl`，`code=impl_not_found`），停止，不进入阶段三。
- 至少一个入口成功 → 继续阶段三；失败入口不进入最终 JSON，但其 unresolved 计入报告。

### 6.5 02_impl.json

```json
{
  "service_name": "demo_service",
  "entries": [
    {
      "on_remote_request_file": "/abs/stub/demo_interface_stub.cpp",
      "on_remote_request_line": 45,
      "stub_class": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub",
      "stub_called_functions": [
        {
          "function": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::GetVersion",
          "file": "/abs/stub/demo_interface_stub.cpp",
          "line": 60
        }
      ],
      "interface_type": "OHOS::HDI::Demo::V1_0::IDemoInterface",
      "impl_member": {
        "name": "impl_",
        "type": "sptr<IDemoInterface>",
        "file": "/abs/stub/demo_interface_stub.h",
        "line": 35
      },
      "impl_class": "OHOS::HDI::Demo::V1_0::DemoService",
      "impl_file": "/abs/vendor/demo_service.h",
      "strategy": "feature_match",
      "evidence": [
        {
          "file": "/abs/vendor/demo_service.h",
          "line": 18,
          "expression": "class DemoService : public IDemoInterface {"
        }
      ],
      "candidates": [],
      "status": "resolved",
      "unresolved_reason": null
    }
  ],
  "unresolved": []
}
```

- `strategy` 取值：`feature_match` / `naming` / `binding` / `registration`。
- `impl_file` 记 impl 类定义所在文件。
- `stub_called_functions` 只包含 `OnRemoteRequest` 中实际调用的 stub 成员函数。
- 写入失败 → 写 `error.json`（`phase=resolve_impl`，`code=snapshot_write_failed`），停止。

## 七、阶段三：cmdId 用例与 stub→impl 映射

阶段三只做一件事：枚举 `OnRemoteRequest` 中全部 cmdId 用例，逐个追踪 stub 接口函数到 impl 接口函数，写穿 `03_cases.json`。不进入 impl 方法内部继续展开。

### 7.1 预规划

对每个已解析入口完整读取 `OnRemoteRequest`，枚举全部 cmdId 用例，先写入 `03_cases.json`，状态为 `pending`：

- **switch/case**：每个 `case` 标签一个用例；连续多个标签共享同一段代码时，每个标签分别形成用例；fallthrough 的代码归属每个落入它的标签；`default` 单独记录，不进入最终 JSON。
- **嵌套 switch**：外层 cmd 作为最终 `cmd_id_macro` / `cmd_id_value`，内层条件写入 evidence；没有外层时，内层成为最终 cmd。
- **if/else if/else**：每个能区分 cmdId 的条件分支一个用例；`else` 按 default 处理；复合条件、范围、掩码记录表达式原文。
- **其他形式**（数组/map、函数指针表、宏生成 switch、注册表）：能静态枚举就分析；否则在入口级记 `dispatch_form_unsupported`，继续其他用例。
- **转发**：把 cmdId 转给已知 dispatcher、`SendRequest` 或 `OnRemoteRequest` 时，作为内层分发继续分析；转发位置没有外层 cmd 时，内层 case 成为最终 cmd；已有外层 cmd 时保留外层；维护 visited 防环。

### 7.2 逐用例追踪

对每个用例读取完整分支代码（含 fallthrough 共享代码），识别 stub 接口调用：

- 只接受 stub 类的成员函数调用：`Foo(...)`、`this->Foo(...)`、`StubClass::Foo(...)`，以及其他可静态确定的等价成员调用。
- 排除：`IPCObjectStub::OnRemoteRequest`、基类框架调用、日志/断言/纯校验、构造/析构/运算符、free 函数、`GetDescriptor` 等非业务成员。
- 0 个 stub 成员调用 → `stub_interface_not_found`。
- 多个 stub 成员调用 → 每个分别形成映射。

对每个 stub 接口函数：

1. 定义必须在 stub 域内，且属于 stub 类或 stub 域内的基类；否则记 `stub_interface_scope`。
2. 读取完整函数体，寻找对 impl 成员的调用：
   - `impl_->Foo(...)`、`this->impl_->Foo(...)`；
   - 局部别名：`auto p = impl_; p->Foo(...)`；
   - 其他可静态确定的等价形式。
3. 没有直接调用时，可沿 stub 域内的本地 helper 继续追踪；visited 防环，不设深度上限；找到 impl 调用即停，不进入 impl 方法内部。helper 定义在 stub 域外 → `stub_interface_scope`。
4. 仍无 impl 调用 → `impl_call_not_found`。
5. 对每个 impl 方法名，在 vendor 域解析定义：
   - 优先 `ImplClass::Foo`；
   - 找不到则沿 vendor 域内的基类查找，记录实际定义类与文件；
   - 仅存在于 stub 域、system 域或 scope 外 → `impl_scope`；
   - 仍未找到 → `impl_method_not_found`；
   - 重载无法按实参唯一确定 → `impl_overload_ambiguous`。
6. 任何函数定义无法唯一确定、或不在 scope 内且不属于上述域规则时，记 `definition_not_found`。
7. 一条 `03_cases.json` 用例记录对应一个 stub 接口函数；该函数命中的全部 impl 方法写入 `impl_interfaces` 列表。若一个分支调用多个 stub 接口函数，拆成多条用例记录，共享同一 `case_site` / `cmd_id_macro` / `cmd_id_value`，`id` 递增。

### 7.3 cmdId 取值解析

| 形式 | 处理 |
|------|------|
| 整数字面量 | 直接取值 |
| `#define` 宏链 | 在 scope 内递归解析，支持括号和简单运算符；维护宏展开 visited 防环 |
| 枚举常量 | 按显式值或隐式递增规则解析 |
| `const` / `constexpr` 变量 | 读取初始化值 |
| `macro.json` 中的 `-D` 宏 | 作为辅助来源，不覆盖代码中的确定定义 |
| 算术/移位/位运算 | 常量可计算则求值；除数为 0 视为无法解析 |
| 范围 `A ... B` | 表达式原文，值记 `null` |
| 掩码/组合表达式 | 表达式原文；能算出单一值则记值，否则 `null` |
| 同名宏多定义/条件编译冲突 | 值记 `null`，记 `cmd_value` unresolved 并列出候选 |
| 无法解析 | 值记 `null`，记 `cmd_value` unresolved |

- `cmd_id_value = null` 不阻止该映射进入最终 JSON；映射仍输出，unresolved 计入报告。
- `default` 分支不进入最终 JSON。
- 具名 cmdId 即使值无法解析，也必须保留。

### 7.4 写穿与防遗忘

- 预规划完成后立即写 `03_cases.json`，全部用例 `status = "pending"`。
- `pending` 用例的 stub/impl 字段可以为 `null`；`done` 用例必须有完整的 stub 接口函数与至少一个 impl 接口函数；`unresolved` 用例必须有 unresolved 记录。
- 每完成一个用例，立即更新该用例的 `status`（`done` / `unresolved`）、结果与证据并落盘。
- 每次更新先写 `03_cases.json.tmp`，Read 回读校验通过后 `mv` 覆盖 `03_cases.json`。
- 处理下一个用例前先核对清单，存在 `pending` 则继续处理；不得跳过。
- 全部用例处理完后，`03_cases.json` 中不得存在 `pending`。
- 回读、`mv` 或写盘失败 → 删除可能残留的 `03_cases.json.tmp`，写 `error.json`（`phase=trace_cases`，`code=snapshot_write_failed`），停止。
- 不做自动恢复；每次运行先备份旧输出，从零分析。中间文件只用于调试和人工查看。

### 7.5 03_cases.json

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
              "evidence": "impl_->GetVersion()"
            }
          ],
          "evidence_chain": ["return GetVersion(data, reply);"],
          "status": "done",
          "unresolved": null
        }
      ],
      "default_cases": [
        {
          "case_site": {"file": "/abs/stub/demo_interface_stub.cpp", "line": 70},
          "evidence": "default: return IPCObjectStub::OnRemoteRequest(...)"
        }
      ],
      "unresolved": []
    }
  ],
  "unresolved": []
}
```

### 7.6 unresolved 类型

| kind | 级别 | 含义 |
|------|------|------|
| `impl_not_found` | 入口 | 所有策略都无法定位 impl 类 |
| `impl_ambiguous` | 入口 | 多个同等置信度 impl 候选 |
| `dispatch_form_unsupported` | 入口 | 分发结构无法静态枚举 |
| `stub_interface_not_found` | 用例 | 分支内没有可识别的 stub 成员调用 |
| `stub_interface_scope` | 用例 | stub 接口函数不在 stub 域，或 helper 在 stub 域外 |
| `impl_call_not_found` | 用例 | stub 接口函数内找不到 impl 调用 |
| `impl_method_not_found` | 用例 | impl 类及 vendor 基类中没有对应方法定义 |
| `impl_scope` | 用例 | impl 方法不在 vendor 域 |
| `impl_overload_ambiguous` | 用例 | 重载无法按实参唯一确定 |
| `cmd_value` | 用例 | cmdId 值或宏表达式无法解析 |
| `definition_not_found` | 用例 | 函数定义不在 scope 内或无法唯一确定 |
| `cycle` | 用例 | 追踪路径形成真环 |

每条 unresolved 记录 `kind`、`detail`、`tried`；同一 `(kind, detail)` 去重，`tried` 合并去重。

## 八、输出与自检

本阶段只做两件事：完成自检，然后把 `03_cases.json` 的已解析用例组装为最终 JSON 并写入磁盘。不新增分析、不修改追踪结论。

### 8.1 组装规则

最终输出：`<out>/<service_name>_api.json`。按入口分组，只含已解析项：

```json
{
  "service_name": "demo_service",
  "entries": [
    {
      "on_remote_request": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest",
      "on_remote_request_file": "/abs/stub/demo_interface_stub.cpp",
      "on_remote_request_line": 45,
      "stub_class": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub",
      "impl_class": "OHOS::HDI::Demo::V1_0::DemoService",
      "impl_file": "/abs/vendor/demo_service.h",
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

1. 只从 `03_cases.json` 中 `status = "done"` 的用例组装；unresolved 不进入最终 JSON。
2. 一个 case 对应一个 `(stub 函数, impl 函数)` 对；同一 cmdId 命中多个 impl 函数时输出多条 case。
3. 只保留字段：入口的 `on_remote_request` / `on_remote_request_file` / `on_remote_request_line` / `stub_class` / `impl_class` / `impl_file` / `cases`；case 的 `cmd_id_macro` / `cmd_id_value` / `stub_interface_function` / `stub_interface_file` / `impl_interface_function` / `impl_interface_file`。不添加额外字段。
4. 没有可解析用例的入口不进入最终 JSON；全部入口都没有可解析用例时写 `"entries": []`。
5. 同一入口内按六元组去重：`(cmd_id_macro, cmd_id_value, stub_interface_function, stub_interface_file, impl_interface_function, impl_interface_file)`。
6. 排序：入口按 `on_remote_request_file` → `on_remote_request_line`；case 按 `cmd_id_value` 升序（null 最后）→ `cmd_id_macro` → `stub_interface_function` → `impl_interface_function`。
7. `default` 分支、unresolved 项不输出。
8. UTF-8 编码，JSON 缩进 2 空格。
9. 先写 `<out>/<service_name>_api.json.tmp`，Read 回读校验通过后 `mv` 为最终文件；最终文件必须存在，否则本次任务失败。

### 8.2 自检

以 `03_cases.json` 和最终数组为准，执行以下清单：

1. `01_entry.json` 中的全部入口都已处理。
2. `03_cases.json` 中不存在 `pending` 用例。
3. 每个最终入口至少有一条 case。
4. 每条最终 case 的 stub 接口函数定义在 stub 域且文件在 `scope_stub_files.txt` 内；impl 接口函数定义在 vendor 域且文件在 `scope_vendor_files.txt` 内。
5. 所有 stub/impl 函数都是完整定义，函数名为定义处作用域限定名；重载以 `(文件, 定义行)` 区分。
6. `default` 没有混入最终 JSON；unresolved 没有混入最终 JSON。
7. 最终 JSON 字段集合与 8.1 完全一致；已按六元组去重并排序。
8. 写临时文件后 Read 回读，核对数组长度、字段名、类型；截断视为失败。
9. `mv` 后双方法确认最终文件存在（`ls` + `Read`）。

失败处理：

- 自检第 1–7 项失败 → 写 `error.json`（`phase=output`，`code=self_check_failed`），停止，不写最终 JSON。
- 写临时文件、回读、`mv` 或最终文件存在性确认失败 → 删除可能残留的 `.tmp`，写 `error.json`（`phase=output`，`code=output_write_failed`），停止。

### 8.3 完成报告

写盘成功并确认最终文件存在后报告：

- scope 各域文件数与总数；
- service_name；
- 入口来源（conf/discovered）与入口数量；
- stub 类数量、impl 类数量；
- cmdId 用例数（预规划用例数，不含 default）；
- 已解析映射数（最终 JSON 的 case 总数）；
- unresolved 数量；
- 最终输出文件绝对路径。

服务存在但未识别到可解析接口时，报告 0 条映射，并说明最终 JSON 为 `"entries": []`。

## 九、权限申请

| 工具 | 用途 |
|------|------|
| `Bash` | find/grep/xargs/ast-grep/ls/mkdir/mv/rm；scope 内搜索、备份/清理输出目录 |
| `Read` | 读取源码、conf、macro.json、阶段快照、回读临时 JSON |
| `Write` | 写入 scope 清单、01_entry.json、02_impl.json、03_cases.json、error.json、最终 JSON（含临时文件） |
| `ast-grep`（可选） | AST 模式搜索；未安装时用 grep 代替，不中断 |

权限表只列上述工具；不得出现任何代码图/MCP 代码探索工具。未安装可选工具时用常规工具代替，不中断分析。

## 十、约束规则

1. 全部由主 agent 完成，不调用 sub-agent。
2. 质量优先；不设深度/扇出上限；所有静态可确定的用例全部追踪；环检测是唯一循环保护。
3. scope 是硬边界；所有搜索只用 scope 清单；禁止全仓搜索。
4. 入口只在 stub 域搜索；未指定 entry 时分析全部合法入口；指定 entry 时只分析该入口。
5. stub 接口函数必须在 stub 域；impl 接口函数必须在 vendor 域；system 域函数只作辅助追踪。
6. 只分析 OnRemoteRequest 中实际调用的 stub 成员函数，不扫描 stub 类其他成员。
7. impl 定位顺序：S1 特征匹配 → S2 命名 → S3 绑定点/注册流程；S3 绝不在 stub 域搜索。
8. 不臆造：无法确定写 unresolved，记录 detail 与 tried。
9. 输出路径一律绝对路径；函数名使用定义处作用域限定名；重载以 `(文件, 定义行)` 区分。
10. 关键文件存在性双方法确认，禁止 `test -f` / `[ -f ]`。
11. 全中文提示词，同一概念用词前后一致。
12. skill 正文不得出现任何代码图/MCP 代码探索工具，不得体现与其他 skill 的关联性。
13. skill 正文不使用需求文档原示例值；示例一律使用中性 demo 命名。
14. 每次运行先备份旧输出，再从零分析，不复用历史结果。
15. 不做自动恢复；中间文件只用于调试与人工查看。

### error.json

仅在失败时写入：

```json
{
  "phase": "resolve_impl",
  "code": "impl_not_found",
  "message": "所有入口都无法定位 impl 类",
  "evidence": "已尝试 S1 特征匹配、S2 命名启发式、S3 绑定点与注册流程；候选文件列表见 02_impl.json"
}
```

- `phase` 取值：`init` / `locate_entry` / `resolve_impl` / `trace_cases` / `output`。
- `error.json` 只在新 `<out>` 创建后写入；备份/`mv`/`mkdir` 失败只在对话中报告。
- 若 `error.json` 本身写入失败，改为在对话中报告，不再重试。

### 全局错误码

| 错误码 | 触发条件 | 处理 |
|--------|----------|------|
| `usage_error` | 参数缺失、过多、为空 | 打印用法，结束 |
| `invalid_project_dir` | project_dir 不存在或不是目录 | 报告，结束 |
| `conf_not_found` | conf 文件不存在 | 报告，结束 |
| `invalid_conf` | JSON 解析失败、必填字段缺失/类型错误、路径非绝对、service_name 非法、entry 格式错误、未知字段 | 报告字段与原因，不创建输出 |
| `output_dir_failed` | 备份或创建输出目录失败 | 报告，结束 |
| `empty_scope` | stub 域或 vendor 域展开后为空 | 写 error.json（phase=init），结束 |
| `entry_invalid` | 指定 entry 不是合法 OnRemoteRequest 定义，或文件不在 stub 域 | 写 error.json（phase=locate_entry），结束 |
| `entry_not_found` | 未指定 entry 且 stub 域内无合法入口 | 写 error.json（phase=locate_entry），结束 |
| `impl_not_found` | 所有入口都无法定位 impl 类（含全部入口 impl_ambiguous） | 写 error.json（phase=resolve_impl），结束 |
| `snapshot_write_failed` | 01/02/03 写入失败 | 写 error.json（对应 phase），结束 |
| `output_write_failed` | 最终 JSON 写入、回读、mv 或存在性确认失败 | 写 error.json（phase=output），结束 |
| `self_check_failed` | 自检第 1–7 项失败 | 写 error.json（phase=output），结束 |
| `internal_error` | 工具/文件 I/O 或无法继续读取源码 | 写 error.json（当前 phase），结束 |

### 部分失败

- 单个入口 impl 解析失败：记 unresolved，继续其他入口；只有全部入口失败才 `impl_not_found`。
- 单个用例追踪失败：记 unresolved，继续其他用例；不中止整体分析。
- 单个 cmdId 值解析失败：值记 null，映射仍输出，记 `cmd_value` unresolved。
- 全局不可恢复错误：写 `error.json`，停止。

### 术语表

| 用词 | 含义 |
|------|------|
| stub 域 / vendor 域 / system 域 | conf 指定的三类源码范围 |
| scope | 三个域全部源码文件的并集，所有搜索的硬边界 |
| OnRemoteRequest 入口 | 符合条件的 stub 类成员函数 OnRemoteRequest |
| stub 类 | OnRemoteRequest 所属的类 |
| impl 类 | 在 vendor 域实现实际接口的类 |
| stub 接口函数 | OnRemoteRequest 分支中直接调用的 stub 类成员函数 |
| impl 接口函数 | stub 接口函数最终调用的、定义在 vendor 域的 impl 类或其 vendor 基类成员函数 |
| cmdId 用例 | OnRemoteRequest 中一个能够区分 cmdId 取值的分支 |
| 绑定点 | stub 实例与 impl 实例建立关联的代码位置，通常在 vendor 域的注册流程中 |
| 注册流程 | 创建 stub、创建 impl、绑定二者的服务注册/初始化流程 |
| unresolved | 无法按规则确定的分析项，记录原因与已尝试方法，不臆造 |

## 十一、抗理性化检查

| 想法 | 现实 |
|------|------|
| “参数名不是 cmdId/data，跳过” | 按位置与类型识别，不看参数名。 |
| “stub 类成员太多，全部扫一遍” | 只分析 OnRemoteRequest 中实际调用的 stub 成员函数。 |
| “找不到绑定点就直接报错” | 先做 S1 特征匹配、S2 命名启发式，最后才走 S3 绑定点与注册流程。 |
| “绑定点在 stub 域里找” | 绑定点不在 stub 域；S3 只在 vendor/system 域搜索。 |
| “cmd 值算不出来，丢掉这条” | 保留映射，值记 null，并记 cmd_value unresolved。 |
| “default 也算接口” | default 只记录，不进入最终 JSON。 |
| “impl 在 stub/system 域也算” | impl 接口必须定义在 vendor 域。 |
| “短函数名就够了” | 必须使用定义处完整作用域限定名。 |
| “注册流程太复杂，跳过” | S1/S2 失败后必须执行 S3，不得半途而废。 |
| “范围外 grep 一下” | 禁止；scope 是硬边界。 |
| “用 sub-agent 并行快一点” | 禁止；全部由主 agent 完成。 |
| “阶段结果最后一起写” | 03_cases.json 逐用例写穿，避免长任务遗忘。 |
| “复用上次结果” | 每次备份旧输出后从零分析。 |
| “函数像 helper，跳过” | 读函数体确认，不能凭名字判断。 |

## 十二、使用示例

### 示例一：首次分析

```text
/HDF-api-finder /srv/demo/project /srv/demo/conf.json
```

1. 初始化：校验参数与 conf；若输出目录已存在，先备份为 `<out>_bak`，再新建空 `<out>`；生成 scope 清单。
2. 阶段一：在 stub 域定位 `OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest`，校验命名空间、直接父类和四参数签名；写 `01_entry.json`。
3. 阶段二：用 S1 特征匹配在 vendor 域定位 `OHOS::HDI::Demo::V1_0::DemoService`，记录 impl 成员与证据；写 `02_impl.json`。
4. 阶段三：枚举 `CMD_DEMO_GET_VERSION` 等 cmdId 用例，追踪 `impl_->GetVersion()` 到 `OHOS::HDI::Demo::V1_0::DemoService::GetVersion`；逐用例写穿 `03_cases.json`。
5. 输出：自检后组装 `demo_service_api.json`，写 `.tmp` 并回读校验，再 `mv` 为最终文件。
6. 报告：scope 文件数、cmdId 用例数、已解析映射数、unresolved 数、最终文件绝对路径。

### 示例二：指定 entry

conf 中包含：

```json
{
  "service_name": "demo_service",
  "stub_code": ["/abs/stub"],
  "vendor_code": ["/abs/vendor"],
  "entry": "/abs/stub/demo_interface_stub.cpp:45"
}
```

1. 阶段一只分析该 entry 指向的 `OnRemoteRequest`，不再搜索其他入口。
2. 校验行号落在该函数定义范围内，并归一化到签名行；校验所属类、直接父类和四参数签名。
3. 校验失败 → `entry_invalid`；校验通过后继续阶段二、阶段三。
4. 后续流程与示例一相同。
