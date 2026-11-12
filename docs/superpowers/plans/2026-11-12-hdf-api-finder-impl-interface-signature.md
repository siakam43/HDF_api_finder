# HDF-api-finder impl 接口函数参数补充实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 修改 `HDF-api-finder/SKILL.md`，让最终 JSON 的 `impl_interface_function` 输出 `限定名(参数列表)`，参数列表取自最终接口函数在 vendor 域的定义处。

**Architecture:** 只做文档级增量：阶段二判定最终接口时记录 `signature`，经 `02_impl.json`、`03_cases.json` 传递，输出阶段拼接进 `impl_interface_function`。最终 JSON 不新增字段；stub 侧、追踪、扇出、域边界、去重排序规则不变。特殊场景（多行参数、参数名缺失）由执行 AI 自行处理，不新增规则。

**Tech Stack:** Markdown、JSON、bash（grep/python3）、Read、Edit。

---

## 0. 输入与约束

**当前分支与提交：**

- 工作分支：`dev`（领先 `origin/dev` 1 个提交）。
- 增量设计已提交：`76102bc`，路径：

```text
docs/superpowers/specs/2026-11-12-hdf-api-finder-impl-interface-signature-design.md
```

**交付/修改文件：**

| 文件 | 责任 |
|------|------|
| `HDF-api-finder/SKILL.md` | 唯一需要修改的交付物 |
| `docs/superpowers/specs/2026-11-12-hdf-api-finder-impl-interface-signature-design.md` | 增量设计；Task 9 更新状态 |
| `docs/superpowers/plans/2026-11-12-hdf-api-finder-impl-interface-signature.md` | 本计划；Task 1 提交 |

**全局约束：**

- 修改前先完整读取增量设计；冲突处以该文档为准。
- 只修改 `HDF-api-finder/SKILL.md`；不改历史 spec/plan，不改其他文件（Task 9 例外：更新本增量设计状态与本计划复选框）。
- 本次为小需求：只加流程与功能说明，不新增边界规则、不扩充抗理性化表、不新增 unresolved 类型。
- 示例统一使用中性 demo 命名；SKILL.md 正文不得出现代码图/MCP 代码探索工具名称。
- 最终 JSON 字段集合不变；不新增 `impl_interface_signature` 等字段。
- 当前只在 `dev` 分支提交；不合入 `main`；不执行 `git push`。
- 每个 Task 完成后运行验证命令；发现 `missing`、旧文本残留或 JSON 解析失败时先修复再进入下一 Task。

---

## Task 1: 提交实现计划

**Files:**
- Add: `docs/superpowers/plans/2026-11-12-hdf-api-finder-impl-interface-signature.md`

- [x] **Step 1: 确认工作区状态**

Run:

```bash
git status --short
git branch --show-current
```

Expected: 只有本计划为未跟踪文件；当前分支为 `dev`。

- [x] **Step 2: 提交计划**

Run:

```bash
git add docs/superpowers/plans/2026-11-12-hdf-api-finder-impl-interface-signature.md
git commit -m "$(cat <<'EOF'
docs(HDF-api-finder): add impl interface signature implementation plan

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

## Task 2: 更新最终 JSON 示例与输出说明

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第二节「输出」）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '"impl_interface_function": "OHOS::HDI::Demo::V1_0::DemoService::GetVersion"' HDF-api-finder/SKILL.md
```

Expected: 2 处（第二节输出示例与 8.1 示例共用一个同文行）。

- [x] **Step 2: 更新两处示例中的 impl_interface_function**

使用 Edit，`replace_all: true`。

将：

```json
          "impl_interface_function": "OHOS::HDI::Demo::V1_0::DemoService::GetVersion",
```

替换为：

```json
          "impl_interface_function": "OHOS::HDI::Demo::V1_0::DemoService::GetVersion(std::string& version)",
```

- [x] **Step 3: 第二节输出规则补充一条**

将：

```text
- 不添加示例之外的字段；不生成 summary 报告。
```

替换为：

