# Rigify Metarig Preset 导出机制详解

> 本文档说明 Rigify 如何从已有骨架生成 Preset（Python 脚本），涵盖支持范围、执行过程和代码级细节。

---

## 一、核心结论

| 场景 | 是否支持 | 说明 |
|------|---------|------|
| 从 **Metarig**（已设置 rigify_type 的骨架）导出 Preset | **支持** | 内置功能，一键导出 |
| 从 **任意骨架** 导出 Preset | **半自动** | 需先手动为骨骼设置 `rigify_type` 和参数，再导出 |
| 从 **已生成的 Rig**（ORG/MCH/DEF 骨骼）反向还原 Metarig | **不支持** | 生成是单向过程，无逆向代码 |

Rigify 的管线是**严格单向**的：

```
Metarig (rigify_type + rigify_parameters)
    │
    ▼  [generate_rig()]
Generated Rig (ORG-/MCH-/DEF- 前缀骨骼 + 约束 + 驱动器)
```

---

## 二、前置条件：什么是 Metarig

`is_metarig()` 的判断逻辑（`ui.py:1044`）：

```python
def is_metarig(obj):
    if not (obj and obj.data and obj.type == 'ARMATURE'):
        return False
    if 'rig_id' in obj.data:       # 已生成的 rig 会被排除
        return False
    for b in obj.pose.bones:
        if b.rigify_type != "":     # 至少一个骨骼设置了 rigify_type
            return True
    return False
```

一个骨架要成为 Metarig，必须满足：
1. 是 Armature 类型对象
2. 没有 `rig_id` 属性（即尚未被 Rigify 生成过）
3. 至少有一个 PoseBone 的 `rigify_type` 非空

---

## 三、导出操作器

Rigify 提供两个内置导出操作器：

### 3.1 EncodeMetarig — 完整导出

| 属性 | 值 |
|------|---|
| 操作器 ID | `armature.rigify_encode_metarig` |
| 菜单位置 | Rigify Dev Tools 面板 / 3D 视口头部 Rigify 菜单 |
| 前置条件 | Edit Mode + `is_metarig()` 返回 True |
| 输出目标 | Blender 文本数据块 `metarig.py` |

```python
# 调用方式（ui.py:1219）
text = write_metarig(obj, layers=True, func_name="create",
                     groups=True, widgets=True)
```

导出内容包括：骨骼结构、rigify_type、rigify_parameters、骨骼集合（layers）、颜色组、自定义控件（widgets）、约束、自定义属性。

### 3.2 EncodeMetarigSample — 精简导出

| 属性 | 值 |
|------|---|
| 操作器 ID | `armature.rigify_encode_metarig_sample` |
| 前置条件 | 同上 |
| 输出目标 | Blender 文本数据块 `metarig_sample.py` |

```python
# 调用方式（ui.py:1247）
text = write_metarig(obj, layers=False, func_name="create_sample")
```

导出内容仅含：骨骼结构、rigify_type、rigify_parameters。**不含**骨骼集合、颜色组、自定义控件。

---

## 四、write_metarig() 核心函数详解

位于 `utils/rig.py:474`，是整个导出机制的核心。

### 4.1 函数签名

```python
def write_metarig(obj, layers=False, func_name="create",
                  groups=False, widgets=False):
    """将 metarig 写为 Python 脚本"""
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `obj` | — | Armature 对象 |
| `layers` | `False` | 是否导出骨骼集合（Bone Collections） |
| `func_name` | `"create"` | 生成的函数名（`create` 或 `create_sample`） |
| `groups` | `False` | 是否导出 Rigify 颜色组 |
| `widgets` | `False` | 是否导出自定义控件生成代码 |

### 4.2 生成脚本的执行阶段

```
Phase 1:  EDIT 模式 — 创建 EditBones
  ├─ Phase 1a: 骨骼集合定义（仅 layers=True）
  └─ Phase 1b: 颜色组定义（仅 groups=True）

Phase 2:  创建 EditBones（按父级深度排序）
  — head, tail, roll, use_connect, inherit_scale
  — B-bone 属性（bbone_segments, bbone_easein/out 等）

Phase 3:  OBJECT 模式 — 设置 PoseBone 属性
  ├─ Phase 3a: rigify_type + 锁定属性 + 旋转模式
  ├─ Phase 3b: 骨骼集合分配（仅 layers=True）
  ├─ Phase 3c: rigify_parameters（try/except 容错）
  ├─ Phase 3d: 自定义属性（rna_idprop_ui_create）
  ├─ Phase 3e: 约束
  └─ Phase 3f: 自定义控件（仅 widgets=True）

