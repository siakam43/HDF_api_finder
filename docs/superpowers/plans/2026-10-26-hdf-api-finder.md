# HDF-api-finder Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建纯 markdown skill `HDF-api-finder/SKILL.md`，完整描述 OnRemoteRequest 入口定位、impl 定位、cmdId 用例追踪与按入口分组的 JSON 输出流程。

**Architecture:** 单文件 SKILL.md，共 12 个章节；运行时按“初始化 → 阶段一入口 → 阶段二 impl → 阶段三用例 → 输出与自检”顺序执行，每阶段写中间快照，阶段三逐用例写穿 `03_cases.json`。无代码、无自动恢复、无 sub-agent。

**Tech Stack:** Markdown、JSON、bash（grep/find/xargs/ls/mkdir/mv/rm）、Read、Write；可选 ast-grep。

---

## 0. 输入与约束

**设计文档（唯一权威来源）：**

```text
docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md
```

**交付物（唯一需要创建的文件）：**

```text
HDF-api-finder/SKILL.md
```

**全局约束：**

- SKILL.md 正文全中文；技术标识符（OnRemoteRequest、cmdId、stub、impl 等）保留英文。
- 正文不得出现任何代码图/MCP 代码探索工具名称，不得出现参考 skill 名称，不得使用需求文档中的原示例值。
- 示例统一使用中性 demo 命名：`demo_service`、`DemoInterfaceStub`、`IDemoInterface`、`DemoService`、`CMD_DEMO_GET_VERSION`。
- 当前项目目录不是 git 仓库；每个任务末尾的提交步骤是条件步骤，只有 `git rev-parse --is-inside-work-tree` 成功时才执行。不要主动 `git init`，如需初始化先询问用户。
- 每个任务先运行“检查缺失”命令确认当前状态，再写内容，最后运行“检查通过”命令。

## 1. 文件结构

| 文件 | 责任 |
|------|------|
| `HDF-api-finder/SKILL.md` | 唯一交付物；完整 skill 提示词 |
| `docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md` | 设计与规则来源；只读 |
| `docs/superpowers/plans/2026-10-26-hdf-api-finder.md` | 本实现计划 |

---

## Task 1: 创建 skill 目录与 SKILL.md 骨架

**Files:**
- Create: `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 确认当前目录与目标路径状态**

Run:

```bash
pwd
ls -la
ls -la HDF-api-finder 2>&1 || true
```

Expected: 当前目录为 `/home/admin/cc/wksp/siakam_security_skills/HDF_api_finder`；`HDF-api-finder` 不存在或为空目录。

- [ ] **Step 2: 创建目录**

Run:

```bash
mkdir -p HDF-api-finder
```

Expected: `ls -la HDF-api-finder` 可见空目录。

- [ ] **Step 3: 写入 frontmatter 与简介**

写入以下内容作为文件开头：

```markdown
---
name: HDF-api-finder
description: 当用户调用 /HDF-api-finder <project_dir> <conf>，或要求在 OpenHarmony/HDF 层 C/C++ 代码中识别指定 service 对外提供的全部接口函数时使用。根据 OnRemoteRequest 入口、cmdId 分发和 stub→impl 调用关系，输出按入口分组的 stub/impl 接口 JSON 映射。
---

# HDF-api-finder

根据 HDF/HAL 层 service 的 IPC stub 实现，识别指定 service 对外提供的全部接口函数，输出 cmdId 到 stub 接口函数、impl 接口函数的映射，用于安全攻击面分析。
```

- [ ] **Step 4: 写入 12 个章节标题**

按顺序写入以下标题，暂不写正文：

```markdown
## 一、核心原则

## 二、使用方法与输出

## 三、初始化与分析范围

## 四、中间结果记录

## 五、阶段一：定位 OnRemoteRequest 入口

## 六、阶段二：定位 impl 类

## 七、阶段三：cmdId 用例与 stub→impl 映射

## 八、输出与自检

## 九、权限申请

## 十、约束规则

## 十一、抗理性化检查

## 十二、使用示例
```

- [ ] **Step 5: 验证骨架**

Run:

```bash
head -5 HDF-api-finder/SKILL.md
grep -c '^## ' HDF-api-finder/SKILL.md
grep -nE '^## (一|二|三|四|五|六|七|八|九|十|十一|十二)、' HDF-api-finder/SKILL.md | wc -l
```

Expected:

- 前 3 行为 `---`、`name: HDF-api-finder`、`description: ...`。
- `grep -c '^## '` 输出 `12`。
- 第二个 `wc -l` 输出 `12`。

- [ ] **Step 6: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: scaffold HDF-api-finder skill"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 已初始化仓库时提交成功；否则输出 `not a git repository; skip commit`。

---

## Task 2: 写第一章核心原则与第二章使用方法与输出

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第一章、第二章）

- [ ] **Step 1: 检查缺失**

Run:

```bash
grep -q '## 一、核心原则' HDF-api-finder/SKILL.md && echo "heading=present"
grep -q 'S1 特征匹配' HDF-api-finder/SKILL.md || echo "principles=missing"
```

Expected: `heading=present`、`principles=missing`。

- [ ] **Step 2: 写第一章“核心原则”**

在 `## 一、核心原则` 下写入以下 10 条原则（每条一段，保留编号）：

