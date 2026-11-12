# HDF-api-finder impl 接口函数参数补充设计（增量）

- 日期：2026-11-12
- 状态：已实现，待用户验证
- 目标文件：`HDF-api-finder/SKILL.md`
- 基础设计：`docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md`
- 前置增量：`docs/superpowers/specs/2026-10-26-hdf-api-finder-system-impl-trace-design.md`、`docs/superpowers/specs/2026-11-04-hdf-api-finder-dynamic-target-fanout-design.md`、`docs/superpowers/specs/2026-11-09-hdf-api-finder-vendor-free-function-interface-design.md`
- 本设计为增量设计；与基础设计及前置增量冲突处，以本设计为准。

## 1. 背景与目标

最终输出 `<service_name>_api.json` 的 `impl_interface_function` 目前只记录限定函数名，缺少参数信息。本次需求：在最终 JSON 中补充最终接口函数的参数，以函数签名形式与函数名写在一起。

目标：

- `impl_interface_function` 改为 `限定名(参数列表)`，参数列表取自该最终接口函数在 vendor 域的定义处，例如 `OHOS::HDI::Demo::V1_0::DemoService::GetVersion(std::string& version)`。
- 参数列表在阶段二判定最终接口时记录，经 `02_impl.json`、`03_cases.json` 传递到最终 JSON。

边界：

- 仅补充 impl 侧；`stub_interface_function` 保持纯限定名不变。
- 最终 JSON 不新增字段，参数信息嵌入 `impl_interface_function`。
- 追踪、扇出、域边界、去重与排序规则不变。
- 本次为小需求，只添加流程与功能说明；多行参数、参数名缺失等特殊场景由执行 AI 自主处理，参数信息存在瑕疵可以接受，不为此新增专门规则。

## 2. 变更点

### 2.1 阶段二记录参数列表

- 6.4 步骤 4（初始 impl 方法即最终接口）与 6.5 步骤 6（追踪到达最终接口）：记录 `function` / `file` / `line` 时，同时记录该定义处的参数列表 `signature`。
- `signature` 包含括号本身；无参数时为 `()`；vendor 自由函数与 vendor 类成员函数规则相同。

### 2.2 02_impl.json

- `final_impl_functions[]` 元素增加 `signature` 字段。
- 6.6 字段规则补一条：`signature` 为该元素 `function` 对应定义处的参数列表，与 `function`、`file`、`line` 一同记录。

### 2.3 03_cases.json

- 7.2 生成 `impl_interfaces[]` 元素时，从 `02_impl.json` 对应元素复制 `signature`。
- 7.5 示例增加 `signature` 字段；`evidence_chain` 继续使用纯函数名，不带参数。

### 2.4 最终 JSON

- 8.1：`impl_interface_function` 由 `function` 与 `signature` 拼接得到，即 `限定名(参数列表)`。
- 最终 JSON 字段集合不变；六元组去重与 case 排序规则不变，自动使用拼接后的字符串。
- 第二节输出示例与 8.1 示例同步更新。

### 2.5 自检

- 8.2 增加一条：每个最终 case 的 `impl_interface_function` 已包含参数列表，且与 `02_impl.json` 对应元素的 `function` 与 `signature` 一致。

## 3. 示例

最终 JSON 的 case：

```json
{
  "cmd_id_macro": "CMD_DEMO_GET_VERSION",
  "cmd_id_value": 0,
  "stub_interface_function": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::GetVersion",
  "stub_interface_file": "/abs/stub/demo_interface_stub.cpp",
  "impl_interface_function": "OHOS::HDI::Demo::V1_0::DemoService::GetVersion(std::string& version)",
  "impl_interface_file": "/abs/vendor/demo_service.cpp"
}
```

自由函数接口同样适用，例如 `demo::hal_open(callback_t cb, void* data)`。

## 4. 不涉及

- `stub_interface_function`、完成报告、术语表、抗理性化检查表、权限表不变。
- 域归属判定、目标级扇出、visited、去重、排序、不设深度/扇出上限规则不变。
- 历史 spec/plan 不改写。
- 无代码改动、无新增依赖、无新增 unresolved 类型。

## 5. 影响范围

`HDF-api-finder/SKILL.md` 涉及：第二节输出示例、6.4、6.5、6.6、7.2、7.5、8.1、8.2、使用示例。

## 6. 验证方式

该 skill 是纯 markdown 提示词，仓库无自动化测试。验证分三层：

1. 措辞审计：`SKILL.md` 中不再存在最终接口只输出纯函数名的表述；所有示例的 `impl_interface_function` 均带参数列表。
2. Schema 审计：最终 JSON 字段集合不变；`02_impl.json` 与 `03_cases.json` 各新增 `signature` 字段且字段说明完整。
3. 端到端（建议）：在真实 HDF 项目上运行 `/HDF-api-finder`，抽查最终 JSON 中 `impl_interface_function` 的签名与 vendor 定义一致。