Phase 4:  返回 EDIT 模式 — 选中骨骼、设置 bbone 宽度
```

### 4.3 各阶段导出的数据

#### Phase 2 — 骨骼几何与层级

```python
bone = arm.edit_bones.new('bone_name')
bone.head = x, y, z
bone.tail = x, y, z
bone.roll = angle
bone.use_connect = True/False
bone.parent = arm.edit_bones[bones['parent_name']]  # 引用已创建的父骨骼
# 非默认值才导出：
bone.inherit_scale = 'NONE'        # 默认 'FULL' 不导出
bone.bbone_segments = 4             # 默认 1 不导出
bone.bbone_easein = 0.5            # 默认 1.0 不导出
# ... 其他 bbone_* 属性同理
```

**排序策略**：按父级递归深度排序（`len(bone.parent_recursive)`），确保父骨骼始终在子骨骼之前创建。

#### Phase 3a — Rigify 核心属性

```python
pbone = obj.pose.bones[bones['bone_name']]
pbone.rigify_type = 'limbs.arm'     # ← 关键：绑定类型标识
pbone.lock_location = (False, False, False)
pbone.lock_rotation = (False, False, False)
pbone.lock_rotation_w = False
pbone.lock_scale = (False, False, False)
pbone.rotation_mode = 'QUATERNIUM'
```

#### Phase 3b — 骨骼集合分配

```python
assign_bone_collections(pbone, 'Face', 'Face (Primary)')
```

#### Phase 3c — Rigify 参数（最核心部分）

```python
try:
    pbone.rigify_parameters.extra_layers = True
except AttributeError:
    pass
try:
    pbone.rigify_parameters.primary_layer_arm = '...'
except AttributeError:
    pass
# 集合引用参数使用特殊函数：
assign_bone_collection_refs(pbone.rigify_parameters,
                            'primary_layer_arm', 'Arm.L (Tweak)')
```

**`try/except` 的意义**：`RigifyParameters` 是所有 rig 类型共享的 PropertyGroup，导出时遍历所有 key，部分 key 可能不属于当前骨骼的 `rigify_type`。`AttributeError` 表示该参数在当前 Rigify 版本中不存在（可能是版本差异或 feature set 未加载），安全跳过。

**集合引用参数的处理**：以 `_coll_refs` 结尾的列表属性（如 `primary_layer_arm_coll_refs`）不直接赋值，而是通过 `assign_bone_collection_refs()` 函数按名称查找集合对象并建立引用。

#### Phase 3d — 自定义属性

```python
rna_idprop_ui_create(
    pbone,
    'custom_prop',
    default=0.0,
    min=0.0, max=1.0,
    soft_min=0.0, soft_max=1.0,
    description='My custom property',
)
```

仅导出 `float` 和 `int` 类型的自定义属性，过滤掉 RNA 内建属性。

#### Phase 3e — 约束

```python
con = pbone.constraints.new('COPY_ROTATION')
con.name = 'Copy Rotation'
con.target = obj
# 所有非默认属性逐一导出...
```

#### Phase 3f — 自定义控件

```python
if 'widget_id' not in widget_map:
    widget_map['widget_id'] = create_widget_name_widget(
        obj, pbone.name, widget_name='WidgetName', widget_force_new=True)
pbone.custom_shape = widget_map['widget_id']
```

---

## 五、从任意骨架创建 Preset 的完整流程

### 5.1 手动方式（Blender GUI）

```
步骤 1: 准备骨架
    ├─ 打开目标 Armature
    ├─ 进入 Pose Mode
    └─ 为每个需要 Rigify 控制的骨骼设置 rigify_type
        （在 Bone Properties → Rigify Type 下拉框中选择）

步骤 2: 配置参数
    ├─ 选中每个骨骼，在 Rigify Parameters 面板中调整参数
    └─ 设置骨骼集合分配（Bone Collections → Rigify UI Row）

步骤 3: 导出
    ├─ 进入 Edit Mode
    ├─ 打开侧边栏 (N) → Rigify Dev Tools 面板
    ├─ 点击 "Encode Metarig" → 生成 metarig.py
    └─ 或点击 "Encode Metarig Sample" → 生成精简版 metarig_sample.py