1. **分析质量优先。** 宁可慢，不可草率；不设深度上限、不设扇出上限；所有静态可确定的 cmdId 用例全部追踪；环检测是唯一循环保护。
2. **scope 是硬边界。** 所有搜索只使用 `scope_files.txt` 或分类清单；禁止 `grep -rn <project_dir>`、`rg <project_dir>`、`find <project_dir> -exec grep`。
3. **域边界。** 入口只在 stub 域搜索；impl 接口必须定义在 vendor 域；system 域函数只作辅助追踪，不能作为 stub 接口或 impl 接口输出。
4. **只分析实际调用的 stub 函数。** 只分析 OnRemoteRequest 中直接调用的 stub 成员函数，不扫描 stub 类的其他成员函数。
5. **证据优先，不臆造。** 无法确定的分析项写入 unresolved，记录 `detail` 与 `tried`，不得猜测。
6. **全部由主 agent 完成。** 不调用 sub-agent，不把分析拆给子代理。
7. **每次从零分析。** 先备份旧输出，再从零分析；不做自动恢复，不复用历史结果。
8. **函数名带作用域。** C++ 函数名使用定义处作用域限定名；重载以 `(文件, 定义行)` 区分。
9. **impl 定位顺序。** S1 特征匹配 → S2 命名启发式 → S3 绑定点与注册流程；S3 绝不在 stub 域搜索。
10. **中文与独立性。** 全中文提示词，同一概念用词前后一致；正文不得出现任何代码图/MCP 代码探索工具，不体现与其他 skill 的关联性。

- [ ] **Step 3: 写第二章“使用方法与输出”**

在 `## 二、使用方法与输出` 下写入：

**使用方法：**

```text
/HDF-api-finder <project_dir> <conf>
```

- 两个参数均必填；缺失、为空、仅含空白或多于两个 → `usage_error`，打印用法并结束。
- `<project_dir>` 必须存在且是目录，解析为绝对路径。
- `<conf>` 是 JSON 配置文件；相对路径先按当前工作目录解析，找不到再按 `<project_dir>` 解析。
- conf 校验失败时报告具体字段与原因，不创建输出目录。

**conf JSON：**

```json
{
  "service_name": "demo_service",
  "stub_code": ["/abs/stub"],
  "vendor_code": ["/abs/vendor"],
  "system_code": ["/abs/system"],
  "entry": "/abs/stub/demo_interface_stub.cpp:123"
}
```

- 必填：`service_name`、`stub_code`、`vendor_code`；可选：`system_code`、`entry`。
- `service_name` 仅允许字母、数字、下划线、点、连字符。
- 路径必须为绝对路径；可指向目录或文件；目录递归展开。
- 未知字段 → `invalid_conf`。

**最终输出：**

```text
<project_dir>/.ethunter_out/HDF-api-finder/<service_name>/<service_name>_api.json
```

**最终 JSON 结构（按入口分组，只含已解析项）：**

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

**完成报告字段：** scope 各域文件数与总数；service_name；入口来源（conf/discovered）与入口数量；stub 类数量、impl 类数量；cmdId 用例数（不含 default）；已解析映射数；unresolved 数量；最终输出文件绝对路径。

- 不生成 summary 报告；不添加最终 JSON 之外的文件。

- [ ] **Step 4: 验证第一章与第二章**

Run:

```bash
grep -q 'S1 特征匹配' HDF-api-finder/SKILL.md && echo "principles=present"
grep -q 'usage_error' HDF-api-finder/SKILL.md && echo "usage=present"
grep -q 'service_name.*demo_service' HDF-api-finder/SKILL.md && echo "conf_example=present"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
print(f"json_blocks={len(blocks)}")
for i, b in enumerate(blocks, 1):
    json.loads(b)
print("json_valid=ok")
PY
```

Expected: `principles=present`、`usage=present`、`conf_example=present`、`json_blocks>=2`、`json_valid=ok`。

- [ ] **Step 5: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: add core principles and usage"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 成功或跳过提示。

---

## Task 3: 写第三章初始化与分析范围与第四章中间结果记录

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第三章、第四章）

- [ ] **Step 1: 检查缺失**

Run:

```bash
grep -q 'scope_files.txt' HDF-api-finder/SKILL.md && echo "scope=present" || echo "scope=missing"
grep -q '03_cases.json' HDF-api-finder/SKILL.md && echo "cases=present" || echo "cases=missing"
```

Expected: `scope=missing`、`cases=missing`。

- [ ] **Step 2: 写第三章“初始化与分析范围”**

必须包含以下小节与规则（完整措辞参考设计文档 §3、§4）：

**3.1 参数解析**

- 校验两个参数非空；缺失、为空或参数多于两个 → `usage_error`。
- `<project_dir>` 解析为绝对路径；不存在或不是目录 → `invalid_project_dir`。
- `<conf>` 不存在 → `conf_not_found`；JSON 解析失败、必填字段缺失/类型错误、路径非绝对、service_name 非法、entry 格式错误、未知字段 → `invalid_conf`。

**3.2 conf 字段表**

| 字段 | 必填 | 类型 | 规则 |
|------|------|------|------|
| `service_name` | 是 | string | 非空；仅字母、数字、下划线、点、连字符 |
| `stub_code` | 是 | string[] | 非空；每项非空绝对路径 |
| `vendor_code` | 是 | string[] | 非空；每项非空绝对路径 |
| `system_code` | 否 | string[] | 默认空；每项非空绝对路径 |
| `entry` | 否 | string | `绝对文件路径:正整数行号` |

- `entry` 格式不合法 → `invalid_conf`；文件存在性、是否属于 stub 域、是否为合法定义，由阶段一校验（`entry_invalid`）。

**3.3 输出目录初始化**

- `<out>` = `<project_dir>/.ethunter_out/HDF-api-finder/<service_name>`。
- `<out>` 已存在：删除旧 `<out>_bak` → `mv <out> <out>_bak` → 新建空 `<out>`；不得直接删除旧结果。
- `<out>` 不存在时保留现有 `<out>_bak`。

