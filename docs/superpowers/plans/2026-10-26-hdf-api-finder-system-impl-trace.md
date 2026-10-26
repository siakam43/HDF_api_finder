# HDF-api-finder System 初始 impl 追踪实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修改 `HDF-api-finder/SKILL.md`，允许初始 impl 在 vendor/system，并通过单线路追踪解析 vendor 最终接口；最终 JSON 入口级改为 `impl_classes` 数组。

**Architecture:** 阶段二扩展为“初始 impl 定位 + stub 函数映射 + 单线路追踪 + 最终接口映射”；阶段三改为查表映射；阶段一、conf、scope、搜索纪律、输出路径、原子写出不变。

**Tech Stack:** Markdown、JSON、bash（grep/find/xargs）、Read、Write。

---

## 0. 输入与约束

**增量设计（权威来源）：**

```text
docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md
```

**基础设计（历史基线，冲突处以增量设计为准）：**

```text
docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md
```

**交付/修改文件：**

| 文件 | 责任 |
|------|------|
| `HDF-api-finder/SKILL.md` | 唯一需要修改的交付物 |
| `docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md` | 增量设计；已写入，待提交 |
| `docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md` | 基础设计；顶部已加增量设计指针 |
| `docs/superpowers/plans/2026-10-26-hdf-api-finder-system-impl-trace.md` | 本计划 |

**全局约束：**

- 修改前先读增量设计 §3–§8；所有新规则以该文档为准。
- 示例统一使用中性 demo 命名；不使用需求原始示例。
- SKILL.md 正文不得出现任何代码图/MCP 代码探索工具名称，不得体现与其他 skill 的关联性。
- 保留阶段一、conf、scope、搜索纪律、输出路径、原子写出、无自动恢复、无 sub-agent。
- 当前仓库已初始化，远端 `origin` 指向 `siakam43/HDF_api_finder`，默认分支 `main`。
- 提交使用 `git -c user.name='siakam43' -c user.email='429872468@qq.com' commit`，不修改 git config。
- 最后推送 `git push origin main`；用户已完成授权。

---

## Task 1: 提交增量设计与计划

**Files:**
- Add: `docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md`
- Modify/Add: `docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md`（顶部指针）
- Add: `docs/superpowers/plans/2026-10-26-hdf-api-finder-system-impl-trace.md`

- [ ] **Step 1: 确认工作区状态**

Run:

```bash
git status --short
```

Expected: 看到增量设计、基础设计修改、本计划为未跟踪/已修改状态；`HDF-api-finder/SKILL.md` 尚未修改。

- [ ] **Step 2: 暂存并提交设计+计划**

Run:

```bash
git add docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md \
        docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md \
        docs/superpowers/plans/2026-10-26-hdf-api-finder-system-impl-trace.md
git -c user.name='siakam43' -c user.email='429872468@qq.com' commit -m "$(cat <<'EOF'
docs(HDF-api-finder): add system impl trace design and plan

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

Expected: 提交成功，包含 3 个文件。

- [ ] **Step 3: 验证提交**

Run:

```bash
git log -1 --format='%h | %an <%ae> | %s'
git status --short
```

Expected: 作者为 `siakam43 <429872468@qq.com>`；除 `HDF-api-finder/SKILL.md` 外无未提交设计/计划改动。

---

## Task 2: 更新第一章核心原则与第三章域归属

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第一章、第三章）

- [ ] **Step 1: 检查旧文本存在**

Run:

```bash
grep -n 'impl 接口必须定义在 vendor 域' HDF-api-finder/SKILL.md
grep -n 'S3 绝不在 stub 域搜索' HDF-api-finder/SKILL.md
```

Expected: 两处都有命中（第一章原则 3/9、第三章域归属）。

- [ ] **Step 2: 替换第一章原则 3**

将：

```text
3. **域边界。** 入口只在 stub 域搜索；impl 接口必须定义在 vendor 域；system 域函数只作辅助追踪，不能作为 stub 接口或 impl 接口输出。
```

替换为：

```text
3. **域边界。** 入口只在 stub 域搜索；初始 impl 类可在 vendor 或 system；最终 impl 类与最终接口函数必须在 vendor 域；system 域函数只作辅助追踪/中间层，不能作为最终接口输出。
```

- [ ] **Step 3: 替换第一章原则 9 与 10**

将：

```text
9. **impl 定位顺序。** S1 特征匹配 → S2 命名启发式 → S3 绑定点与注册流程；S3 绝不在 stub 域搜索。
10. **中文与独立性。** 全中文提示词，同一概念用词前后一致；正文不得出现任何代码图/MCP 代码探索工具，不体现与其他 skill 的关联性。
```

替换为：

```text
9. **impl 定位顺序。** S1 特征匹配 → S2 命名启发式 → S3 绑定点与注册流程；S1/S2/S3 在 vendor + system 两域统一搜索，S3 绝不在 stub 域搜索；初始 impl 在 system 时，按 stub 接口函数做单线路追踪。
10. **单线路追踪与最终接口。** system 透传层每层用多信号打分，只选最可能的下一跳；并列记 `trace_ambiguous`，不猜测；最终接口必须是 vendor 类成员函数，且类定义与函数定义都在 vendor；vendor free function、system 函数、非 vendor 类成员只是中间层。
11. **中文与独立性。** 全中文提示词，同一概念用词前后一致；正文不得出现任何代码图/MCP 代码探索工具，不体现与其他 skill 的关联性。
```

- [ ] **Step 4: 替换第三章域归属规则**

将：

```text
- stub 接口函数必须定义在 stub 域；impl 接口函数必须定义在 vendor 域；system 域函数只作为辅助追踪对象，不能作为 stub 接口或 impl 接口输出。
```

替换为：

```text
- stub 接口函数必须定义在 stub 域；初始 impl 类可在 vendor 或 system；最终 impl 类与最终接口函数必须定义在 vendor 域；system 域函数只作为辅助追踪/中间层，不能作为最终接口输出。
```

- [ ] **Step 5: 验证**

Run:

```bash
grep -q '初始 impl 类可在 vendor 或 system' HDF-api-finder/SKILL.md && echo "initial_impl_domain=present"
grep -q '单线路追踪与最终接口' HDF-api-finder/SKILL.md && echo "trace_principle=present"
grep -q '最终接口必须是 vendor 类成员函数' HDF-api-finder/SKILL.md && echo "final_rule=present"
```

Expected: 3 个 `present`。

---

## Task 3: 更新第二章最终 JSON 示例与完成报告

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第二章）

- [ ] **Step 1: 定位旧字段**

Run:

```bash
grep -n '"impl_class"\|"impl_file"' HDF-api-finder/SKILL.md | head -20
```

Expected: 第二章最终 JSON 示例中出现入口级 `impl_class` / `impl_file`。

- [ ] **Step 2: 替换第二章最终 JSON 示例**

将第二章“输出”小节中的最终 JSON 示例整体替换为：

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

- [ ] **Step 3: 更新输出说明与完成报告字段**

将：

```text
最终 JSON 按入口分组，只包含已成功解析 stub→impl 对应关系的用例：
```

替换为：

```text
最终 JSON 按入口分组，只包含已成功解析最终 vendor 接口的用例：
```

将：

```text
- stub 类数量、impl 类数量；
```

替换为：

```text
- stub 类数量、初始 impl 类数量、最终 impl 类数量；
```

- [ ] **Step 4: 验证第二章**

Run:

```bash
grep -q '"impl_classes"' HDF-api-finder/SKILL.md && echo "impl_classes=present"
grep -q '初始 impl 类数量、最终 impl 类数量' HDF-api-finder/SKILL.md && echo "report=present"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
print(f"json_blocks={len(blocks)}")
for b in blocks:
    json.loads(b)