步骤 4: 使用导出的脚本
    ├─ 在 Text Editor 中打开 metarig.py
    ├─ 可直接运行（脚本末尾有 __main__ 入口）
    └─ 或将脚本放到 metarigs/ 目录下，注册为菜单项
```

### 5.2 编程方式（Python 脚本）

```python
import bpy
from rigify.utils.rig import write_metarig

# 获取目标 Armature 对象
obj = bpy.context.active_object

# 确保是 metarig（至少一个骨骼有 rigify_type）
# 如果不是，需要先设置 rigify_type
for pbone in obj.pose.bones:
    if pbone.name == "upper_arm.L":
        pbone.rigify_type = "limbs.arm"
    elif pbone.name == "spine":
        pbone.rigify_type = "spines.basic_spine"
    # ... 为每个骨骼指定类型

# 导出完整 Preset
python_code = write_metarig(
    obj,
    layers=True,       # 包含骨骼集合
    func_name="create",
    groups=True,        # 包含颜色组
    widgets=True        # 包含自定义控件
)

# 写入文本数据块
text_block = bpy.data.texts.new("my_preset.py")
text_block.write(python_code)

# 或保存到文件
with open("/path/to/my_preset.py", "w") as f:
    f.write(python_code)
```

### 5.3 程序化设置 rigify_type 的策略

如果要从零开始为一个骨架自动设置 rigify_type，核心逻辑是**骨骼名称 → rig 类型映射**：

```python
import bpy

# 定义名称模式到 rigify_type 的映射
RIG_TYPE_MAP = {
    # 脊柱
    "spine": "spines.basic_spine",
    "spine.*": "spines.basic_spine",
    # 手臂
    "upper_arm.*": "limbs.arm",
    "forearm.*": "limbs.arm",
    # 腿部
    "thigh.*": "limbs.leg",
    "shin.*": "limbs.leg",
    # 手指
    "palm.*": "limbs.super_palm",
    "f_index.*": "limbs.super_finger",
    # 头部
    "head": "spines.super_head",
    # 通用
    "*": "basic.super_copy",
}

def assign_rigify_types(obj, mapping):
    """根据名称模式为骨骼批量设置 rigify_type"""
    import re
    for pbone in obj.pose.bones:
        for pattern, rig_type in mapping.items():
            if re.match(pattern.replace('*', '.*') + '$', pbone.name):
                pbone.rigify_type = rig_type
                break
```

---

## 六、将导出脚本注册为 Metarig 菜单项

导出的 Python 脚本可以直接放到 `metarigs/` 目录下作为新的 metarig 模板。

### 6.1 文件结构

```
rigify/
└── metarigs/
    ├── __init__.py          # 模块注册
    ├── human.py             # 内置人体 metarig
    └── my_custom.py         # ← 你的自定义 preset
```

### 6.2 脚本模板

导出的脚本必须包含 `create(obj)` 函数：

```python
import bpy
from rna_prop_ui import rna_idprop_ui_create
from mathutils import Color

def create(obj):
    # generated by rigify.utils.write_metarig
    bpy.ops.object.mode_set(mode='EDIT')
    arm = obj.data

    bones = {}
    # ... 骨骼创建代码 ...

    bpy.ops.object.mode_set(mode='OBJECT')

    # ... rigify_type 和参数设置代码 ...

    return bones

if __name__ == "__main__":
    create(bpy.context.active_object)
```

### 6.3 菜单注册

`metarig_menu.py` 中的 `get_metarigs()` 会自动扫描 `metarigs/` 目录，发现模块后生成菜单项。对于自定义 metarig，放在正确的目录结构下即可自动出现在菜单中。

如果使用外部 Feature Set，可以将 metarig 脚本放在 feature set 的 `metarigs/` 目录下，通过 Rigify 的 Feature Set 系统加载。

---

## 七、参数序列化的内部机制

### 7.1 rigify_parameters 的存储结构

`RigifyParameters` 是一个**单例共享 PropertyGroup**，所有 rig 类型的参数都动态注册到同一个类上：

```python
# __init__.py:473
class RigifyParameters(bpy.types.PropertyGroup):
    name: StringProperty()
    # 其他参数在运行时通过 add_parameters() 动态注入

# __init__.py:671 — 注册时遍历所有 rig 类型
def register_rig_parameters():
    for rig_info in rig_lists.get_rigs().values():
        rig_info["module"].Rig.add_parameters(params)