```text
- `impl_interface_function` 采用 `限定名(参数列表)` 格式，参数列表取自最终接口函数定义处。
- 不添加示例之外的字段；不生成 summary 报告。
```

- [x] **Step 4: 验证替换结果**

Run:

```bash
grep -c 'DemoService::GetVersion(std::string& version)' HDF-api-finder/SKILL.md
grep -n '"impl_interface_function".*GetVersion",' HDF-api-finder/SKILL.md || echo "old_field=absent"
grep -q '`impl_interface_function` 采用 `限定名(参数列表)` 格式' HDF-api-finder/SKILL.md && echo "format_rule=present"
```

Expected: 计数为 `2`；`old_field=absent`；`format_rule=present`。

---

## Task 3: 阶段二记录 signature（6.4 / 6.5 / 6.6）

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（6.4 步骤 4、6.5 步骤 6、6.6 示例与字段规则）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '直接作为最终接口：写入 `final_impl_functions` 数组' HDF-api-finder/SKILL.md
grep -n '记录到 `final_impl_functions`（含该分支 trace 与 `target_evidence`）' HDF-api-finder/SKILL.md
grep -n '"function": "OHOS::HDI::Demo::V1_0::DemoVendorA::HalOpen",' HDF-api-finder/SKILL.md
```

Expected: 前两条各 1 处；`DemoVendorA::HalOpen` 行存在（6.6 示例，1 处）。

- [x] **Step 2: 更新 6.4 步骤 4**

将：

```text
4. 方法定义在 vendor 类成员函数中，且该类的完整定义也在 vendor → 直接作为最终接口：写入 `final_impl_functions` 数组（一个元素，`trace = []`，`target_evidence = []`）。
```

替换为：

```text
4. 方法定义在 vendor 类成员函数中，且该类的完整定义也在 vendor → 直接作为最终接口：写入 `final_impl_functions` 数组（一个元素，`trace = []`，`target_evidence = []`，`signature` 取该定义处参数列表）。
```

- [x] **Step 3: 更新 6.5 步骤 6 的最终接口分支**

将：

```text
   - 最终接口（任一满足）：vendor 类成员函数且类定义与函数定义都在 vendor；vendor 自由函数且函数定义在 vendor → 记录到 `final_impl_functions`（含该分支 trace 与 `target_evidence`），分支结束。
```

替换为：

```text
   - 最终接口（任一满足）：vendor 类成员函数且类定义与函数定义都在 vendor；vendor 自由函数且函数定义在 vendor → 记录到 `final_impl_functions`（含该分支 trace、`target_evidence` 与该定义处参数列表 `signature`），分支结束。