print("json_valid=ok")
PY
```

Expected: `impl_classes=present`、`report=present`、`json_valid=ok`。

---

## Task 4: 重写第六章阶段二

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第六章整章）

- [ ] **Step 1: 确认第六章边界**

Run:

```bash
grep -n '^## 六、\|^## 七、' HDF-api-finder/SKILL.md
```

Expected: 得到第六章起始行与第七章起始行。

- [ ] **Step 2: 用增量设计 §3 重写第六章**

将 `## 六、阶段二：定位 impl 类` 到 `## 七、阶段三：cmdId 用例与 stub→impl 映射` 之前的内容整体替换为以下结构（完整措辞按增量设计 §3；不得删减规则）：

```markdown
## 六、阶段二：定位 impl 类

阶段二做三件事：定位初始 impl 类、把每个 stub 接口函数映射到初始 impl 方法、必要时做单线路追踪到 vendor 最终接口，最后写出 `02_impl.json`。本阶段不枚举 cmdId 用例。

### 6.1 分析范围收窄

- 读取 `OnRemoteRequest` 完整函数体，只收集其中被直接调用的 stub 成员函数作为 stub 接口函数集合。
- 排除：`IPCObjectStub::OnRemoteRequest`、基类框架调用、日志/断言/纯校验、构造/析构/运算符、free 函数、`GetDescriptor` 等非业务成员。
- 只读取上述 stub 接口函数的函数体；不扫描 stub 类的其他成员函数。
- 某个 stub 接口函数在 stub 域内找不到完整定义 → 映射记录记 `stub_interface_scope`。
- `OnRemoteRequest` 没有调用任何 stub 成员函数 → S1/S2 无法打分，直接进入 S3。

### 6.2 stub 函数 → 初始 impl 方法映射

对每个 stub 接口函数：

1. 读取完整函数体，收集对 impl 成员的调用：`impl_->Foo(...)`、`this->impl_->Foo(...)`、`auto p = impl_; p->Foo(...)`、其他可静态确定的等价形式。
2. 每个不同的 impl 方法调用形成一条映射记录：
   - `stub_interface_function` / `stub_interface_file` / `stub_interface_line`；
   - `initial_impl_function` 先记方法名，后续解析到具体类与文件。
3. 一个 stub 函数调用多个 impl 方法 → 多条映射记录。
4. 没有 impl 调用 → 映射记录 `status = "unresolved"`，`unresolved_reason = impl_call_not_found`。
5. stub 函数定义不在 stub 域 → 映射记录 `status = "unresolved"`，`unresolved_reason = stub_interface_scope`。

### 6.3 初始 impl 定位

先收集特征：

- impl 成员候选：类型为 `sptr<I...>` / `sptr<...>` 的成员，名称含 `impl` / `service`（不区分大小写）的成员，以及 stub 函数体中作为接收者的成员。
- 接口类型候选：impl 成员声明类型、stub 类命名、`GetDescriptor` 返回值、stub 函数签名中出现的接口类型。
- impl 方法名集合：stub 接口函数体中对 impl 成员的调用，如 `impl_->Foo(...)` 中的 `Foo`。
- 命名候选：stub 类名去掉 `Stub` / `InterfaceStub` 后的基名，尝试 `<Base>`、`<Base>Impl`、`<Base>Service`、`<Base>StubImpl`、`<Base>ImplService`。

按 S1 → S2 → S3 顺序，命中唯一结果即停。

#### S1 特征匹配（首选）

在 vendor + system 两域统一找候选初始 impl 类：

- 实现接口类型的类（直接或间接继承接口）；
- 定义与 stub 接口方法集合一一对应的类；
- 继承 stub 类的实现类。

打分从高到低：impl 方法名覆盖率 → 接口继承关系 → 同命名空间前缀 → 类名相似度（弱信号）。

impl 方法名集合为空时，退化为用 stub 接口函数名做覆盖度匹配。

唯一最高分 → 选定，`initial_impl_domain` 取 `vendor` 或 `system`，`initial_strategy = "feature_match"`；最高分并列 → S2；仍不确定 → S3。

#### S2 命名启发式（辅助排序）

用本节命名候选排序，只用于打破 S1 平局，不单独作为定论。S1 无候选时直接进入 S3；唯一 → `initial_strategy = "naming"`；仍并列 → S3。

#### S3 绑定点 + 注册流程（最后，合并为一档）

仅当 S1/S2 无法唯一确定时执行：

- 在 vendor + system 搜索；绝不在 stub 域搜索绑定点。
- 搜索 `GetOrNewObject(<impl>, <interface>::GetDescriptor())`、`new <StubClass>(<impl>)`、`std::make_shared<StubClass>(<impl>)`、`SetImpl(<impl>)`、`impl_ = <impl>`、`host->stub = ...`。
- 对 `<impl>` 表达式回溯：变量声明与初始化、`new XxxService()`、工厂/单例返回、强制转换后的真实对象。
- 若绑定点提供接口描述符，用其校验候选类。
- 搜索 Bind/Init/OnStart/Register/动态注册；visited（键为 `(文件, 函数签名行)`）防环。
- 唯一候选 → `initial_strategy = "binding"` 或 `"registration"`；仍并列 → `impl_ambiguous`；无候选 → `impl_not_found`。

约束：

- 初始 impl 类必须是 vendor 或 system 域内的完整类定义，前置声明不算。
- 同一 stub 类被多个入口引用时，按 `(stub 类限定名, 定义文件, 定义行)` 缓存复用。

### 6.4 初始 impl 方法解析

对每条映射记录：

1. 在 `initial_impl_class` 及其 vendor/system 基类中解析 `initial_impl_function` 定义。
2. 方法定义不存在 → `impl_method_not_found`。
3. 重载无法按实参唯一确定 → `impl_overload_ambiguous`。
4. 方法定义在 vendor 类成员函数中，且该类的完整定义也在 vendor → 直接作为最终接口：记录 `final_impl_function/file/line`、`final_impl_class/class_file`，`trace = []`。
5. 其他情况 → 进入 6.5 单线路追踪。

### 6.5 单线路追踪

从当前中间函数出发循环：

1. 完整读取当前函数定义；定义不在 scope 或无法唯一确定 → `definition_not_found`，停止该映射。
2. 收集候选调用：成员调用、虚函数调用、free 函数调用、可静态确定的函数指针/回调；排除日志/断言/纯校验/构造/析构/运算符/框架 sbuf-parcel 转换。
3. 解析每个候选目标：
   - system 类虚函数 + vendor 唯一 override → 以 vendor override 为候选目标；多个 override → 该候选 `impl_overload_ambiguous`，不参与打分；无 override → 按 system 静态定义。
   - 目标定义不在 scope 或无法唯一确定 → `definition_not_found`，不参与打分。
4. 多信号打分：
   1. 实参包含 stub 传入的 data/reply 或派生值；
   2. 方法名相同，或去掉命名空间/类限定后是前缀/后缀扩展且多出部分 ≤ 8 字符；
   3. 接收者是下一层服务对象，不是日志/校验/框架 helper；
   4. 返回值被返回或用于构造 reply；
   5. 命名空间/接口类型匹配。
5. 唯一最高分 → 下一跳；并列 → `trace_ambiguous`，停止；无候选 → `trace_no_candidate`，停止。
6. 下一跳分类：
   - 最终接口：vendor 类成员函数，且类定义与函数定义都在 vendor → 停止，记录 `final_impl_*`、`final_impl_class*`。
   - 中间层：system 函数、vendor free function、所属类不在 vendor 的成员函数、vendor 类但函数/类定义不在 vendor 的成员函数 → 追加 trace，继续。
7. visited 键 `(文件, 函数签名行)`；重复 → `cycle`。
8. 不设深度上限；耗尽且未命中其他 unresolved 原因 → `final_impl_not_found`。

补充：

- vendor free function 只是中间 helper，不算最终接口。
- 最终 impl 类定义与最终接口函数定义都必须在 vendor。
- 单线路只选一个最可能的下一跳，不做全调用图扇出；证据不足记 unresolved，不猜测。

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

字段与规则按增量设计 §3.6；`stub_function_map` 按 `(stub 接口函数, 初始 impl 方法)` 对记录；`trace` 含初始 impl 方法、不含最终函数；`impl_classes` 去重。写入失败 → `snapshot_write_failed`。

### 6.7 入口状态与失败处理

- 单映射失败 → unresolved，继续其他映射。
- 单入口全部映射失败 → entry unresolved，继续其他入口。
- 所有入口 unresolved：
  - 全部没有初始 impl → `error.json`（`phase=resolve_impl`，`code=impl_not_found`）；
  - 至少一个入口有初始 impl，但无任何最终接口 → `error.json`（`phase=resolve_impl`，`code=final_impl_not_found`）。
- 至少一个 entry resolved → 继续阶段三。
```