**3.4 macro.json**

- 路径 `<project_dir>/.ethunter_out/macro.json`；按需查阅，不预先全量读取。
- `#ifdef` / `#if`：先查代码，再查 macro.json；都无定义时默认分析对应分支；无法判断时分析受保护分支；`#if/#else` 无法判断时两个分支都分析。

**3.5 scope 生成**

- 展开 `stub_code`、`vendor_code`、`system_code` 中全部 C/C++ 文件；扩展名 `.c .h .cpp .cxx .cc .hpp .hxx .hh .inl .ipp`。
- 生成文件清单：

| 文件 | 内容 |
|------|------|
| `scope_files.txt` | 三类源码并集 |
| `scope_stub_files.txt` | stub 域文件 |
| `scope_vendor_files.txt` | vendor 域文件 |
| `scope_system_files.txt` | system 域文件；system_code 为空时不生成 |
| `scope_roots.json` | 三类根路径、文件条目、missing 列表 |

- 不存在的路径记入 `scope_roots.json` 的 `missing`，不中断分析。
- stub 域或 vendor 域展开后为空 → `error.json`（`phase=init`，`code=empty_scope`），停止；system 域为空不构成失败。

**3.6 域归属判定**

- 文件路径以某个域根路径为前缀，或等于该域清单中的文件条目 → 属于该域。
- 允许一个文件同时属于多个域。
- stub 接口函数必须定义在 stub 域；impl 接口函数必须定义在 vendor 域；system 域函数只作辅助追踪。

**3.7 搜索方法与纪律**

写入以下 4 条命令模板：

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

纪律：

- 只搜索 scope 清单；禁止全仓搜索。
- `scope_system_files.txt` 不存在时，不执行 system 域搜索。
- `grep -I` 保留；不得用 `2>/dev/null` 隐藏错误；退出码 1/123 表示无匹配，不是失败。
- 路径变量一律双引号；不使用任何代码图/MCP 代码探索工具；使用 grep、find、Read，可选 ast-grep。

**3.8 文件存在性检查**

- 双方法：`ls <path>` + `Read <path>`（目录返回目录类错误也视为存在）。
- 两次一致 → 采用；不一致 → 用 find 等第三种方法取多数；仍不确定 → 停止并报告。
- 禁止 `test -f` / `[ -f ]`。

- [ ] **Step 3: 写第四章“中间结果记录”**

在 `## 四、中间结果记录` 下写入：

| 文件 | 写入时机 |
|------|----------|
| `scope_files.txt` / `scope_stub_files.txt` / `scope_vendor_files.txt` / `scope_system_files.txt` / `scope_roots.json` | 初始化阶段 |
| `01_entry.json` | 阶段一结束 |
| `02_impl.json` | 阶段二结束 |
| `03_cases.json` | 预规划后首次写入，之后每完成一个用例更新一次 |
| `error.json` | 任一阶段失败时 |
| `<service_name>_api.json` | 输出阶段，原子写入 |

规则：

- 长任务防遗忘依靠 `03_cases.json` 的用例清单与 `status` 写穿；每个用例处理前后核对清单，不得跳过 `pending`。
- 不做自动恢复；每次执行先备份旧输出，再从零分析。
- 中间文件只用于调试与人工查看，不得作为下次运行输入。

- [ ] **Step 4: 验证第三章与第四章**

Run:

```bash
for f in scope_files.txt scope_stub_files.txt scope_vendor_files.txt scope_system_files.txt scope_roots.json 01_entry.json 02_impl.json 03_cases.json error.json; do
  grep -q "$f" HDF-api-finder/SKILL.md || echo "missing $f"
done
grep -q 'xargs -r -a' HDF-api-finder/SKILL.md && echo "search=present"
grep -q '不做自动恢复' HDF-api-finder/SKILL.md && echo "recovery=present"
grep -q '禁止 `test -f`' HDF-api-finder/SKILL.md && echo "dualcheck=present"
```

Expected: 无 `missing ...` 行；`search=present`、`recovery=present`、`dualcheck=present`。

- [ ] **Step 5: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: add initialization scope and snapshots"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 成功或跳过提示。

---

## Task 4: 写第五章阶段一：定位 OnRemoteRequest 入口

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第五章）

- [ ] **Step 1: 检查缺失**

Run:

```bash
grep -q 'entry_invalid' HDF-api-finder/SKILL.md && echo "entry=present" || echo "entry=missing"
```

Expected: `entry=missing`。

- [ ] **Step 2: 写第五章正文**

必须包含：

**5.1 指定 entry**

1. 解析 `<绝对文件路径>:<行号>`；文件必须存在于 stub 域，行号在该文件行数范围内。
2. 读取文件，确认行号落在某个 `OnRemoteRequest` 成员函数定义范围（签名行到配对闭合 `}`）内；归一化到签名行。
3. 校验：
   - 是成员函数定义，不是声明、调用、注释或字符串。
   - 所属类命名空间以 `OHOS::HDI` 开头。
   - 所属类完整定义在 stub 域内。
   - 所属类直接基类列表包含 `IPCObjectStub`；按最后一级标识符匹配，允许命名空间限定；不得经 `IRemoteStub<T>` 等中间基类间接继承。
   - 参数恰好四个，按位置类型为 `uint32_t`、`MessageParcel&`、`MessageParcel&`、`MessageOption&`；允许空白差异；返回类型不限。
   - 参数名任意：第一个 cmdId，第二个 data，第三个 reply，第四个 option。
4. 任一不满足 → `error.json`（`phase=locate_entry`，`code=entry_invalid`），停止，不写 `01_entry.json`。
5. 只分析该入口。

**5.2 未指定 entry**