```

- [x] **Step 4: 6.6 示例增加 signature 字段**

将：

```json
            {
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorA::HalOpen",
              "file": "/abs/vendor/demo_vendor_a.cpp",
              "line": 80,
              "impl_class": "OHOS::HDI::Demo::V1_0::DemoVendorA",
              "impl_class_file": "/abs/vendor/demo_vendor_a.h",
              "trace": [
```

替换为：

```json
            {
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorA::HalOpen",
              "signature": "(callback_t pCback, void* pDataCback)",
              "file": "/abs/vendor/demo_vendor_a.cpp",
              "line": 80,
              "impl_class": "OHOS::HDI::Demo::V1_0::DemoVendorA",
              "impl_class_file": "/abs/vendor/demo_vendor_a.h",
              "trace": [
```

说明：6.6 与 7.5 的 VendorA 元素前几行相同，必须带上 `"trace": [` 才能唯一定位 6.6。

- [x] **Step 5: 6.6 字段规则补充 signature 说明**

将：

```text
- `final_impl_functions` 每个元素对应一条最终接口（追踪分支或 6.4 直通映射）；元素自带该分支的 `trace`（含初始 impl 方法，不含最终函数）与 `target_evidence`；初始 impl 方法本身是最终接口时 `trace = []`。
```

替换为：

```text
- `final_impl_functions` 每个元素对应一条最终接口（追踪分支或 6.4 直通映射）；元素自带该分支的 `trace`（含初始 impl 方法，不含最终函数）与 `target_evidence`；初始 impl 方法本身是最终接口时 `trace = []`。
- `signature` 为元素 `function` 对应定义处的参数列表（含括号），与 `function`、`file`、`line` 一同记录；无参数为 `()`。
```

- [x] **Step 6: 验证替换结果**

Run:

```bash
grep -q '`signature` 取该定义处参数列表' HDF-api-finder/SKILL.md && echo "6.4=present"
grep -q '该定义处参数列表 `signature`' HDF-api-finder/SKILL.md && echo "6.5=present"
grep -q '"signature": "(callback_t pCback, void\* pDataCback)"' HDF-api-finder/SKILL.md && echo "6.6_example=present"
grep -q '`signature` 为元素 `function` 对应定义处的参数列表' HDF-api-finder/SKILL.md && echo "6.6_rule=present"
```

Expected: 四项均为 `present`。

---

## Task 4: 阶段三传递 signature（7.2 / 7.5）

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（7.2 用例映射、7.5 示例）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '每个元素生成一条 `(stub 接口函数, 最终 impl 函数)` 对。' HDF-api-finder/SKILL.md
grep -c '"function": "OHOS::HDI::Demo::V1_0::DemoVendor[AB]::HalOpen",' HDF-api-finder/SKILL.md
```

Expected: 7.2 有 1 处；`HalOpen` 的 function 行共 3 处（6.6 示例 1 处 + 7.5 示例 VendorA/VendorB 各 1 处）。

- [x] **Step 2: 更新 7.2 步骤 2**

将：

```text
2. 每条映射记录遍历 `final_impl_functions`，每个元素生成一条 `(stub 接口函数, 最终 impl 函数)` 对。
```

替换为：

```text
2. 每条映射记录遍历 `final_impl_functions`，每个元素生成一条 `(stub 接口函数, 最终 impl 函数)` 对，并复制该元素的 `signature`。
```

- [x] **Step 3: 7.5 示例 VendorA 元素增加 signature**

将：

```json
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorA::HalOpen",
              "file": "/abs/vendor/demo_vendor_a.cpp",
```

替换为：

```json
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorA::HalOpen",
              "signature": "(callback_t pCback, void* pDataCback)",
              "file": "/abs/vendor/demo_vendor_a.cpp",
```

- [x] **Step 4: 7.5 示例 VendorB 元素增加 signature**

将：

```json
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorB::HalOpen",
              "file": "/abs/vendor/demo_vendor_b.cpp",
```

替换为：

```json
              "function": "OHOS::HDI::Demo::V1_0::DemoVendorB::HalOpen",
              "signature": "(callback_t pCback, void* pDataCback)",
              "file": "/abs/vendor/demo_vendor_b.cpp",
```

- [x] **Step 5: 验证替换结果**

Run:

```bash
grep -q '并复制该元素的 `signature`' HDF-api-finder/SKILL.md && echo "7.2=present"
grep -c '"signature": "(callback_t pCback, void\* pDataCback)"' HDF-api-finder/SKILL.md
```

Expected: `7.2=present`；计数为 `3`（6.6 示例 1 处 + 7.5 示例 2 处）。

---

## Task 5: 输出阶段拼接与自检（8.1 / 8.2）

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（8.1 规则 3、8.2 自检与失败处理）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '`impl_interface_function` / `impl_interface_file`。不添加额外字段。' HDF-api-finder/SKILL.md
grep -n '第 11–15 项为写盘前检查' HDF-api-finder/SKILL.md
grep -n '自检第 1–8 项或第 11–15 项失败' HDF-api-finder/SKILL.md
```

Expected: 各 1 处。

- [x] **Step 2: 更新 8.1 规则 3**

将：

```text
3. 入口字段：`on_remote_request` / `on_remote_request_file` / `on_remote_request_line` / `stub_class` / `impl_classes` / `cases`；case 字段：`cmd_id_macro` / `cmd_id_value` / `stub_interface_function` / `stub_interface_file` / `impl_interface_function` / `impl_interface_file`。不添加额外字段。
```

替换为：

```text
3. 入口字段：`on_remote_request` / `on_remote_request_file` / `on_remote_request_line` / `stub_class` / `impl_classes` / `cases`；case 字段：`cmd_id_macro` / `cmd_id_value` / `stub_interface_function` / `stub_interface_file` / `impl_interface_function` / `impl_interface_file`；`impl_interface_function` 由 `03_cases.json` 中对应元素的 `function` 与 `signature` 拼接为 `限定名(参数列表)`。不添加额外字段。
```

- [x] **Step 3: 8.2 增加自检项并更新顺序说明**

将：

```text
15. 部分失败分支只出现在 unresolved 记录中，不得混入最终 JSON。

第 11–15 项为写盘前检查，必须在第 9–10 项写盘之前执行。
```

替换为：

```text
15. 部分失败分支只出现在 unresolved 记录中，不得混入最终 JSON。
16. 每个最终 case 的 `impl_interface_function` 已包含参数列表，且与 `02_impl.json` 对应元素的 `function` 与 `signature` 拼接结果一致。

第 11–16 项为写盘前检查，必须在第 9–10 项写盘之前执行。
```

- [x] **Step 4: 更新自检失败处理引用**

将：

```text
- 自检第 1–8 项或第 11–15 项失败 → `error.json`（`phase=output`，`code=self_check_failed`），停止，不写最终 JSON。
```

替换为：

```text
- 自检第 1–8 项或第 11–16 项失败 → `error.json`（`phase=output`，`code=self_check_failed`），停止，不写最终 JSON。
```

同时更新「全局错误码」表中同一引用：

将：

```text
| `self_check_failed` | 自检第 1–8 项或第 11–15 项失败 | 写 error.json（phase=output），结束 |
```

替换为：

```text
| `self_check_failed` | 自检第 1–8 项或第 11–16 项失败 | 写 error.json（phase=output），结束 |
```

- [x] **Step 5: 验证替换结果**

Run:

```bash
grep -q '`impl_interface_function` 由 `03_cases.json` 中对应元素的 `function` 与 `signature` 拼接' HDF-api-finder/SKILL.md && echo "8.1_rule=present"
grep -q '16. 每个最终 case 的 `impl_interface_function` 已包含参数列表' HDF-api-finder/SKILL.md && echo "8.2_item=present"
grep -q '第 11–16 项为写盘前检查' HDF-api-finder/SKILL.md && echo "8.2_order=present"
grep -q '第 1–8 项或第 11–16 项失败' HDF-api-finder/SKILL.md && echo "8.2_failure=present"
grep -n '第 11–15 项\|第 1–8 项或第 11–15 项' HDF-api-finder/SKILL.md || echo "old_refs=absent"
```

Expected: 四项均为 `present`；`old_refs=absent`。

---

## Task 6: 更新使用示例六

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第十二章 示例六）

- [x] **Step 1: 确认旧文本存在**

Run:

```bash
grep -n '`function = "demo::hal_open"`' HDF-api-finder/SKILL.md
grep -n '`impl_interface_function = "demo::hal_open"`' HDF-api-finder/SKILL.md
```

Expected: 各 1 处。

- [x] **Step 2: 更新 02_impl 元素说明**

将：

```text
1. `02_impl.json` 的 `final_impl_functions` 元素：`function = "demo::hal_open"`，`impl_class = null`，`impl_class_file = null`，`trace` 含 `DemoSystemService::VendorOpen`。
```

替换为：

```text
1. `02_impl.json` 的 `final_impl_functions` 元素：`function = "demo::hal_open"`，`signature = "(callback_t cb, void* data)"`，`impl_class = null`，`impl_class_file = null`，`trace` 含 `DemoSystemService::VendorOpen`。
```

- [x] **Step 3: 更新最终 JSON case 说明**

将：

```text
2. 最终 JSON 对该 cmdId 输出 case：`impl_interface_function = "demo::hal_open"`，`impl_interface_file = "/abs/vendor/demo_hal.cpp"`。
```

替换为：

```text
2. 最终 JSON 对该 cmdId 输出 case：`impl_interface_function = "demo::hal_open(callback_t cb, void* data)"`，`impl_interface_file = "/abs/vendor/demo_hal.cpp"`。
```

- [x] **Step 4: 验证替换结果**

Run:

```bash
grep -q '`signature = "(callback_t cb, void\* data)"`' HDF-api-finder/SKILL.md && echo "example6_02=present"
grep -q '`impl_interface_function = "demo::hal_open(callback_t cb, void\* data)"`' HDF-api-finder/SKILL.md && echo "example6_out=present"
grep -n '`impl_interface_function = "demo::hal_open"`' HDF-api-finder/SKILL.md || echo "old_example6=absent"
```

Expected: 前两项 `present`；`old_example6=absent`。

---

## Task 7: 全文一致性审计

**Files:**
- Check: `HDF-api-finder/SKILL.md`

- [x] **Step 1: 检查 JSON 示例字段与 JSON 有效性**

Run:

```bash
grep -n '"impl_interface_function":' HDF-api-finder/SKILL.md
python3 - <<'PY'
import json, re
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
assert blocks, 'no json blocks'
for i, block in enumerate(blocks, 1):
    json.loads(block)
print(f'json_blocks_ok={len(blocks)}')
PY
```

Expected: `impl_interface_function` 的 JSON 行均带 `(`；`json_blocks_ok` 输出且无异常。

- [x] **Step 2: 检查新字段与旧口径**

Run:

```bash
grep -c '"signature":' HDF-api-finder/SKILL.md
grep -n '"impl_interface_function": "[^"]*",' HDF-api-finder/SKILL.md | grep -v '(' || echo "pure_name_values=absent"
grep -n '只记录限定函数名\|impl_interface_function.*只有函数名' HDF-api-finder/SKILL.md || echo "old_wording=absent"
```

Expected: 计数为 `3`；`pure_name_values=absent`；`old_wording=absent`。

- [x] **Step 3: 检查未越界修改**

Run:

```bash
git diff --stat
git diff -- HDF-api-finder/SKILL.md | grep -nE '^\+.*(抗理性化|unresolved 类型|权限申请)' || echo "no_out_of_scope_additions"
git diff -- HDF-api-finder/SKILL.md | head -200
```

Expected: 仅 `HDF-api-finder/SKILL.md` 被修改；`no_out_of_scope_additions`；diff 仅涉及第二节、6.4、6.5、6.6、7.2、7.5、8.1、8.2、示例六。

- [x] **Step 4: 记录审计结论**

在对话中报告：JSON 示例块数量、`signature` 字段数量、旧口径检查结果、diff 涉及小节。明确声明：仅验证文本与 JSON 结构一致性，未在真实 HDF 项目验证参数提取准确性。

---

## Task 8: 提交 SKILL.md

**Files:**
- Modify: `HDF-api-finder/SKILL.md`

- [x] **Step 1: 检查待提交内容**

Run:

```bash
git status --short
git diff --stat
```

Expected: 只有 `HDF-api-finder/SKILL.md` 被修改。

- [x] **Step 2: 提交**

Run:

```bash
git add HDF-api-finder/SKILL.md
git commit -m "$(cat <<'EOF'
docs(HDF-api-finder): add impl interface parameters to final JSON

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

## Task 9: 标记设计与计划完成

**Files:**
- Modify: `docs/superpowers/specs/2026-11-12-hdf-api-finder-impl-interface-signature-design.md`
- Modify: `docs/superpowers/plans/2026-11-12-hdf-api-finder-impl-interface-signature.md`

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
git add docs/superpowers/specs/2026-11-12-hdf-api-finder-impl-interface-signature-design.md docs/superpowers/plans/2026-11-12-hdf-api-finder-impl-interface-signature.md
git commit -m "$(cat <<'EOF'
docs(HDF-api-finder): mark impl interface signature plan complete

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
git log --oneline --decorate -6
git branch --show-current
```

Expected: `dev` 分支领先 `main` 4 个提交（设计、计划、SKILL.md、完成标记）；未合并到 `main`；未 push。