- [ ] **Step 3: 验证第六章**

Run:

```bash
for t in initial_impl_domain stub_function_map trace_ambiguous trace_no_candidate final_impl_not_found '6.5 单线路追踪'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
grep -q 'S1 特征匹配' HDF-api-finder/SKILL.md && echo "s1=present"
grep -q 'S3 绑定点' HDF-api-finder/SKILL.md && echo "s3=present"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
print(f"json_blocks={len(blocks)}")
for b in blocks:
    json.loads(b)
print("json_valid=ok")
PY
```

Expected: 无 `missing ...`；`s1=present`、`s3=present`、`json_valid=ok`。

---

## Task 5: 重写第七章阶段三

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第七章整章）

- [ ] **Step 1: 确认第七章边界**

Run:

```bash
grep -n '^## 七、\|^## 八、' HDF-api-finder/SKILL.md
```

Expected: 得到第七章起始行与第八章起始行。

- [ ] **Step 2: 用增量设计 §4 重写第七章**

将第七章整体替换为：

```markdown
## 七、阶段三：cmdId 用例与 stub→impl 映射

阶段三只做一件事：枚举 cmdId 用例，识别 stub 接口函数调用，从 `02_impl.json` 的 `stub_function_map` 取最终 vendor 接口，写穿 `03_cases.json`。不解析 impl 方法定义，不读取 system 调用链。

### 7.1 预规划

对每个已解析入口完整读取 `OnRemoteRequest`，枚举全部 cmdId 用例，先写入 `03_cases.json`，状态为 `pending`：

- switch/case：每个 `case` 标签一个用例；连续标签共享代码分别记录；fallthrough 归属所有落入标签；`default` 单独记录、不进最终 JSON。
- 嵌套 switch：外层 cmd 作为最终 cmd，内层条件写 evidence；无外层时内层成为最终 cmd。
- if/else if/else：每个能区分 cmdId 的条件分支一个用例；`else` 按 default 处理；复合条件、范围、掩码记录表达式原文。
- 其他形式（数组/map、函数指针表、宏生成 switch、注册表）：能静态枚举就分析；否则入口级记 `dispatch_form_unsupported`，继续其他用例。
- 转发：转给已知 dispatcher、`SendRequest` 或 `OnRemoteRequest` 时作为内层分发继续；无外层 cmd 时内层 case 成为最终 cmd，已有外层则保留；visited 防环。

### 7.2 用例映射

对每个用例读取完整分支代码（含 fallthrough 共享代码），识别 stub 接口调用：

- 只接受 stub 类成员函数调用：`Foo(...)`、`this->Foo(...)`、`StubClass::Foo(...)` 及可静态确定的等价成员调用。
- 排除 `IPCObjectStub::OnRemoteRequest`、基类框架调用、日志/断言/纯校验、构造/析构/运算符、free 函数、`GetDescriptor`。
- 0 个 stub 成员调用 → `stub_interface_not_found`；多个 → 每个分别形成映射。

对每个 stub 接口函数：

1. 在 `02_impl.json` 的 `stub_function_map` 中查该 stub 函数对应的全部映射记录。
2. 映射记录 resolved → 用 `final_impl_function` / `final_impl_file` / `final_impl_class` / `final_impl_class_file` 生成结果；`evidence_chain` 记录 `[stub 调用] + trace + [final_impl_function]`。
3. 映射记录 unresolved → 把该记录的 `unresolved_reason` 写入用例 unresolved。
4. 一个 stub 函数对应多条 resolved 映射记录 → 生成多条 `(stub, final impl)` 对。
5. 阶段三不解析 impl 方法定义，不读取 system 调用链，也不做 impl 域判定。

### 7.3 cmdId 取值解析

| 形式 | 处理 |
|------|------|
| 整数字面量 | 直接取值 |
| `#define` 宏链 | 在 scope 内递归解析，维护宏展开 visited 防环 |
| 枚举常量 | 显式值或隐式递增 |
| `const` / `constexpr` | 读取初始化值 |
| macro.json 中的 `-D` 宏 | 辅助来源，不覆盖代码确定定义 |
| 算术/移位/位运算 | 常量可计算则求值；除数为 0 视为无法解析 |
| 范围 `A ... B` | 表达式原文，值 `null` |
| 掩码/组合表达式 | 表达式原文；能算单一值则记值，否则 `null` |
| 同名宏多定义/条件编译冲突 | 值 `null`，记 `cmd_value` unresolved 并列出候选 |
| 无法解析 | 值 `null`，记 `cmd_value` unresolved |