```

每个 rig 类型的 `add_parameters()` 向 `RigifyParameters` 添加属性：

```python
# rigs/limbs/arm.py 示例
class Rig(BaseLimbRig):
    @classmethod
    def add_parameters(cls, params):
        params.extra_ik_toe = BoolProperty(name='Extra IK Toe', default=False)
        params.limb_type = EnumProperty(items=[('ARM', 'Arm', ''), ('LEG', 'Leg', '')])
        # ...
```

### 7.2 序列化路径

**路径 A：write_metarig() 内置方式**（推荐）

```python
# utils/rig.py:634-652
for param_name in rigify_parameters.keys():
    param = _get_property_value(rigify_parameters, param_name)
    if isinstance(param, bpy_prop_collection):
        # 集合引用参数特殊处理
        if layers and param_name.endswith(REFS_LIST_SUFFIX):
            # 生成 assign_bone_collection_refs() 调用
        continue
    if param is not None:
        code.append("    try:")
        code += _format_property_value(
            f"        pbone.rigify_parameters.{param_name} = ", param)
        code.append("    except AttributeError:")
        code.append("        pass")
```

**路径 B：propgroup_to_dict() 方式**（5.1 新增，用于程序化处理）

```python
from rigify.utils.misc import propgroup_to_dict, assign_rna_properties

# 序列化
param_dict = propgroup_to_dict(pbone.rigify_parameters)
# → {'extra_ik_toe': True, 'limb_type': 'ARM', ...}

# 反序列化
assign_rna_properties(target_pbone.rigify_parameters, param_dict)
```

两条路径的区别：

| 特性 | write_metarig() | propgroup_to_dict() |
|------|----------------|---------------------|
| 输出格式 | Python 源代码字符串 | Python 字典对象 |
| 用途 | 生成可执行的 .py 文件 | 程序间传递参数数据 |
| 容错 | try/except 处理未知参数 | 跳过只读属性 |
| 集合引用 | 生成 `assign_bone_collection_refs()` 调用 | 递归转为列表/字典 |

---

## 八、局限性与注意事项

### 8.1 无法从已生成的 Rig 反向还原

已生成的 Rig（含有 `rig_id` 属性）被 `is_metarig()` 明确排除。原因：

1. **骨骼名称已变换**：原始骨骼名被加 `ORG-` 前缀，新增了 `MCH-` 和 `DEF-` 骨骼
2. **层级结构已重组**：生成过程添加了大量机构骨骼和变形骨骼
3. **参数已展开**：rigify_parameters 中存储的是输入参数，不是生成后的约束/驱动器配置
4. **无可逆映射**：没有代码记录"哪根 ORG 骨骼对应哪根 metarig 骨骼"的映射关系

### 8.2 rigify_type 必须手动设置

Rigify 无法自动推断骨骼的 rigify_type。每个骨骼的绑定类型必须由用户指定——这是导出 Preset 的唯一前置门槛。

### 8.3 参数版本兼容性

导出脚本中的 `rigify_parameters` 使用 `try/except` 包装，因为：
- 不同 Rigify 版本的参数名可能不同
- 外部 Feature Set 可能未加载
- 参数类型可能变更

运行导出脚本时，如果参数不存在会被静默跳过。

### 8.4 控件（Widget）导出需要依赖 Rigify

当 `widgets=True` 时，导出的脚本依赖 `rigify.utils.widgets.widget_generator` 装饰器。如果要在不加载 Rigify 的环境下运行导出脚本，应使用 `widgets=False` 导出并自行处理控件。

---

## 九、关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| `write_metarig()` 核心序列化函数 | `utils/rig.py` | 474-747 |
| `write_metarig_widgets()` 控件序列化 | `utils/rig.py` | 445-471 |
| `write_widget()` 单个控件序列化 | `utils/widgets.py` | 493+ |
| `EncodeMetarig` 操作器 | `ui.py` | 1199-1224 |
| `EncodeMetarigSample` 操作器 | `ui.py` | 1227-1253 |
| `is_metarig()` 判断函数 | `ui.py` | 1044-1052 |
| `propgroup_to_dict()` 参数序列化 | `utils/misc.py` | 275-325 |
| `assign_rna_properties()` 参数反序列化 | `utils/misc.py` | 344-427 |
| `RigifyParameters` PropertyGroup | `__init__.py` | 473 |
| `register_rig_parameters()` 参数注册 | `__init__.py` | 671-683 |
| `Sample` 操作器（添加 rig 类型样本） | `ui.py` | 1150+ |
| `generate_rig()` 生成入口 | `generate.py` | 692+ |