1. 在 `scope_stub_files.txt` 内搜索 `OnRemoteRequest`。
2. 读取上下文，只接受完整成员函数定义；调用、声明、注释、字符串丢弃并记录原因。
3. 按 5.1 的类与方法规则校验；按 `(文件, 签名行)` 去重。
4. 0 个合法入口 → `error.json`（`phase=locate_entry`，`code=entry_not_found`），停止。
5. 多个合法入口 → 全部作为入口，逐个独立分析。

**5.3 01_entry.json**

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
  "rejected_candidates": []
}
```

- `entry_source` 取 `conf` 或 `discovered`。
- 参数名仅示例；实际按位置与类型识别。
- 写入失败 → `error.json`（`phase=locate_entry`，`code=snapshot_write_failed`），停止。

- [ ] **Step 3: 验证第五章**

Run:

```bash
grep -q 'entry_invalid' HDF-api-finder/SKILL.md && echo "entry_invalid=present"
grep -q 'entry_not_found' HDF-api-finder/SKILL.md && echo "entry_not_found=present"
grep -q 'IPCObjectStub' HDF-api-finder/SKILL.md && echo "parent=present"
grep -q 'MessageOption&' HDF-api-finder/SKILL.md && echo "params=present"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
for b in re.findall(r'```json\n(.*?)```', text, re.S):
    json.loads(b)
print("json_valid=ok")
PY
```

Expected: 4 个 `present`、`json_valid=ok`。

- [ ] **Step 4: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: add phase one entry location"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 成功或跳过提示。

---

## Task 5: 写第六章阶段二：定位 impl 类

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第六章）

- [ ] **Step 1: 检查缺失**

Run:

```bash
grep -q 'feature_match' HDF-api-finder/SKILL.md && echo "strategy=present" || echo "strategy=missing"
grep -q 'impl_ambiguous' HDF-api-finder/SKILL.md && echo "ambiguity=present" || echo "ambiguity=missing"
```

Expected: `strategy=missing`、`ambiguity=missing`。

- [ ] **Step 2: 写第六章正文**

必须包含：

**6.1 分析范围收窄**

- 读取 OnRemoteRequest 完整函数体，只收集其中被直接调用的 stub 成员函数。
- 排除 `IPCObjectStub::OnRemoteRequest`、基类框架调用、日志/断言/纯校验、构造/析构/运算符、free 函数、`GetDescriptor`。
- 只读取这些 stub 接口函数的函数体；不扫描 stub 类其他成员。
- 某个 stub 接口函数在 stub 域内找不到完整定义 → 跳过其函数体特征，阶段三记 `stub_interface_scope`。
- OnRemoteRequest 没有调用任何 stub 成员函数 → S1/S2 无法打分，直接进入 S3。

**6.2 特征收集**

- impl 成员候选：`sptr<I...>` / `sptr<...>` 成员、名称含 `impl`/`service` 的成员、stub 函数体中作为接收者的成员。
- 接口类型候选：impl 成员声明类型、stub 类命名、`GetDescriptor` 返回值、stub 函数签名中的接口类型。
- impl 方法名集合：`impl_->Foo(...)` 中的 `Foo`。
- 命名候选：stub 类名去掉 `Stub`/`InterfaceStub` 后的基名，尝试 `<Base>`、`<Base>Impl`、`<Base>Service`、`<Base>StubImpl`、`<Base>ImplService`。

**6.3 impl 定位优先级**

按 S1 → S2 → S3 顺序，命中唯一结果即停。

S1 特征匹配（首选）：

- 在 vendor 域找候选：实现接口类型的类、定义与 stub 接口方法集合一一对应的类、继承 stub 类的实现类。
- 打分从高到低：impl 方法名覆盖率 → 接口继承关系 → 同命名空间前缀 → 类名相似度。
- impl 方法名集合为空时，退化为用 stub 接口函数名（去掉命名空间与类限定）做覆盖度匹配。
- 唯一最高分 → 选定，`strategy = "feature_match"`；并列 → S2；仍不确定 → S3。

S2 命名启发式（辅助排序）：

- 用 6.2 的命名候选对候选类排序，只用于打破 S1 平局，不单独作为定论。
- 唯一候选 → `strategy = "naming"`；仍并列 → S3。

S3 绑定点 + 注册流程（最后，合并为一档）：

- 仅当 S1/S2 无法唯一确定时执行。
- 只在 vendor 域搜索，必要时扩展 system 域；绝不在 stub 域搜索绑定点。
- 搜索形式：`GetOrNewObject(<impl>, <interface>::GetDescriptor())`、`new <StubClass>(<impl>)`、`std::make_shared<StubClass>(<impl>)`、`SetImpl(<impl>)`、`impl_ = <impl>`、`host->stub = ...`。
- 沿 `<impl>` 表达式回溯：变量声明与初始化、`new XxxService()`、工厂/单例返回、强制转换后的真实对象。
- 若绑定点提供接口描述符，用其校验候选。
- 搜索 Bind/Init/OnStart/Register/动态注册；维护 visited（键为 `(文件, 函数签名行)`）防环。
- 唯一候选 → `strategy = "binding"` 或 `"registration"`；仍并列 → `impl_ambiguous`；无候选 → `impl_not_found`。

**6.4 约束与失败处理**

- impl 类必须是 vendor 域内的完整类定义；前置声明不算。
- stub 域、system 域中的类不能作为 impl 类。
- 同一 stub 类被多个入口引用时，按 `(stub 类限定名, 定义文件, 定义行)` 缓存复用。
- 单个入口失败：记入 `02_impl.json` 的 `unresolved`，`status = "unresolved"`，继续其他入口。
- 所有入口 `status` 均非 `resolved`（含全部 `impl_ambiguous`）→ `error.json`（`phase=resolve_impl`，`code=impl_not_found`），停止。
- 至少一个入口成功 → 继续阶段三；失败入口不进入最终 JSON。

**6.5 02_impl.json**

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
- `impl_file` 为 impl 类定义文件。
- `stub_called_functions` 只包含 OnRemoteRequest 中实际调用的 stub 成员函数。
- 写入失败 → `error.json`（`phase=resolve_impl`，`code=snapshot_write_failed`），停止。

- [ ] **Step 3: 验证第六章**

Run:

```bash
for t in "S1 特征匹配" "S2 命名启发式" "S3 绑定点" "feature_match" "naming" "binding" "registration" "impl_ambiguous" "impl_not_found"; do
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