- `cmd_id_value = null` 不阻止映射进入最终 JSON；映射仍输出，unresolved 计入报告。
- default 不进入最终 JSON；具名 cmdId 即使值无法解析也必须保留。

### 7.4 写穿与防遗忘

- 预规划后立即写 `03_cases.json`，全部 `status = "pending"`。
- pending 用例的 stub/impl 字段可为 null；done 用例必须有完整 stub 接口函数与至少一个 impl 接口函数；unresolved 用例必须有 unresolved 记录。
- 每完成一个用例，立即更新 status（done/unresolved）、结果与证据并落盘。
- 每次更新先写 `03_cases.json.tmp`，Read 回读校验后 mv 覆盖。
- 处理下一个用例前核对清单，存在 pending 则继续，不得跳过。
- 全部处理完后不得存在 `pending`。
- 回读、mv 或写盘失败 → 删除残留 .tmp，写 `error.json`（`phase=trace_cases`，`code=snapshot_write_failed`），停止。
- 不做自动恢复；每次运行先备份旧输出，从零分析。

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

### 7.6 unresolved 类型

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
| `trace_ambiguous` | 映射 | system 层多个候选同等置信度，无法唯一选择 |
| `trace_no_candidate` | 映射 | system 层没有可信的下一跳候选 |
| `final_impl_not_found` | 映射 | 追踪耗尽仍没有 vendor 类成员函数作为最终接口 |
| `cmd_value` | 用例 | cmdId 值或宏表达式无法解析 |
| `definition_not_found` | 映射/用例 | 函数定义不在 scope 内或无法唯一确定 |
| `cycle` | 映射/用例 | 追踪路径形成真环 |

