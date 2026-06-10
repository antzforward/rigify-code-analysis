# Rigify 核心变更分析：Blender 4.5 → 5.1

> 本文档对比分析 `rigify_4_5` 与 `rigify_5_1` 两个版本的源码差异，聚焦核心功能变化与技术变革。

---

## 一、技术变革总览

两个版本之间最大的驱动力是 **Blender 5.x 引入的 Slotted Actions（分层动作槽）系统**。这是 Blender 动画底层架构的一次重大重构，直接影响了 Rigify 中所有与 Action/F-Curve 交互的代码。

| 变革领域 | 影响范围 | 严重程度 |
|---------|---------|---------|
| Slotted Actions API | 6+ 文件，核心动画管线 | **重大技术变革** |
| `bone.select`/`bone.hide` 提升到 Pose 层级 | 5+ 文件 | API 适配 |
| Usetime Properties 独立注册 | `__init__.py`, `rig_ui_template.py` | 架构变革 |
| PropertyGroup 序列化/反序列化 | `utils/misc.py`, `copy_mirror_parameters.py` | 功能增强 |
| 动画数据完整性修复 | `generate.py` | Bug 修复 |
| Library Override 防护 | `base_generate.py` | 安全增强 |

---

## 二、重大技术变革：Slotted Actions 系统

### 2.1 背景

Blender 4.x 中，Action 是一个扁平结构：

```
Action → F-Curves (直接包含)
```

Blender 5.x 中，Action 变为分层结构：

```
Action → Layers → Strips → Channelbags → F-Curves
```

同时引入了 **ActionSlot** 概念：一个 Action 可以包含多个 Slot，每个 Slot 对应不同的对象/数据块，F-Curve 存储在 Slot 对应的 Channelbag 中。

### 2.2 API 变化对照表

| 4.x API | 5.x API | 说明 |
|---------|---------|------|
| `action.fcurves` | `channelbag.fcurves` | F-Curve 必须通过 Channelbag 访问 |
| `action.fcurves.new(path, action_group=g)` | `channelbag.fcurves.ensure(path, group_name=g)` | 创建/获取 F-Curve 的新方式 |
| `action.fcurves.find(path)` | `channelbag.fcurves.find(path)` | 查找范围缩小到 Channelbag |
| `find_action(obj)` | `obj.animation_data.action` + `obj.animation_data.action_slot` | 需要显式获取 Slot |
| N/A | `anim_utils.action_get_channelbag_for_slot(action, slot)` | 新增：获取 Slot 对应的 Channelbag |
| N/A | `anim_utils.action_ensure_channelbag_for_slot(action, slot)` | 新增：确保 Channelbag 存在 |
| `con.action = action` | `con.action = action` + `con.action_slot = slot` | ACTION 约束需要指定 Slot |

### 2.3 受影响文件及具体变化

#### `utils/action_layers.py` — 动作层核心（7处功能变更）

- **触发器引用从 Action 变为 ActionSlot**：`trigger_action_a/b: Action` → `trigger_a/b: ActionSlotBase`
- **新增 `action_slot` 属性**：包含完整的 Slot 选择系统（UI、unique_id、自动选择回调）
- **`keyed_bone_names` 属性**：必须先获取 channelbag，再遍历 fcurves
- **ACTION 约束**：必须设置 `con.action_slot`
- **排序逻辑**：`sort_slots()` → `sort_action_setups()`，按 `unique_id` 排序而非 action name
- **移除重复 Action 名称检查**：同一 Action 可有多个 Slot，不再冲突
- **新增 `versioning_5_0()` 函数**：`@bpy.app.handlers.persistent` 装饰的 `load_post` 处理器，自动迁移旧文件的 Action 引用到新 Slot 格式

#### `utils/animation.py` — 动画工具（6处功能变更）

- **新增 `_get_channelbag_for_rig()` 辅助函数**：封装 `action + slot → channelbag` 的两步查找
- **`get_keyed_frames_in_range()`**：改用 channelbag
- **`bones_in_frame()`**：改用 channelbag
- **`overwrite_prop_animation()`**：改用 channelbag
- **`clean_action_empty_curves()`**：需遍历 `action.layers > strips > channelbags > fcurves` 完整层级
- **`ActionCurveTable` 类**：构造参数从 `action` 改为 `rig: Object`，内部使用 channelbag

#### `rigs/limbs/limb_rigs.py` — 肢体绑定

- `make_property` 函数中创建 F-Curve 的逻辑完全重写为 Slotted Actions API
- 新增 `anim_utils` 导入和 channelbag 获取逻辑