Expected: 无 `missing ...` 行；`json_valid=ok`。

- [ ] **Step 4: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: add phase two impl resolution"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 成功或跳过提示。

---

## Task 6: 写第七章阶段三：cmdId 用例与 stub→impl 映射

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第七章）

- [ ] **Step 1: 检查缺失**

Run:

```bash
grep -q 'stub_interface_not_found' HDF-api-finder/SKILL.md && echo "case=present" || echo "case=missing"
grep -q '03_cases.json' HDF-api-finder/SKILL.md && echo "cases_file=present" || echo "cases_file=missing"
```

Expected: `case=missing`、`cases_file=present`（第四章已提及文件名）。

- [ ] **Step 2: 写第七章正文**

必须包含：

**7.1 预规划**

- 对每个已解析入口完整读取 OnRemoteRequest，枚举全部 cmdId 用例，写入 `03_cases.json`，状态 `pending`。
- switch/case：每个 case 标签一个用例；连续标签共享代码分别记录；fallthrough 归属所有落入标签；default 单独记录、不进最终 JSON。
- 嵌套 switch：外层 cmd 作为最终 cmd，内层条件写 evidence；无外层时内层成为最终 cmd。
- if/else if/else：每个能区分 cmdId 的条件分支一个用例；else 按 default 处理；复合条件、范围、掩码记录表达式原文。
- 其他形式（数组/map、函数指针表、宏生成 switch、注册表）：能静态枚举就分析；否则入口级记 `dispatch_form_unsupported`，继续其他用例。
- 转发：转给已知 dispatcher、SendRequest 或 OnRemoteRequest 时作为内层分发继续；无外层 cmd 时内层 case 成为最终 cmd，已有外层则保留；visited 防环。

**7.2 逐用例追踪**

- 只接受 stub 类成员函数调用：`Foo(...)`、`this->Foo(...)`、`StubClass::Foo(...)` 及可静态确定的等价成员调用。
- 排除 IPCObjectStub::OnRemoteRequest、基类框架调用、日志/断言/纯校验、构造/析构/运算符、free 函数、GetDescriptor。
- 0 个 stub 成员调用 → `stub_interface_not_found`；多个 → 每个分别形成映射。
- stub 接口函数定义必须在 stub 域内，且属于 stub 类或 stub 域内基类；否则 `stub_interface_scope`。
- 读取完整函数体找 impl 调用：`impl_->Foo(...)`、`this->impl_->Foo(...)`、局部别名 `auto p = impl_; p->Foo(...)`、其他可静态确定的等价形式。
- 无直接调用时，可沿 stub 域内本地 helper 继续追踪；visited 防环、不设深度上限；找到 impl 调用即停，不进入 impl 方法内部；helper 在 stub 域外 → `stub_interface_scope`。
- 仍无 impl 调用 → `impl_call_not_found`。
- impl 方法定义必须在 vendor 域：优先 `ImplClass::Foo`；找不到沿 vendor 域内基类查找，记录实际定义类与文件；仅在 stub/system/scope 外 → `impl_scope`；重载无法按实参唯一确定 → `impl_overload_ambiguous`。
- 一条 03_cases 用例记录对应一个 stub 接口函数；命中的全部 impl 方法写入 `impl_interfaces` 列表。若一个分支调用多个 stub 接口函数，拆成多条记录，共享 `case_site` / `cmd_id_macro` / `cmd_id_value`，`id` 递增。

**7.3 cmdId 取值解析**

| 形式 | 处理 |
|------|------|
| 整数字面量 | 直接取值 |
| `#define` 宏链 | scope 内递归解析，维护宏展开 visited 防环 |
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

**7.4 写穿与防遗忘**

- 预规划后立即写 `03_cases.json`，全部 `pending`。
- pending 用例的 stub/impl 字段可为 null；done 用例必须有完整 stub 接口函数与至少一个 impl 接口函数；unresolved 用例必须有 unresolved 记录。
- 每完成一个用例，立即更新 status（done/unresolved）、结果与证据并落盘。
- 每次更新先写 `03_cases.json.tmp`，Read 回读校验后 mv 覆盖。
- 处理下一个用例前核对清单，存在 pending 则继续，不得跳过。
- 全部处理完后不得存在 pending。
- 回读、mv 或写盘失败 → 删除残留 .tmp，写 `error.json`（`phase=trace_cases`，`code=snapshot_write_failed`），停止。
- 不做自动恢复；每次运行先备份旧输出，从零分析。

**7.5 03_cases.json**

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

**7.6 unresolved 类型表**

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

- [ ] **Step 3: 验证第七章**

Run:

```bash
for t in stub_interface_not_found stub_interface_scope impl_call_not_found impl_method_not_found impl_scope impl_overload_ambiguous cmd_value definition_not_found cycle; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing $t"
done
grep -q 'pending' HDF-api-finder/SKILL.md && echo "pending=present"
grep -q '03_cases.json.tmp' HDF-api-finder/SKILL.md && echo "atomic=present"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
for b in re.findall(r'```json\n(.*?)```', text, re.S):
    json.loads(b)
print("json_valid=ok")
PY
```

Expected: 无 `missing ...` 行；`pending=present`、`atomic=present`、`json_valid=ok`。

- [ ] **Step 4: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: add phase three case tracing"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 成功或跳过提示。

---

## Task 7: 写第八章输出与自检

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第八章）

- [ ] **Step 1: 检查缺失**

Run:

```bash
grep -q 'output_write_failed' HDF-api-finder/SKILL.md && echo "output=present" || echo "output=missing"
grep -q 'self_check_failed' HDF-api-finder/SKILL.md && echo "selfcheck=present" || echo "selfcheck=missing"
```

Expected: `output=missing`、`selfcheck=missing`。

- [ ] **Step 2: 写第八章正文**

必须包含：

**8.1 组装规则**

最终输出 `<out>/<service_name>_api.json`，按入口分组，只含已解析项：

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
7. default、unresolved 不输出。
8. UTF-8，JSON 缩进 2 空格。
9. 先写 `<out>/<service_name>_api.json.tmp`，Read 回读校验通过后 mv 为最终文件；最终文件必须存在，否则任务失败。

**8.2 自检**

1. 01_entry.json 中的全部入口都已处理。
2. 03_cases.json 中不存在 pending。
3. 每个最终入口至少有一条 case。
4. 每条最终 case 的 stub 接口函数定义在 stub 域且文件在 scope_stub_files.txt 内；impl 接口函数定义在 vendor 域且文件在 scope_vendor_files.txt 内。
5. 所有 stub/impl 函数都是完整定义，函数名为定义处作用域限定名；重载以 `(文件, 定义行)` 区分。
6. default 没有混入最终 JSON；unresolved 没有混入最终 JSON。
7. 最终 JSON 字段集合与 8.1 完全一致；已按六元组去重并排序。
8. 写临时文件后 Read 回读，核对数组长度、字段名、类型；截断视为失败。
9. mv 后双方法确认最终文件存在（ls + Read）。

失败处理：

- 自检第 1–7 项失败 → `error.json`（`phase=output`，`code=self_check_failed`），停止，不写最终 JSON。
- 写临时文件、回读、mv 或最终文件存在性确认失败 → 删除可能残留的 .tmp，写 `error.json`（`phase=output`，`code=output_write_failed`），停止。

**8.3 完成报告**

- scope 各域文件数与总数；
- service_name；
- 入口来源与数量；
- stub 类数量、impl 类数量；
- cmdId 用例数（预规划、不含 default）；
- 已解析映射数（最终 JSON case 总数）；
- unresolved 数量；
- 最终输出文件绝对路径。

- [ ] **Step 3: 验证第八章**

Run:

```bash
grep -q 'self_check_failed' HDF-api-finder/SKILL.md && echo "selfcheck=present"
grep -q 'output_write_failed' HDF-api-finder/SKILL.md && echo "output=present"
grep -q 'on_remote_request_line' HDF-api-finder/SKILL.md && echo "entry_line=present"
grep -q 'impl_interface_file' HDF-api-finder/SKILL.md && echo "impl_file=present"
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
for b in re.findall(r'```json\n(.*?)```', text, re.S):
    json.loads(b)
print("json_valid=ok")
PY
```

Expected: 4 个 `present`、`json_valid=ok`。

- [ ] **Step 4: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: add output and self-check"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 成功或跳过提示。

---

## Task 8: 写第九章至第十二章

**Files:**
- Modify: `HDF-api-finder/SKILL.md`（第九章、第十章、第十一章、第十二章）

- [ ] **Step 1: 检查缺失**

Run:

```bash
for h in '## 九、权限申请' '## 十、约束规则' '## 十一、抗理性化检查' '## 十二、使用示例'; do
  grep -q "$h" HDF-api-finder/SKILL.md || echo "missing $h"
done
grep -q 'invalid_conf' HDF-api-finder/SKILL.md && echo "conf=present" || echo "conf=missing"
```

Expected: 四个标题已存在（Task 1 写入）；`conf=present`（Task 3 写入）。

- [ ] **Step 2: 写第九章“权限申请”**

写入：

| 工具 | 用途 |
|------|------|
| `Bash` | find/grep/xargs/ast-grep/ls/mkdir/mv/rm；scope 内搜索、备份/清理输出目录 |
| `Read` | 读取源码、conf、macro.json、阶段快照、回读临时 JSON |
| `Write` | 写入 scope 清单、01_entry.json、02_impl.json、03_cases.json、error.json、最终 JSON（含临时文件） |
| `ast-grep`（可选） | AST 模式搜索；未安装时用 grep 代替，不中断 |

- 权限表只列上述工具；不得出现任何代码图/MCP 代码探索工具。

- [ ] **Step 3: 写第十章“约束规则”**

写入 15 条约束：

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

**error.json 格式：**

```json
{
  "phase": "resolve_impl",
  "code": "impl_not_found",
  "message": "所有入口都无法定位 impl 类",
  "evidence": "已尝试 S1 特征匹配、S2 命名启发式、S3 绑定点与注册流程；候选文件列表见 02_impl.json"
}
```

- `phase` 取值：`init` / `locate_entry` / `resolve_impl` / `trace_cases` / `output`。
- `error.json` 只在新 `<out>` 创建后写入；备份/mv/mkdir 失败只在对话中报告；`error.json` 写入失败改为对话报告，不再重试。

**全局错误码表：**

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