每条 unresolved 记录 `kind`、`detail`、`tried`；同一 `(kind, detail)` 去重，`tried` 合并去重。
```

- [ ] **Step 3: 验证第七章**

Run:

```bash
for t in stub_function_map trace_ambiguous trace_no_candidate final_impl_not_found impl_interfaces evidence_chain; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
grep -q 'impl_scope' HDF-api-finder/SKILL.md && echo "impl_scope=present" || echo "impl_scope=absent"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
for b in re.findall(r'```json\n(.*?)```', text, re.S):
    json.loads(b)
print("json_valid=ok")
PY
```

Expected: 无 `missing ...`；`impl_scope=absent`；`json_valid=ok`。

---

## Task 6: 重写第八章输出与自检

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第八章整章）

- [ ] **Step 1: 确认第八章边界**

Run:

```bash
grep -n '^## 八、\|^## 九、' HDF-api-finder/SKILL.md
```

Expected: 得到第八章起始行与第九章起始行。

- [ ] **Step 2: 用增量设计 §5 重写第八章**

将第八章整体替换为：

```markdown
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

1. 只从 `03_cases.json` 中 `status = "done"` 的用例组装；unresolved 不进入最终 JSON。
2. 一个 case 对应一个 `(stub 函数, 最终 impl 函数)` 对；同一 cmdId 命中多个 impl 函数时输出多条 case。
3. 入口字段：`on_remote_request` / `on_remote_request_file` / `on_remote_request_line` / `stub_class` / `impl_classes` / `cases`；case 字段：`cmd_id_macro` / `cmd_id_value` / `stub_interface_function` / `stub_interface_file` / `impl_interface_function` / `impl_interface_file`。不添加额外字段。
4. 没有可解析用例的入口不进入最终 JSON；全部入口都没有可解析用例时写 `"entries": []`。
5. `impl_classes` 只收录最终 vendor 类，按 `(impl_class, impl_file)` 去重，按 `impl_class` → `impl_file` 排序。
6. 同一入口内 case 按六元组去重：`(cmd_id_macro, cmd_id_value, stub_interface_function, stub_interface_file, impl_interface_function, impl_interface_file)`。
7. 排序：入口按 `on_remote_request_file` → `on_remote_request_line`；case 按 `cmd_id_value` 升序（null 最后）→ `cmd_id_macro` → `stub_interface_function` → `impl_interface_function`。
8. default、unresolved 不输出。
9. UTF-8，JSON 缩进 2 空格。
10. 先写 `<out>/<service_name>_api.json.tmp`，Read 回读校验通过后 mv 为最终文件；最终文件必须存在，否则任务失败。

### 8.2 自检

1. `01_entry.json` 中的全部入口都已处理。
2. `03_cases.json` 中不存在 `pending`。
3. 每个最终入口至少有一条 case。
4. 每个有 case 的入口 `impl_classes` 非空；每个 case 的最终 impl 函数所属类在 `impl_classes` 中。
5. 每条最终 case 的 stub 接口函数定义在 stub 域且文件在 `scope_stub_files.txt` 内；impl 接口函数定义在 vendor 域且文件在 `scope_vendor_files.txt` 内。
6. 所有 stub/impl 函数都是完整定义，函数名为定义处作用域限定名；重载以 `(文件, 定义行)` 区分。
7. default 没有混入最终 JSON；unresolved 没有混入最终 JSON。
8. 最终 JSON 字段集合与 8.1 完全一致；已按六元组去重并排序；`impl_classes` 无重复、已排序。
9. 写临时文件后 Read 回读，核对数组长度、字段名、类型；截断视为失败。
10. mv 后双方法确认最终文件存在（ls + Read）。

失败处理：