#### `operators/action_layers.py` — 操作器（大幅重构）

- 新增完整的 Slot 选择 UI 系统
- 触发器从 `PointerProperty(type=Action)` 改为基于 `unique_id` 的字符串引用
- 活跃 Slot 切换从整数索引改为 `unique_id` 引用（更稳定）
- 移除 `layout_type` 检查（Blender 5.x 移除了 GRID 布局类型）

#### `rot_mode.py` — 旋转模式转换器（完全重写）

- 所有函数签名从 `action` 参数改为 `channelbag` 参数
- 移除 `get_or_create_fcurve()` 辅助函数，改用 `channelbag.fcurves.ensure()`
- 操作器需确定正确的 Action Slot 并获取 Channelbag

#### `rig_ui_template.py` — 生成脚手架脚本

- `clear_animation()` 从操作 `action.fcurves` 改为操作 `channelbag.fcurves`
- 生成脚本中注入 channelbag 获取逻辑

#### `ui.py` — UI 面板

- 同样的 Slotted Actions 适配模式

---

## 三、API 适配：`bone.select` / `bone.hide` 提升到 Pose 层级

### 3.1 变化说明

Blender 4.x 中，`select` 和 `hide` 属性位于 `PoseBone.bone`（数据级 `Bone`）上：

```python
# 4.x
pose_bone.bone.select = True
pose_bone.bone.hide = True
```

Blender 5.x 中，这些属性直接提升到 `PoseBone` 上：

```python
# 5.x
pose_bone.select = True
pose_bone.hide = True
```

### 3.2 受影响文件

| 文件 | 变更数量 |
|------|---------|
| `rig_ui_template.py` | ~30+ 处 |
| `ui.py` | ~30+ 处 |
| `rigs/limbs/limb_rigs.py` | 驱动目标路径变更 |
| `rigs/limbs/spline_tentacle.py` | 驱动目标路径变更 |
| `operators/copy_mirror_parameters.py` | 选择逻辑变更 |

---

## 四、架构变革：Usetime Properties 独立注册

### 4.1 问题

Blender 4.x 中，Rigify 假定生成绑定后插件始终启用。如果禁用 Rigify 插件，所有 RNA 属性（包括 `rigify_ui_row`、`rigify_ui_title`）都会被注销，导致生成的绑定的 UI 脚本无法正常工作。

### 4.2 解决方案

5.1 将属性注册拆分为两个阶段：

```
register()
├── 子模块注册
├── 类注册
├── register_usetime_properties()    ← 新增：即使插件禁用也保留
├── register_rna_properties()        ← 新增：仅插件启用时存在
├── feature_sets 注册
└── rig_parameters 注册
```

- **Usetime Properties**（`rigify_ui_row`、`rigify_ui_title`）：生成绑定后始终可用，即使 Rigify 插件被禁用
- **RNA Properties**（`rigify_type`、`rigify_parameters` 等）：仅 Rigify 插件启用时存在

### 4.3 生成脚本的适配

`rig_ui_template.py` 新增 `_write_rna_prop_register_funcs()` 方法，将 `register_usetime_properties()` 和 `unregister_usetime_properties()` 的源代码注入到生成的绑定脚本中，使生成的绑定完全自包含。

### 4.4 意义

这标志着 Rigify 设计理念的转变：**生成的绑定从"依赖 Rigify 插件的附属品"变为"可独立运行的工件"**。

---

## 五、功能增强：PropertyGroup 序列化/反序列化

### 5.1 新增函数（`utils/misc.py`）

#### `propgroup_to_dict(propgroup) → dict`

将 `bpy.types.PropertyGroup` 转换为 Python 字典，支持递归处理嵌套的 COLLECTION 和 POINTER 属性。使用 Python `match`/`case` 语法按属性类型分派处理。

#### `assign_rna_properties(target, source)`

`propgroup_to_dict()` 的逆操作，将属性值从源（PropertyGroup 或 dict）赋值到目标 PropertyGroup。自动跳过只读属性，递归处理集合和指针。

### 5.2 应用场景

`operators/copy_mirror_parameters.py` 的参数复制逻辑从旧的脆弱方式：

```python
# 4.x（脆弱：需先删除再赋值，仅支持简单类型）
del to_bone['rigify_parameters']
to_bone['rigify_parameters'] = property_to_python(from_bone.rigify_parameters)
```

改为新的健壮方式：

```python
# 5.x（类型安全，支持所有 RNA 属性类型）
assign_rna_properties(to_bone.rigify_parameters, from_bone.rigify_parameters)
```