**术语表：**

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
| unresolved | 无法按规则确定的分析项，记录原因与已尝试方法，不臆造 |

- [ ] **Step 4: 写第十一章“抗理性化检查”**

写入表格：

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

- [ ] **Step 5: 写第十二章“使用示例”**

写入两个示例，全部使用中性 demo 命名：

示例一：首次分析

```text
/HDF-api-finder /srv/demo/project /srv/demo/conf.json
```

说明：初始化并备份输出目录 → 生成 scope 清单 → 阶段一定位 DemoInterfaceStub::OnRemoteRequest → 阶段二定位 DemoService → 阶段三枚举 CMD_DEMO_GET_VERSION 等用例并追踪 impl_ 调用 → 输出 demo_service_api.json → 报告计数。

示例二：指定 entry

```text
/HDF-api-finder /srv/demo/project /srv/demo/conf.json
```

conf 中包含 `entry` 为 `/abs/stub/demo_interface_stub.cpp:45`；说明只分析该入口，并校验其为合法 OnRemoteRequest 定义。

- [ ] **Step 6: 验证第九章至第十二章**

Run:

```bash
for h in '## 九、权限申请' '## 十、约束规则' '## 十一、抗理性化检查' '## 十二、使用示例'; do
  grep -q "$h" HDF-api-finder/SKILL.md || echo "missing $h"
done
for c in usage_error invalid_project_dir conf_not_found invalid_conf output_dir_failed empty_scope entry_invalid entry_not_found impl_not_found snapshot_write_failed output_write_failed self_check_failed internal_error; do
  grep -q "$c" HDF-api-finder/SKILL.md || echo "missing $c"
done
grep -q '术语表' HDF-api-finder/SKILL.md && echo "terms=present"
grep -q '抗理性化' HDF-api-finder/SKILL.md && echo "rationalization=present"
grep -q '/HDF-api-finder /srv/demo/project' HDF-api-finder/SKILL.md && echo "examples=present"
```

Expected: 无 `missing ...` 行；`terms=present`、`rationalization=present`、`examples=present`。

- [ ] **Step 7: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: add permissions constraints and examples"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 成功或跳过提示。

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

Expected: 第一行 `12`；第二个命令输出 12 行，行号递增，标题顺序为一至十二。

- [ ] **Step 2: frontmatter 与编码**

Run:

```bash
head -5 HDF-api-finder/SKILL.md
file HDF-api-finder/SKILL.md
```

Expected: 前 3 行为 `---`、`name: HDF-api-finder`、`description: ...`；`file` 显示 UTF-8 Unicode text。

- [ ] **Step 3: JSON 代码块全部可解析**

Run:

```bash
python3 - <<'PY'
import re, json
text = open('HDF-api-finder/SKILL.md', encoding='utf-8').read()
blocks = re.findall(r'```json\n(.*?)```', text, re.S)
print(f"json_blocks={len(blocks)}")
for i, b in enumerate(blocks, 1):
    try:
        json.loads(b)
    except Exception as e:
        raise SystemExit(f"block {i}: INVALID: {e}")
print("json_valid=ok")
PY
```

Expected: `json_blocks=6` 或更多；`json_valid=ok`。

- [ ] **Step 4: 禁用字符串与占位符**

Run:

```bash
grep -nE 'codegraph|hal-service-api-finder|ar_service|ActivityRecognition|ActivityInterfaceStub|ArService|CMD_ACTIVITY_INTERFACE_GET_VERSION' HDF-api-finder/SKILL.md || echo "forbidden_strings=none"
grep -nE 'TBD|TODO|待定|FIXME|XXX' HDF-api-finder/SKILL.md || echo "placeholders=none"
```

Expected: `forbidden_strings=none`、`placeholders=none`。

- [ ] **Step 5: 关键文件与错误码覆盖**

Run:

```bash
for f in scope_files.txt scope_stub_files.txt scope_vendor_files.txt scope_system_files.txt scope_roots.json 01_entry.json 02_impl.json 03_cases.json error.json; do
  grep -q "$f" HDF-api-finder/SKILL.md || echo "missing $f"
done
for c in usage_error invalid_project_dir conf_not_found invalid_conf output_dir_failed empty_scope entry_invalid entry_not_found impl_not_found snapshot_write_failed output_write_failed self_check_failed internal_error; do
  grep -q "$c" HDF-api-finder/SKILL.md || echo "missing $c"
done
```

Expected: 无任何输出。

- [ ] **Step 6: 记录结构验证结果**

Run:

```bash
wc -l HDF-api-finder/SKILL.md
wc -c HDF-api-finder/SKILL.md
```

Expected: 记录行数与字节数，供完成报告使用；不设硬性上限。

---

## Task 10: 流程一致性验证

**Files:**
- Verify: `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 阶段顺序与输出文件**

Run:

```bash
grep -nE '^## (五|六|七|八)、' HDF-api-finder/SKILL.md
grep -n '01_entry.json\|02_impl.json\|03_cases.json\|<service_name>_api.json' HDF-api-finder/SKILL.md | head -40
```

Expected: 五个阶段/输出章节按五→六→七→八排列；中间文件与最终文件都被对应阶段引用。

- [ ] **Step 2: 状态流闭合**

Run:

```bash
grep -nE 'pending|done|unresolved' HDF-api-finder/SKILL.md | head -40
grep -q '不得存在 `pending`' HDF-api-finder/SKILL.md && echo "pending_closed=present"
```

Expected: 出现 pending → done/unresolved 状态说明；`pending_closed=present`。

- [ ] **Step 3: 最终 JSON 只含已解析项**

Run:

```bash
grep -q 'status = "done"' HDF-api-finder/SKILL.md && echo "done_filter=present"
grep -q 'unresolved 不进入最终 JSON' HDF-api-finder/SKILL.md && echo "unresolved_filter=present"
grep -q '六元组去重' HDF-api-finder/SKILL.md && echo "dedup=present"
grep -q 'cmd_id_value.*null 最后' HDF-api-finder/SKILL.md && echo "sort=present"
```

Expected: 4 个 `present`。

- [ ] **Step 4: 错误码覆盖终止路径**

Run:

```bash
for c in usage_error invalid_project_dir conf_not_found invalid_conf output_dir_failed empty_scope entry_invalid entry_not_found impl_not_found snapshot_write_failed output_write_failed self_check_failed internal_error; do
  grep -q "$c" HDF-api-finder/SKILL.md || echo "missing $c"