- 自检第 1–8 项失败 → `error.json`（`phase=output`，`code=self_check_failed`），停止，不写最终 JSON。
- 写临时文件、回读、mv 或最终文件存在性确认失败 → 删除残留 .tmp，写 `error.json`（`phase=output`，`code=output_write_failed`），停止。

### 8.3 完成报告

- scope 各域文件数与总数；
- service_name；
- 入口来源与数量；
- stub 类数量、初始 impl 类数量、最终 impl 类数量；
- cmdId 用例数（预规划、不含 default）；
- 已解析映射数（最终 JSON case 总数）；
- unresolved 数量；
- 最终输出文件绝对路径。

服务存在但未识别到可解析接口时，报告 0 条映射，并说明最终 JSON 为 `"entries": []`。
```

- [ ] **Step 3: 验证第八章**

Run:

```bash
for t in impl_classes '初始 impl 类数量、最终 impl 类数量' self_check_failed output_write_failed; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
objs = [json.loads(b) for b in blocks]
print(f"json_blocks={len(blocks)}")
for obj in objs:
    if isinstance(obj, dict) and 'entries' in obj:
        for e in obj['entries']:
            if isinstance(e, dict) and 'on_remote_request' in e and 'cases' in e:
                assert 'impl_classes' in e, 'final entry missing impl_classes'
                assert 'impl_class' not in e, 'final entry has old impl_class'
print("final_schema_ok")
PY
```

Expected: 无 `missing ...`；`final_schema_ok`。

---

## Task 7: 更新第十章约束、错误码、术语表

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第十章）

- [ ] **Step 1: 更新约束 4、5、7**

将约束 4：

```text
4. 入口只在 stub 域搜索；未指定 entry 时分析全部合法入口；指定 entry 时只分析该入口。
```

替换为：

```text
4. 入口只在 stub 域搜索；未指定 entry 时分析全部合法入口；指定 entry 时只分析该入口；初始 impl 可在 vendor/system，最终 impl 必须在 vendor。
```

将约束 5：

```text
5. stub 接口函数必须在 stub 域；impl 接口函数必须在 vendor 域；system 域函数只作辅助追踪。
```

替换为：

```text
5. stub 接口函数必须在 stub 域；初始 impl 可在 vendor/system；最终 impl 类与最终接口函数必须在 vendor 域；system 域函数只作辅助追踪/中间层。
```

将约束 7：

```text
7. impl 定位顺序：S1 特征匹配 → S2 命名 → S3 绑定点/注册流程；S3 绝不在 stub 域搜索。
```

替换为：

```text
7. impl 定位顺序：S1/S2/S3 在 vendor + system 统一搜索；S3 绝不在 stub 域搜索；初始 impl 在 system 时按 stub 接口函数做单线路追踪，只选最可能的下一跳，并列记 trace_ambiguous。
```

- [ ] **Step 2: 更新 error.json 与错误码表**

在错误码表中，将 `impl_not_found` 行替换为以下两行：

```markdown
| `impl_not_found` | 所有入口都没有初始 impl 类 | 写 error.json（phase=resolve_impl），结束 |
| `final_impl_not_found` | 至少一个入口有初始 impl，但所有入口都没有解析出 vendor 最终接口 | 写 error.json（phase=resolve_impl），结束 |
```

同时将 `self_check_failed` 行的触发条件从“自检第 1–7 项失败”改为“自检第 1–8 项失败”。

- [ ] **Step 3: 更新术语表**

在术语表中：

- 将 `impl 类` 行替换为三行：`初始 impl 类`、`中间 impl 类`、`最终 impl 类`。
- 新增 `单线路追踪` 行。
- 将 `impl 接口函数` 行更新为 `最终接口函数：stub 接口经初始/中间 impl 最终调用的 vendor 类成员函数`。
- 保留 `stub 域 / vendor 域 / system 域`、`scope`、`OnRemoteRequest 入口`、`stub 类`、`stub 接口函数`、`cmdId 用例`、`绑定点`、`注册流程`、`unresolved`。

使用以下文本：

```markdown
| 初始 impl 类 | 直接与 stub 绑定的类，允许定义在 vendor 或 system |
| 中间 impl 类 | 初始 impl 在 system 时的透传类；其接口方法不是最终接口 |
| 最终 impl 类 | 最终接口函数所属的 vendor 类，类定义与函数定义都必须在 vendor |
| 单线路追踪 | system 透传层每层只选择一个最可能的下一跳，直到 vendor 类成员函数 |
```

- [ ] **Step 4: 更新部分失败规则**

将“部分失败”小节补充/替换为：

```markdown
- 单个映射记录失败：记 unresolved，继续其他映射。
- 单个入口全部映射失败：入口 unresolved，继续其他入口。
- 单个用例追踪失败：记 unresolved，继续其他用例；不中止整体分析。
- 单个 cmdId 值解析失败：值记 null，映射仍输出，记 `cmd_value` unresolved。
- 全局不可恢复错误：写 `error.json`，停止。
```

- [ ] **Step 5: 验证第十章**

Run:

```bash
for t in 'final_impl_not_found' '初始 impl 类' '中间 impl 类' '最终 impl 类' '单线路追踪'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
grep -q 'impl_scope' HDF-api-finder/SKILL.md && echo "impl_scope=present" || echo "impl_scope=absent"
```

Expected: 无 `missing ...`；`impl_scope=absent`。

---

## Task 8: 更新第十一章抗理性化检查与第十二章示例

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第十一章、第十二章）

- [ ] **Step 1: 更新抗理性化检查表**

先将现有行：

```text
| “impl 在 stub/system 域也算” | impl 接口必须定义在 vendor 域。 |
```

替换为：

```text
| “impl 在 stub/system 域也算” | 初始 impl 可在 system；最终接口必须在 vendor 域。 |
```

然后在现有表格中补充以下行（保留其他原有行）：

```markdown
| “system impl 就是最终接口” | 不是；system 只是中间层，必须追到 vendor 类成员函数。 |
| “system 层有多个候选，随便选一个” | 多信号打分；并列记 trace_ambiguous。 |
| “追踪太深，先停在这里” | 不设深度上限，直到 vendor 类成员或明确终止条件。 |
| “vendor free function 也算接口” | 不算；继续追到 vendor 类成员函数。 |
| “初始 impl 只能在 vendor” | 初始 impl 可在 vendor/system；最终接口必须在 vendor。 |
| “虚函数按静态类型继续” | vendor 内存在唯一 override 时优先该 override。 |
| “system impl 就记 unresolved” | 不是；system 是合法中间层，继续追踪到 vendor。 |
| “阶段三再解析 impl 方法” | 阶段二已完成初始 impl → 最终接口映射；阶段三只查表。 |
```

- [ ] **Step 2: 更新使用示例**

将“示例一：首次分析”替换为 system 初始 impl 场景：

```markdown
### 示例一：system 初始 impl