---

## 六、Bug 修复与安全增强

### 6.1 动画数据完整性修复（`generate.py`）

**问题**：`__rename_org_bones()` 将骨骼从原名重命名为 ORG- 前缀名时，Blender 会自动重命名 F-Curve 的数据路径，导致动画数据损坏——F-Curve 指向重命名后的 ORG 骨骼而非原始骨骼名。

**修复**：在重命名前清除 Action：

```python
# 5.1 新增
if obj.animation_data:
    obj.animation_data.action = None
```

### 6.2 Library Override 防护（`base_generate.py`）

**新增检查**：禁止在 Library Override Collection 中生成绑定：

```python
if self.collection.override_library:
    raise RuntimeError(
        "Cannot generate rig into a library override collection. "
        "Select a different collection in the outliner")
```

### 6.3 `find_collection` 早期返回（`__init__.py`）

当 `uid < 0`（未设置集合引用）时直接返回 None，避免不必要的查找。

---

## 七、其他变更

### 7.1 国际化（i18n）

- 新增 `pgettext_rpt`（报告上下文翻译函数），用于控制台/报告消息
- 4.x 中所有此类消息使用 `pgettext_iface`（UI 上下文），5.x 按语境区分

### 7.2 元骨骼模板微调（`metarigs/human.py`）

4 处亚毫米级坐标修正，修复默认人体模板中腿部/髋部区域骨骼连接的微小间隙。

### 7.3 操作器改进

- `bl_options` 新增 `UNDO` 标志，使操作可撤销
- 错误/无操作时返回 `CANCELLED` 而非 `FINISHED`（语义更准确）
- `poll` 类方法参数从 `self` 改为 `cls`

### 7.4 代码风格

- 多行导入从反斜杠续行改为括号化格式（PEP 8）
- `assert(condition)` → `assert condition`（assert 是语句不是函数）
- `not(x in y)` → `x not in y`（惯用写法）
- 运算符周围空格规范化

---

## 八、未变更的核心模块

以下模块在两个版本间**完全相同**，说明 Rigify 的核心绑定生成架构是稳定的：

| 模块 | 说明 |
|------|------|
| `base_rig.py` | 绑定基类、阶段回调系统、骨骼所有权模型 |
| `utils/bones.py` | 骨骼创建与操作工具 |
| `utils/mechanism.py` | 约束与驱动器创建工具 |
| `utils/objects.py` | ArtifactManager |
| `utils/switch_parent.py` | 父级切换工具 |
| `utils/widgets_basic.py` | 基础控件生成 |
| `utils/widgets_special.py` | 特殊控件生成 |
| `utils/naming.py` | 命名约定（仅 assert 风格） |
| `utils/rig.py` | 绑定工具（仅空格风格） |
| 所有 `rigs/skin/` | 皮肤绑定系统 |
| 所有 `rigs/spines/` | 脊柱绑定系统 |
| 所有 `rigs/basic/` | 基础绑定系统 |
| 所有 `rigs/face/` | 面部绑定系统 |

---

## 九、变更文件清单

| 文件 | 变更类型 | 变更级别 |
|------|---------|---------|
| `__init__.py` | Usetime 属性拆分 + i18n + 版本迁移 | **架构** |
| `base_generate.py` | Library Override 防护 + 代码风格 | 安全增强 |
| `generate.py` | 动画数据完整性修复 + 代码风格 | Bug 修复 |
| `utils/action_layers.py` | Slotted Actions 全面适配 + 版本迁移 | **重大** |
| `utils/animation.py` | Slotted Actions 全面适配 | **重大** |
| `utils/misc.py` | PropertyGroup 序列化/反序列化 | 功能增强 |
| `rigs/limbs/limb_rigs.py` | Slotted Actions + bone.hide 适配 | API 适配 |
| `rigs/limbs/spline_tentacle.py` | bone.hide 适配 | API 适配 |
| `rigs/faces/super_face.py` | 代码风格 | 仅风格 |
| `rigs/experimental/super_chain.py` | 代码风格 | 仅风格 |
| `operators/action_layers.py` | Slotted Actions UI 重构 | **重大** |
| `operators/copy_mirror_parameters.py` | 参数复制重构 + bone.select 适配 | 功能增强 |
| `rig_ui_template.py` | Slotted Actions + bone.select + Usetime 注入 | **重大** |
| `rot_mode.py` | Slotted Actions 完全重写 | **重大** |
| `ui.py` | Slotted Actions + bone.select + i18n | API 适配 |
| `metarigs/human.py` | 坐标微调 | 微调 |