done
grep -q 'error.json' HDF-api-finder/SKILL.md && echo "error_file=present"
```

Expected: 无 `missing ...`；`error_file=present`。

- [ ] **Step 5: 约束与工具边界**

Run:

```bash
grep -q '不调用 sub-agent' HDF-api-finder/SKILL.md && echo "no_subagent=present"
grep -q '不做自动恢复' HDF-api-finder/SKILL.md && echo "no_recovery=present"
grep -q '禁止 `test -f`' HDF-api-finder/SKILL.md && echo "dualcheck=present"
grep -q 'scope 是硬边界' HDF-api-finder/SKILL.md && echo "scope_boundary=present"
grep -q 'S3 绝不在 stub 域搜索' HDF-api-finder/SKILL.md && echo "s3_boundary=present"
```

Expected: 5 个 `present`。

- [ ] **Step 6: 发现缺口时修复并重跑本任务**

如果任一检查输出 `missing`，回到对应任务补充内容，然后重新执行 Task 9 和 Task 10 的全部检查，直到无缺失。

---

## Task 11: 最终通读与交付确认

**Files:**
- Verify: `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 分段通读全文**

Run:

```bash
wc -l HDF-api-finder/SKILL.md
```

Expected: 记录行数。然后分 3 段 Read 全文（每段不超过 300 行），检查：

- 12 个章节齐全，顺序正确；
- 无 TBD/TODO/待定；
- 同一概念用词一致（stub 接口函数、impl 接口函数、cmdId 用例、unresolved、绑定点、注册流程）；
- 阶段一/二/三的输入、输出、失败处理衔接；
- 最终 JSON 字段集合与第八章一致；
- 示例全部为中性 demo 命名。

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
```

Expected: `12`、`json_valid=ok`、`forbidden_strings=none`。

- [ ] **Step 3: 交付报告**

报告以下内容：

- SKILL.md 绝对路径；
- 行数与字节数；
- 章节数（应为 12）；
- JSON 代码块数量；
- 禁用字符串检查结果；
- 结构验证与流程验证结果；
- 明确声明：仅验证流程正确性，未验证实际接口识别准确性；后者由用户在实际项目中手工验证。

- [ ] **Step 4: 条件提交**

Run:

```bash
if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  git add HDF-api-finder/SKILL.md
  git commit -m "docs: complete HDF-api-finder skill"
else
  echo "not a git repository; skip commit"
fi
```

Expected: 成功或跳过提示。

---

## Task 12: 计划自查

**Files:**
- Verify: `docs/superpowers/specs/2026-10-26-hdf-api-finder-design.md` 与 `HDF-api-finder/SKILL.md`

- [ ] **Step 1: 规格覆盖检查**

对照设计文档章节，确认每个规则都有落地位置：

| 设计文档 | SKILL.md 章节 |
|----------|----------------|
| §3 调用方式与 conf | 第二章、第三章 |
| §3.3 output 与 macro.json | 第三章 |
| §4 分析范围与搜索纪律 | 第三章 |
| §5 阶段一 | 第五章 |
| §6 阶段二 | 第六章 |
| §7 阶段三 | 第七章 |
| §8 最终输出与自检 | 第八章 |
| §9 中间结果 | 第四章、第七章 |
| §10 错误处理 | 第五章至第八章、第十章 |
| §11 约束规则 | 第一章、第十章 |
| §12.1 权限申请 | 第九章 |
| §12.2 抗理性化检查 | 第十一章 |
| §13 验证方式 | Task 9、Task 10、Task 11 |
| §14 不涉及 | 第一章、第十章 |

- [ ] **Step 2: 占位符扫描**

Run:

```bash
grep -nE 'TBD|TODO|待定|FIXME|XXX|similar to|类似 Task' HDF-api-finder/SKILL.md || echo "plan_placeholders=none"
```

Expected: `plan_placeholders=none`。

- [ ] **Step 3: 术语一致性检查**

Run:

```bash
for t in 'stub 接口函数' 'impl 接口函数' 'cmdId 用例' 'unresolved' '绑定点' '注册流程'; do
  grep -q "$t" HDF-api-finder/SKILL.md || echo "missing term: $t"
done
```

Expected: 无输出。

- [ ] **Step 4: 结果记录**

将 Task 9、Task 10、Task 11、Task 12 的结果写入最终报告；如果发现任何缺口，回到对应任务修复并重跑全部验证。

---

## 执行说明

- 本计划只创建一个交付文件 `HDF-api-finder/SKILL.md`；不要创建其他代码或测试文件。
- 所有验证命令都在项目根目录 `/home/admin/cc/wksp/siakam_security_skills/HDF_api_finder` 执行。
- 当前目录不是 git 仓库；提交步骤只在已有仓库时执行，不要主动 `git init`。
- 完成标准：Task 9、Task 10 的全部检查无缺失；Task 11 通读无矛盾；Task 12 规格覆盖完整。