```text
/HDF-api-finder /srv/demo/project /srv/demo/conf.json
```

1. 初始化：校验参数与 conf；备份旧输出；生成 scope 清单。
2. 阶段一：在 stub 域定位 `OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest`，校验命名空间、直接父类和四参数签名；写 `01_entry.json`。
3. 阶段二：S1 在 system 域定位初始 impl `DemoSystemService`；对 `GetVersion` 等 stub 接口函数做单线路追踪，最终落到 vendor 的 `DemoService::GetVersion`；写 `02_impl.json`（含 `initial_impl_*`、`trace`、`final_impl_*`、`impl_classes`）。
4. 阶段三：枚举 `CMD_DEMO_GET_VERSION` 等 cmdId 用例，从 `stub_function_map` 取 `DemoService::GetVersion`；写 `03_cases.json`。
5. 输出：自检后组装 `demo_service_api.json`，入口级使用 `impl_classes` 数组；写 `.tmp` 回读后 mv。
6. 报告：scope 文件数、cmdId 用例数、已解析映射数、unresolved 数、初始/最终 impl 类数、最终文件绝对路径。
```

新增“示例二：vendor 初始 impl（兼容场景）”：

```markdown
### 示例二：vendor 初始 impl

初始 impl 在 vendor 且方法定义也在 vendor 时，单线路追踪不产生额外中间层；`trace = []`，`final_impl_*` 直接取初始 impl 方法，`initial_impl_class = final_impl_class`。输出 JSON 的 `impl_classes` 只包含该 vendor 类。
```

将原“示例二：指定 entry”保留为“示例三：指定 entry”，内容不变。

- [ ] **Step 3: 验证第十一、十二章**

Run:

```bash
for t in 'system impl 就是最终接口' 'system 层有多个候选' '阶段三再解析 impl 方法' '示例一：system 初始 impl' '示例二：vendor 初始 impl' '示例三：指定 entry'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
```

Expected: 无 `missing ...`。

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
impl_entries = 0
case_entries = 0
for obj in objs:
    if not isinstance(obj, dict) or 'entries' not in obj:
        continue
    for e in obj['entries']:
        if not isinstance(e, dict):
            continue
        if 'on_remote_request' in e and 'cases' in e:
            final_entries += 1
            assert 'impl_classes' in e, 'final entry missing impl_classes'
            assert 'impl_class' not in e, 'final entry has old impl_class'
            assert 'impl_file' not in e, 'final entry has old impl_file'
        if 'stub_function_map' in e:
            impl_entries += 1
            assert 'initial_impl_class' in e
            assert 'initial_impl_domain' in e
            assert 'impl_classes' in e
        if 'cmd_param' in e:
            case_entries += 1
            assert 'cases' in e
print(f"final_entries={final_entries} impl_entries={impl_entries} case_entries={case_entries}")
print("json_schema_ok")
PY
```

Expected: `json_blocks>=8`、`final_entries>=2`、`impl_entries>=1`、`case_entries>=1`、`json_schema_ok`。

- [ ] **Step 4: 禁用字符串与占位符**

Run:

```bash
grep -nE 'codegraph|hal-service-api-finder|ar_service|ActivityRecognition|ActivityInterfaceStub|ArService|CMD_ACTIVITY_INTERFACE_GET_VERSION' HDF-api-finder/SKILL.md || echo "forbidden_strings=none"
grep -nE 'TBD|TODO|待定|FIXME|XXX' HDF-api-finder/SKILL.md || echo "placeholders=none"
grep -n 'impl_scope' HDF-api-finder/SKILL.md || echo "impl_scope=absent"
```

Expected: `forbidden_strings=none`、`placeholders=none`、`impl_scope=absent`。

- [ ] **Step 5: 新字段/新错误码/新 unresolved 覆盖**

Run:

```bash
for t in impl_classes initial_impl_class initial_impl_domain stub_function_map final_impl_function final_impl_class trace_ambiguous trace_no_candidate final_impl_not_found; do
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

- [ ] **Step 2: 初始/最终 impl 规则**

Run:

```bash
grep -q '初始 impl 类可在 vendor 或 system' HDF-api-finder/SKILL.md && echo "initial_domain=present"
grep -q '最终 impl 类与最终接口函数必须在 vendor' HDF-api-finder/SKILL.md && echo "final_domain=present"
grep -q 'vendor free function 只是中间 helper' HDF-api-finder/SKILL.md && echo "freefn=present"
grep -q 'vendor 内存在唯一 override 时优先该 override' HDF-api-finder/SKILL.md && echo "virtual=present"
```

Expected: 4 个 `present`。

- [ ] **Step 3: 单线路追踪与失败**

Run:

```bash
grep -q '多信号打分' HDF-api-finder/SKILL.md && echo "scoring=present"
grep -q 'trace_ambiguous' HDF-api-finder/SKILL.md && echo "ambiguous=present"
grep -q 'trace_no_candidate' HDF-api-finder/SKILL.md && echo "no_candidate=present"
grep -q 'final_impl_not_found' HDF-api-finder/SKILL.md && echo "final_missing=present"
grep -q '不设深度上限' HDF-api-finder/SKILL.md && echo "depth=present"
```

Expected: 5 个 `present`。

- [ ] **Step 4: 输出与状态流**

Run:

```bash
grep -q '"impl_classes"' HDF-api-finder/SKILL.md && echo "impl_classes=present"
grep -q '每个有 case 的入口 `impl_classes` 非空' HDF-api-finder/SKILL.md && echo "selfcheck=present"
grep -q '初始 impl 类数量、最终 impl 类数量' HDF-api-finder/SKILL.md && echo "report=present"
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
- 初始 impl / 中间 impl / 最终 impl / 单线路追踪术语一致；
- 阶段二与阶段三边界清晰，阶段三不重复解析 impl；
- 最终 JSON 入口级只有 `impl_classes`，不残留 `impl_class` / `impl_file`；
- 示例全部使用中性 demo 命名；
- 无 `impl_scope`、无代码图工具名称、无参考 skill 名称。

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
grep -nE 'codegraph|hal-service-api-finder|ar_service|ActivityRecognition|ActivityInterfaceStub|ArService|CMD_ACTIVITY_INTERFACE_GET_VERSION' HDF-api-finder/SKILL.md || echo "forbidden_strings=none"
grep -n 'impl_scope' HDF-api-finder/SKILL.md || echo "impl_scope=absent"
```

Expected: `12`、`json_valid=ok`、`forbidden_strings=none`、`impl_scope=absent`。

- [ ] **Step 3: 交付报告**

报告：

- SKILL.md 绝对路径；
- 行数与字节数；
- 章节数（12）；
- JSON 代码块数量；
- 新字段/新错误码/新 unresolved 检查结果；
- 明确声明：仅验证流程正确性，未验证真实项目中的接口识别准确性。

---

## Task 12: 提交并推送

**Files:**
- Modify: `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 检查 diff**

Run:

```bash
git status --short
git diff --stat
```

Expected: 仅 `HDF-api-finder/SKILL.md` 有未提交修改（设计/计划已在 Task 1 提交）。

- [ ] **Step 2: 提交 SKILL.md**

Run:

```bash
git add HDF-api-finder/SKILL.md
git -c user.name='siakam43' -c user.email='429872468@qq.com' commit -m "$(cat <<'EOF'
docs(HDF-api-finder): support system initial impl tracing

Allow the initial impl to live in vendor or system. Trace system
passthrough layers single-path to vendor class member functions, and
emit final impl classes via the impl_classes array.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

Expected: 提交成功。

- [ ] **Step 3: 推送**

Run:

```bash
git push origin main
```

Expected: 推送成功，`main` 跟踪 `origin/main`。

- [ ] **Step 4: 验证远端**

Run:

```bash
git log -1 --format='%h | %an <%ae> | %s'
git status -sb
gh repo view siakam43/HDF_api_finder --json nameWithOwner,visibility,defaultBranchRef,url
sha256sum HDF-api-finder/SKILL.md
curl -sL https://raw.githubusercontent.com/siakam43/HDF_api_finder/main/HDF-api-finder/SKILL.md | sha256sum
```

Expected: 作者为 `siakam43 <429872468@qq.com>`；`main...origin/main` 无未推送提交；仓库仍为 PUBLIC；本地与远端 SKILL.md 校验和一致。

---

## 计划自查

- 规格覆盖：增量设计 §3-§8 分别对应 Task 4、Task 5、Task 6、Task 7、Task 8、Task 9/10；基础设计指针与设计提交对应 Task 1；约束/错误码/术语对应 Task 7；示例对应 Task 8；提交推送对应 Task 12。
- 占位符：无 TBD/TODO；所有新 JSON schema 已给出；章节内容以增量设计 §3-§5 为权威来源，执行时必须完整读取。
- 类型/字段一致性：`initial_impl_class`、`initial_impl_domain`、`stub_function_map`、`final_impl_function`、`final_impl_class`、`impl_classes`、`trace_ambiguous`、`trace_no_candidate`、`final_impl_not_found` 在 Task 4-10 中一致使用。
