# P4Runtime helper.py 代码详解

> 文件：`tutorials/utils/p4runtime_lib/helper.py`（208 行）
>
> 一句话定位：**helper.py 是"名字 ↔ ID"的翻译层 + 表项的组装工厂。**
> 它把"人类读得懂的名字"（`MyIngress.ipv4_lpm`、`hdr.ipv4.dstAddr`、`dstAddr`）翻译成
> "P4Runtime 协议里用的数字 ID 和二进制值"，组装成一条完整的 `TableEntry` protobuf 消息，
> 交给 switch.py 发出去。

---

## 1. 它解决什么问题（先看动机）

P4Runtime 协议规定：**网络上传送的每条消息都只用数字 ID 和二进制数据，不用字符串名字**。
原因：数字小、省带宽、解析快。

但人在写代码时当然想用名字。于是就需要一个"字典"：

```
人的说法（字符串）                          P4Runtime 协议（数字/二进制）
─────────────────────      ─────────────────────────────────────
'MyIngress.ipv4_lpm'   →   table_id = 37375156
'hdr.ipv4.dstAddr'     →   field_id = 1, bitwidth = 32, type = LPM
'10.0.1.1'             →   value = \x0a\x00\x01\x01
32                     →   prefix_len = 32
'MyIngress.ipv4_forward' → action_id = 28792405
'dstAddr'              →   param_id = 1
'08:00:00:00:02:22'    →   value = \x08\x00\x00\x00\x02\x22
```

**这份"字典"哪来的？** 就是编译 basic.p4 时生成的 `build/basic.p4info.txtpb`。
helper.py 的 `__init__` 只干一件事：把这个文件读进内存，供后面的查找方法使用。

---

## 2. 类结构总览

```
class P4InfoHelper
│
├─ __init__(p4_info_filepath)            ← 读 p4info 文件，存进 self.p4info
│
├─【通用查找】名字↔ID 翻译
│    get()                ← 核心查找，按名字或 ID 遍历找对象
│    get_id()  get_name()  get_alias()   ← 三个包装
│    __getattr__()        ← 魔法方法：自动生成 get_tables_id()/get_actions_id() 等
│
├─【匹配字段】表项的 key 部分
│    get_match_field()  get_match_field_id()  get_match_field_name()
│    get_match_field_pb()        ← 字符串 → FieldMatch protobuf（编码）
│    get_match_field_value()     ← FieldMatch protobuf → 字符串（解码）
│
├─【动作参数】表项的 action 部分
│    get_action_param()  get_action_param_id()  get_action_param_name()
│    get_action_param_pb()       ← 字符串 → Action.Param protobuf（编码）
│
├─【组装工厂】
│    buildTableEntry()            ← 把上面的零件拼成一条完整 TableEntry
│
└─【复制引擎（PRE）】
     buildMulticastGroupEntry()   ← 组播组
     buildCloneSessionEntry()     ← 镜像会话
```

---

## 3. 头部 import（第 4-10 行）

```python
 4: import re                                        # 正则表达式（__getattr__ 用）
 5:
 6: import google.protobuf.text_format               # protobuf 的文本解析
 7: from p4.config.v1 import p4info_pb2              # p4info 的 protobuf 定义
 8: from p4.v1 import p4runtime_pb2                  # P4Runtime 消息的 protobuf 定义
 9:
10: from .convert import encode                       # 把字符串/数字编码成字节
```

| import | 类型 | 用途 |
|---|---|---|
| `google.protobuf.text_format` | 第三方库 | 把 `.txtpb` 文本文件解析成 protobuf 对象 |
| `p4info_pb2` | 第三方库（p4lang 的 proto） | 定义 `P4Info`、`Table`、`Action` 等"契约"对象 |
| `p4runtime_pb2` | 第三方库（p4lang 的 proto） | 定义 `TableEntry`、`FieldMatch`、`Action.Param` 等"运行时消息"对象 |
| `convert.encode` | 教程自写（同目录 convert.py） | 把 `'10.0.1.1'`、`'08:...:22'`、数字转成二进制字节 |

注意区分两类 protobuf 对象（这是理解 helper.py 的关键）：

| | p4info_pb2（契约） | p4runtime_pb2（消息） |
|---|---|---|
| 描述什么 | "P4 程序长什么样"（静态） | "发给交换机的请求长什么样"（动态） |
| 什么时候生成 | 编译时（p4c 产出的 txtpb） | 运行时（helper.py 现场组装） |
| 例子 | `tables`、`match_fields`、`params` 的**定义** | `TableEntry`、`FieldMatch`、`Action.Param` 的**实例** |

---

## 4. `__init__`：读 p4info 文件（第 14-20 行）

```python
14:     def __init__(self, p4_info_filepath):
15:         p4info = p4info_pb2.P4Info()              # ① 建一个空的 P4Info 对象
16:         # Load the p4info file into a skeleton P4Info object
17:         with open(p4_info_filepath) as p4info_f:  # ② 打开 txtpb 文件
18:             google.protobuf.text_format.Merge(p4info_f.read(), p4info,
19:                                               allow_unknown_field=True)
20:         self.p4info = p4info                      # ③ 存起来，供全类使用
```

- **第 15 行**：创建一个"骨架"对象 `P4Info`，它内部有 `tables`、`actions` 这些"列表"字段（目前是空的）。
- **第 17-19 行**：`text_format.Merge` 把**文本**格式的 `basic.p4info.txtpb` 内容"灌"进那个骨架对象，灌完后 `self.p4info.tables` 里就有一条条表的定义。
  - `allow_unknown_field=True`：p4info 文件里可能有这个版本 protobuf 不认识的新字段，忽略它们而不是报错（向前兼容）。
- **第 20 行**：存到 `self.p4info`。后面所有 `self.p4info.xxx` 都是读这份数据。

读完之后 `self.p4info` 内部长这样（对应 basic.p4 编译出的 p4info）：

```
P4Info
├─ tables: [ { preamble{id:37375156, name:"MyIngress.ipv4_lpm"}, match_fields[{id:1,
│             name:"hdr.ipv4.dstAddr", bitwidth:32, match_type:LPM}], ... } ]
└─ actions:[ { preamble{id:28792405, name:"MyIngress.ipv4_forward"},
               params[{id:1,name:"dstAddr",bitwidth:48}, {id:2,name:"port",bitwidth:9}] },
              { preamble{id:25652968, name:"MyIngress.drop"} },
              { preamble{id:21257015, name:"NoAction"} } ]
```

---

## 5. 通用查找组：`get` / `get_id` / `get_name` / `get_alias`

### 5.1 `get`：核心查找方法（第 22-38 行）

```
P4Info (根对象)
├── tables[]              ← 表定义列表
│   └── table[0]
│       ├── preamble.id = 37375156
│       ├── match_fields = [         ← 直接内嵌字段定义（强连接）
│       │   { id:1, name:"dstAddr", bitwidth:32, match_type:LPM }
│       │ ]
│       └── action_ids = [           ← ID 引用（弱连接）
│           28792405,   ← 指向 actions[0]
│           25652968    ← 指向 actions[1]
│       ]
│
└── actions[]             ← 动作定义列表（独立存储）
    ├── actions[0].preamble.id = 28792405
    │   └── params = [ {id:1, name:"dstAddr", bitwidth:48} ]
    └── actions[1].preamble.id = 25652968
        └── params = []
```

```json
{
  "tables": [
    {
      "preamble": {
        "id": 37375156,
        "name": "MyIngress.ipv4_lpm"
      },
      "match_fields": [
        {
          "id": 1,
          "name": "hdr.ipv4.dstAddr",
          "bitwidth": 32,
          "match_type": "LPM"
        }
      ],
      "action_ids": [28792405, 25652968, 21257015],  // ← 外键引用
      "size": 1024
    }
  ],
  "actions": [
    {
      "preamble": { "id": 28792405, "name": "MyIngress.ipv4_forward" },
      "params": [
        { "id": 1, "name": "dstAddr", "bitwidth": 48 },
        { "id": 2, "name": "port", "bitwidth": 9 }
      ]
    },
    {
      "preamble": { "id": 25652968, "name": "MyIngress.drop" },
      "params": []
    },
    {
      "preamble": { "id": 21257015, "name": "NoAction" },
      "params": []
    }
  ]
}
```



```python
22:     def get(self, entity_type, name=None, id=None):
23:         if name is not None and id is not None:
24:             raise AssertionError("name or id must be None")
25:
26:         for o in getattr(self.p4info, entity_type):   # 遍历 self.p4info 里叫 entity_type 的列表，返回值是一整个列表
27:             pre = o.preamble                          # 每个对象都带 preamble（名字+ID 都在这）
28:             if name:
29:                 if (pre.name == name or pre.alias == name):  # 按名字（或别名）匹配
30:                     return o
31:             else:
32:                 if pre.id == id:                      # 按 ID 匹配
33:                     return o
34:
35:         if name:
36:             raise AttributeError("Could not find %r of type %s" % (name, entity_type))
37:         else:
38:             raise AttributeError("Could not find id %r of type %s" % (id, entity_type))
```

逐行：
- **第 23-24 行**：`name` 和 `id` 只能给一个（防止歧义）。
- **第 26 行**：`getattr(self.p4info, entity_type)` 就是 `self.p4info.tables` 或 `self.p4info.actions`（`entity_type` 是字符串 "tables"/"actions"，用 getattr 动态取属性）。
- **第 27 行**：protobuf 里每个实体都带一个 `preamble` 字段，里面统一放 `name`、`alias`、`id`。所以不管查表还是查动作，套路一样。
- **第 29 行**：匹配 `name` **或** `alias`。p4info 里两者都有：`name="MyIngress.ipv4_lpm"`（带作用域全名）、`alias="ipv4_lpm"`（短名）。所以两种写法都能查到。
- **第 35-38 行**：找不到就抛 `AttributeError`，报错信息里带上要找的名字/ID，方便排错。



**一、`tables` 列表中的每个元素**

每个 `table` 对象描述了一张 P4 表的完整结构，包含以下核心字段：

**1. `preamble`（元数据头）**

每个 P4 对象（表、动作、计数器等）都有这个。

| 字段          | 示例值                 | 解释                                                         |
| :------------ | :--------------------- | :----------------------------------------------------------- |
| `id`          | `37375156`             | 全局唯一数字 ID，数据平面用这个 ID 而不是名字来标识表（性能考虑） |
| `name`        | `"MyIngress.ipv4_lpm"` | 完整路径名（<控制块>.<表名>），保证唯一性                    |
| `alias`       | 可选                   | 更短的别名，通常等于 `name` 去掉控制块前缀                   |
| `annotations` | `["@id(1)"]`           | P4 源码中的注解（如 `@id`、`@field` 等），用于额外元数据     |

**2. `match_fields`（匹配字段列表）**

描述这张表用哪些报文头字段做匹配。

每个匹配字段包含：

| 字段          | 示例值                                | 解释                                                       |
| :------------ | :------------------------------------ | :--------------------------------------------------------- |
| `id`          | `1`                                   | 字段在表内的 ID（从 1 开始），用于控制平面指定匹配哪个字段 |
| `name`        | `"hdr.ipv4.dstAddr"`                  | 字段的完整路径名，精确到哪个头部的哪个字段                 |
| `bitwidth`    | `32`                                  | 匹配字段的位宽（单位：bit）                                |
| `match_type`  | `LPM` / `EXACT` / `TERNARY` / `RANGE` | 匹配类型，决定控制平面下发规则时的匹配语义                 |
| `annotations` | `["@id(1)"]`                          | 字段级别的注解                                             |

**`match_type` 详解**：

| 类型                  | 含义                         | 控制平面下发的值                                      |
| :-------------------- | :--------------------------- | :---------------------------------------------------- |
| `EXACT`               | 完全匹配                     | 必须提供一个精确值                                    |
| `LPM`（最长前缀匹配） | 用于 IP 路由，匹配最长的前缀 | 提供 `(value, prefix_length)`，如 `(192.168.1.0, 24)` |
| `TERNARY`             | 三态匹配，支持掩码           | 提供 `(value, mask)`，mask 中 0 表示“忽略该位”        |
| `RANGE`               | 范围匹配                     | 提供 `(low, high)`，匹配落在该范围内的值              |

**3. `size`（可选）**

- 示例：`size: 1024`
- 含义：表的最大条目数（硬件资源限制），控制平面不应下发超过此数量的规则。

**4. `with_counters` / `with_meters`（可选）**

- 示例：`with_counters: true`
- 含义：这张表是否关联了计数器（统计命中次数）或 meter（限速）。如果为 `true`，控制平面可以读取统计信息。

**5. `idle_timeout_behavior`（可选）**

- 含义：是否支持空闲超时（条目长期不命中自动老化），用于实现路由老化等场景。

------

**二、`actions` 列表中的每个元素**

每个 `action` 对象描述了一个 P4 动作（Action）的定义。**注意**：表中的 `actions` 字段通常是一个**动作引用列表**，而全局的 `actions` 列表才是动作的完整定义。

**1. `preamble`（同表）**

| 字段    | 示例值                     | 解释          |
| :------ | :------------------------- | :------------ |
| `id`    | `28792405`                 | 动作的唯一 ID |
| `name`  | `"MyIngress.ipv4_forward"` | 完整动作名    |
| `alias` | `"ipv4_forward"`           | 短别名        |

**2. `params`（参数列表）**

动作执行时需要的参数，相当于函数的形参。

每个参数包含：

| 字段       | 示例值      | 解释                           |
| :--------- | :---------- | :----------------------------- |
| `id`       | `1`         | 参数在动作内的 ID（从 1 开始） |
| `name`     | `"dstAddr"` | 参数名                         |
| `bitwidth` | `48`        | 参数位宽（单位：bit）          |

在 `ipv4_forward` 的例子中：

- `dstAddr`（48bit，MAC 地址）
- `port`（9bit，端口号）

控制平面下发规则时，需要为这个动作提供具体的参数值，例如：`action = ipv4_forward(dstAddr=0x112233445566, port=3)`

**3. `annotations`（可选）**

- 示例：`annotations: "@proto_id(1)"`
- 含义：用于生成 protobuf 消息时的字段编号映射

------

**三、完整示例对照**

以你贴出的 `basic.p4info` 为例：

```
tables:
  - preamble: { id: 37375156, name: "MyIngress.ipv4_lpm" }
    match_fields:
      - id: 1
        name: "hdr.ipv4.dstAddr"
        bitwidth: 32
        match_type: LPM
    actions:
      - 28792405   # 引用 MyIngress.ipv4_forward
      - 25652968   # 引用 MyIngress.drop
      - 21257015   # 引用 NoAction

actions:
  - preamble: { id: 28792405, name: "MyIngress.ipv4_forward" }
    params:
      - id: 1, name: "dstAddr", bitwidth: 48
      - id: 2, name: "port", bitwidth: 9
  - preamble: { id: 25652968, name: "MyIngress.drop" }
    params: []   # drop 动作无参数
  - preamble: { id: 21257015, name: "NoAction" }
    params: []   # NoAction 是 P4 内置的空动作
```

### 5.2 三个包装（第 40-47 行）

```python
40:     def get_id(self, entity_type, name):
41:         return self.get(entity_type, name=name).preamble.id
42:
43:     def get_name(self, entity_type, id):
44:         return self.get(entity_type, id=id).preamble.name
45:
46:     def get_alias(self, entity_type, id):
47:         return self.get(entity_type, id=id).preamble.alias
```

- 就是在 `get` 找到对象后，再取它的 `preamble.id` / `.name` / `.alias`。
- 例：`get_id("tables", "MyIngress.ipv4_lpm")` → `37375156`。

### 5.3 `__getattr__`：魔法方法自动生成便捷函数（第 49-64 行）

**一、Python 属性访问的完整流程**

当你写 `obj.anything` 时，Python 按以下顺序查找：

```python
1. 检查实例的 __dict__ 中是否有 'anything'？
   └─ 没有 → 继续

2. 检查类（以及父类）中是否有 'anything' 方法或属性？
   └─ 没有 → 继续

3. 检查类是否定义了 __getattr__()？
   └─ 有！→ 调用 __getattr__(self, "anything")
   └─ 没有 → 抛出 AttributeError
```

------

**二、具体示例演示**

**示例 1：有 `__getattr__` 的情况**

```python
class MyClass:
    def __init__(self):
        self.existing_attr = "I exist!"  # ← 实例属性
        # 注意：没有定义 anything 属性
    
    def __getattr__(self, name):
        print(f"触发 __getattr__，查找的属性是: {name}")
        return f"动态返回: {name}"

obj = MyClass()

# 场景 1：访问存在的属性
print(obj.existing_attr)  
# 输出: I exist!
# 解释：第1步就找到了 existing_attr，直接返回，不触发 __getattr__

# 场景 2：访问不存在的属性
print(obj.anything)  
# 输出:
# 触发 __getattr__，查找的属性是: anything
# 动态返回: anything
# 解释：第1、2步都找不到，第3步触发 __getattr__

# 场景 3：访问另一个不存在的属性
print(obj.hello)  
# 输出:
# 触发 __getattr__，查找的属性是: hello
# 动态返回: hello
```

------

**三、为什么 Python 要这样设计？**

**用途 1：动态属性访问（像这段代码做的）**

```python
# 不用写死所有方法，让 __getattr__ 动态处理
class DynamicAPI:
    def __getattr__(self, name):
        if name.startswith("get_"):
            return lambda: f"Calling {name}"
        raise AttributeError(name)

api = DynamicAPI()
print(api.get_user())    # "Calling get_user" （动态生成）
print(api.get_order())   # "Calling get_order" （动态生成）
```

**用途 2：代理模式（Proxy Pattern）**

```python
class Proxy:
    def __init__(self, obj):
        self._obj = obj
    
    def __getattr__(self, name):
        print(f"访问了属性: {name}")
        return getattr(self._obj, name)  # 转发给真实对象

real = MyClass()
proxy = Proxy(real)
print(proxy.existing_attr)  # 打印 "访问了属性: existing_attr" 然后返回真实值
```

**用途 3：懒加载（Lazy Loading）**

```python
class LazyData:
    def __init__(self):
        self._data = None
    
    def __getattr__(self, name):
        if self._data is None:
            print("加载数据...")
            self._data = {"name": "Alice", "age": 30}
        return self._data[name]

obj = LazyData()
print(obj.name)  # 第一次访问，加载数据... Alice
print(obj.age)   # 第二次访问，直接返回 30（不重复加载）
```



```python
def __getattr__(self, attr):  # attr 是访问的属性名字符串
    # 1. 匹配模式：get_<类型>_id
    m = re.search(r"^get_(\w+)_id$", attr)
    if m:
        primitive = m.group(1)  # 提取 "tables"、"actions" 等
        return lambda name: self.get_id(primitive, name)  # 返回一个动态生成的函数
    
    # 2. 匹配模式：get_<类型>_name
    m = re.search(r"^get_(\w+)_name$", attr)
    if m:
        primitive = m.group(1)
        return lambda id: self.get_name(primitive, id)
    
    # 3. 都不匹配，抛出异常
    raise AttributeError(...)
```

```text
helper = P4InfoHelper("basic.p4info.txtpb")

# === 场景 1：调用不存在的 get_tables_id() 方法 ===
# Python 找不到 get_tables_id 方法，触发 __getattr__("get_tables_id")
table_id = helper.get_tables_id("MyIngress.ipv4_lpm")
print(table_id)  # 37375156

# 内部发生了什么：
# 1. __getattr__ 收到 attr = "get_tables_id"
# 2. 正则匹配到 "^get_(\w+)_id$"，提取 primitive = "tables"
# 3. 返回 lambda name: self.get_id("tables", name)
# 4. 调用时传入 "MyIngress.ipv4_lpm"，实际执行 helper.get_id("tables", "MyIngress.ipv4_lpm")


# === 场景 2：调用不存在的 get_actions_id() 方法 ===
action_id = helper.get_actions_id("MyIngress.ipv4_forward")
print(action_id)  # 28792405

# 同样触发 __getattr__，提取 primitive = "actions"


# === 场景 3：调用不存在的 get_tables_name() 方法 ===
table_name = helper.get_tables_name(37375156)
print(table_name)  # "MyIngress.ipv4_lpm"

# 触发 __getattr__("get_tables_name")
# 正则匹配到 "^get_(\w+)_name$"，提取 primitive = "tables"
# 返回 lambda id: self.get_name("tables", id)


# === 场景 4：调用不存在的随机方法 ===
helper.get_something_else()  # ❌ AttributeError
```

**这是 helper.py 最"魔法"的地方。** Python 里当你访问一个**不存在的属性**时，会自动调用 `__getattr__(attr)`。它的工作：

- **第 52-55 行**：如果属性名符合 `get_xxx_id` 这种模式（如 `get_tables_id`），就**现场造一个函数**返回给你：
  
  ```python
  self.p4info_helper.get_tables_id('MyIngress.ipv4_lpm')
  #  找不到 get_tables_id 这个属性 → 触发 __getattr__("get_tables_id")
  #  → 造出 lambda name: self.get_id("tables", name)
  #  → 返回 37375156
  ```
- **第 59-62 行**：同理，`get_xxx_name` 模式生成"ID → 名字"的函数。
- **第 64 行**：既不是 `_id` 也不是 `_name` 结尾 → 真的没有这个属性，抛 AttributeError。

> 这样写的好处：**不用为每个实体类型（tables、actions、counters…）手写 lookup 函数**，
> 一个正则就全覆盖。你甚至能调用 `get_counters_id`、`get_direct_counters_id` 而无需提前定义。

---

## 6. 匹配字段组（表项的 key 部分）

### 6.1 `get_match_field`（第 66-77 行）



```python
66:     def get_match_field(self, table_name, name=None, id=None):
67:         for t in self.p4info.tables:               # 遍历所有表
68:             pre = t.preamble
69:             if pre.name == table_name:             # ① 找到指定的表
70:                 for mf in t.match_fields:          # ② 遍历这张表的匹配字段
71:                     if name is not None:
72:                         if mf.name == name:        # 按字段名找
73:                             return mf
74:                     elif id is not None:
75:                         if mf.id == id:            # 按字段 ID 找
76:                             return mf
77:         raise AttributeError(...)
```

和 `get` 的区别：`get` 查的是"顶层实体"（tables、actions），这里要查**嵌套在表里面的匹配字段**，所以要两层循环：先找表，再找字段。返回的 `mf` 是 `match_fields` 里的一个元素，含 `id`、`name`、`bitwidth`、`match_type`。

### 6.2 `get_match_field_pb`：编码核心（第 85-108 行）

```python
85:     def get_match_field_pb(self, table_name, match_field_name, value):
86:         p4info_match = self.get_match_field(table_name, match_field_name)  # ① 找到字段定义
87:         bitwidth = p4info_match.bitwidth        # ② 从定义里读出位宽（如 32）
88:         p4runtime_match = p4runtime_pb2.FieldMatch()   # ③ 建一个运行时匹配消息
89:         p4runtime_match.field_id = p4info_match.id     # ④ 填字段 ID
90:         match_type = p4info_match.match_type           # ⑤ 看匹配类型
91:         if match_type == p4info_pb2.MatchField.EXACT:
92:             exact = p4runtime_match.exact
93:             exact.value = encode(value, bitwidth)      # 精确：只填值
94:         elif match_type == p4info_pb2.MatchField.LPM:
95:             lpm_entry = p4runtime_match.lpm
96:             lpm_entry.value = encode(value[0], bitwidth)  # LPM：填值
97:             lpm_entry.prefix_len = value[1]               # LPM：填前缀长度
98:         elif match_type == p4info_pb2.MatchField.TERNARY:
99:             ternary_entry = p4runtime_match.ternary
100:             ternary_entry.value = encode(value[0], bitwidth)  # 三元：值
101:             ternary_entry.mask = encode(value[1], bitwidth)   # 三元：掩码
102:         elif match_type == p4info_pb2.MatchField.RANGE:
103:             range_entry = p4runtime_match.range  # 返回的是同一个对象的引用
104:             range_entry.low = encode(value[0], bitwidth)     # 范围：下界
105:             range_entry.high = encode(value[1], bitwidth)    # 范围：上界
106:         else:
107:             raise Exception("Unsupported match type with type %r" % match_type)
108:         return p4runtime_match
```

**一、函数作用总览**

```
输入：表名 + 匹配字段名 + 人类可读的值
  ↓
输出：P4Runtime 的 FieldMatch protobuf 对象（可用于 gRPC 下发）

例如：
get_match_field_pb("MyIngress.ipv4_lpm", "hdr.ipv4.dstAddr", ("192.168.1.0", 24))
  ↓
返回 FieldMatch {
    field_id: 1,
    lpm: { value: b'\xc0\xa8\x01\x00', prefix_len: 24 }
}
```



------

**二、逐行解析**

**第 85-87 行：获取字段定义**

```python
def get_match_field_pb(self, table_name, match_field_name, value):
    p4info_match = self.get_match_field(table_name, match_field_name)  # ①
    bitwidth = p4info_match.bitwidth        # ②
```

**① 查找字段定义**：从 p4info 中获取该字段的完整定义（包括 ID、位宽、匹配类型等）。
**② 提取位宽**：获取该字段的位宽（如 IPv4 地址是 32 位，MAC 地址是 48 位）。

**第 88-89 行：创建 protobuf 对象**

```python
p4runtime_match = p4runtime_pb2.FieldMatch()   # ③
p4runtime_match.field_id = p4info_match.id     # ④
```

**③**：创建 P4Runtime 协议中定义的 `FieldMatch` 空对象。
**④**：填入字段 ID（从 p4info 中获取，如 `1`），告诉数据平面"这个匹配是针对哪个字段"。

**第 90 行：判断匹配类型**

```python
match_type = p4info_match.match_type           # ⑤
```

根据 p4info 中定义的匹配类型，选择不同的编码方式。

------

**三、四种匹配类型的处理**

**1. EXACT（精确匹配）**

```python
if match_type == p4info_pb2.MatchField.EXACT:
    exact = p4runtime_match.exact
    exact.value = encode(value, bitwidth)      # 精确：只填值
```

**使用场景**：完全匹配（如 MAC 地址、TCP 端口）。

**示例**：

```python
# 表：mac_table，匹配字段：hdr.ethernet.dstAddr (bitwidth=48)
# 调用
get_match_field_pb("mac_table", "hdr.ethernet.dstAddr", "aa:bb:cc:dd:ee:ff")

# 生成
FieldMatch {
    field_id: 1,
    exact: { value: b'\xaa\xbb\xcc\xdd\xee\xff' }  # 48位二进制
}
```

------

**2. LPM（最长前缀匹配）**

```python
elif match_type == p4info_pb2.MatchField.LPM:
    lpm_entry = p4runtime_match.lpm
    lpm_entry.value = encode(value[0], bitwidth)  # LPM：填值
    lpm_entry.prefix_len = value[1]               # LPM：填前缀长度
```

**使用场景**：IP 路由（如目标网段）

**示例**：

```python
# 表：ipv4_lpm，匹配字段：hdr.ipv4.dstAddr (bitwidth=32)
# value 是一个元组 (IP地址, 前缀长度)
get_match_field_pb("ipv4_lpm", "hdr.ipv4.dstAddr", ("192.168.1.0", 24))

# 生成
FieldMatch {
    field_id: 1,
    lpm: { 
        value: b'\xc0\xa8\x01\x00',  # 192.168.1.0 的二进制
        prefix_len: 24              # 匹配前24位
    }
}
```

------

**3. TERNARY（三态匹配）**

```python
elif match_type == p4info_pb2.MatchField.TERNARY:
    ternary_entry = p4runtime_match.ternary
    ternary_entry.value = encode(value[0], bitwidth)  # 三元：值
    ternary_entry.mask = encode(value[1], bitwidth)   # 三元：掩码
```

**使用场景**：通配符匹配（如 ACL，允许/拒绝特定范围的流量）。

**示例**：

```python
# 匹配 192.168.1.x 的所有地址（忽略最后8位）
# value = (值, 掩码)
get_match_field_pb("acl_table", "hdr.ipv4.srcAddr", ("192.168.1.0", "255.255.255.0"))

# 生成
FieldMatch {
    field_id: 1,
    ternary: {
        value: b'\xc0\xa8\x01\x00',      # 192.168.1.0
        mask: b'\xff\xff\xff\x00'        # 255.255.255.0（最后8位被忽略）
    }
}
```

**TERNARY vs LPM 的区别**：

- LPM：掩码必须是**前缀形式**（1...10...0），如 `/24` = `11111111.11111111.11111111.00000000`
- TERNARY：掩码可以是**任意形式**，如 `10101010.11110000.00001111.00000000`（非连续的 0/1）

------

**4. RANGE（范围匹配）**

```python
elif match_type == p4info_pb2.MatchField.RANGE:
    range_entry = p4runtime_match.range
    range_entry.low = encode(value[0], bitwidth)     # 范围：下界
    range_entry.high = encode(value[1], bitwidth)    # 范围：上界
```

**使用场景**：端口范围、数值区间。

**示例**：

```python
# 匹配 TCP 端口 1024-65535
# value = (下界, 上界)
get_match_field_pb("acl_table", "hdr.tcp.dstPort", (1024, 65535))

# 生成
FieldMatch {
    field_id: 2,
    range: {
        low: b'\x04\x00',   # 1024 的二进制
        high: b'\xff\xff'   # 65535 的二进制
    }
}
```

------

**四、`encode()` 函数的作用**

虽然这段代码没有展示 `encode()` 的实现，但它的功能是：

```python
def encode(value, bitwidth):
    """
    将人类可读的值转换为指定位宽的二进制字节串（大端序）
    """
    # 示例：
    # encode("192.168.1.0", 32) → b'\xc0\xa8\x01\x00'
    # encode("aa:bb:cc:dd:ee:ff", 48) → b'\xaa\xbb\xcc\xdd\xee\xff'
    # encode(80, 16) → b'\x00\x50'
    # encode(1024, 16) → b'\x04\x00'
```

**关键点**：

- 所有值都转换为**大端序**（网络字节序）
- 长度必须精确等于 `bitwidth / 8` 字节
- 支持字符串（IP、MAC）和整数两种输入

------

**五、完整工作流程图**

```
调用: get_match_field_pb("ipv4_lpm", "hdr.ipv4.dstAddr", ("192.168.1.0", 24))
    │
    ├─ ① self.get_match_field("ipv4_lpm", "hdr.ipv4.dstAddr")
    │   └─ 从 p4info 获取字段定义：
    │       { id: 1, bitwidth: 32, match_type: LPM }
    │
    ├─ ② bitwidth = 32
    │
    ├─ ③ 创建空的 FieldMatch 对象
    │
    ├─ ④ p4runtime_match.field_id = 1
    │
    ├─ ⑤ match_type = LPM
    │
    ├─ ⑥ 进入 LPM 分支：
    │   ├─ lpm_entry.value = encode("192.168.1.0", 32) = b'\xc0\xa8\x01\x00'
    │   └─ lpm_entry.prefix_len = 24
    │
    └─ ⑦ 返回构建好的 FieldMatch 对象
    
最终结果：
FieldMatch {
    field_id: 1,
    lpm: { value: b'\xc0\xa8\x01\x00', prefix_len: 24 }
}
```

**最重要的设计点：匹配类型（EXACT/LPM/TERNARY/RANGE）是从 p4info 里自动读出来的，调用方不用告诉它！**

basic 的例子（LPM 分支）：
```python
# 调用方只写了这一行：
match_fields={'hdr.ipv4.dstAddr': ('10.0.1.1', 32)}
# 内部：
#   p4info_match.match_type == LPM  → 走第 94-97 行
#   value[0]='10.0.1.1' → encode → \x0a\x00\x01\x01（4 字节，因为 bitwidth=32）
#   value[1]=32         → prefix_len = 32
```

- 如果 basic 表的 key 是 `exact`，调用方就不用传 `(value, len)`，传单个值即可，走第 91-93 行。
- **调用格式随匹配类型不同而不同**，这是 P4Runtime 协议的硬性规定。

### 6.3 `get_match_field_value`：解码（第 110-123 行）

这段代码是 `get_match_field_pb()` 的**逆向操作**——它从 P4Runtime 的 `FieldMatch` protobuf 消息中**提取出人类可读的匹配值**。这是一个"解码器"，用于从数据平面读取规则时解析匹配字段。

------

**一、函数作用总览**

```
输入：P4Runtime 的 FieldMatch protobuf 对象（从交换机读回的规则）
  ↓
输出：Python 原生数据类型（int、bytes、tuple 等）

例如：
从交换机读回的 FieldMatch { lpm: { value: b'\xc0\xa8\x01\x00', prefix_len: 24 } }
  ↓
返回： (b'\xc0\xa8\x01\x00', 24)  或解码后返回 ("192.168.1.0", 24)
```

------

**二、`WhichOneof()` 方法详解**

**什么是 `WhichOneof()`？**

在 protobuf 中，`oneof` 字段表示**一组互斥的字段**，最多只有一个会被设置。

```protobuf
// P4Runtime 的 FieldMatch 定义（简化）
message FieldMatch {
    uint32 field_id = 1;
    oneof field_match_type {    // ← 这些字段互斥
        Exact exact = 2;
        LPM lpm = 3;
        Ternary ternary = 4;
        Range range = 5;
        Valid valid = 6;        // 用于 match_type = OPTIONAL
    }
}
```

**`WhichOneof("field_match_type")`** 返回**当前被设置的那个字段的名字**：

- 如果设置了 `exact` → 返回 `"exact"`
- 如果设置了 `lpm` → 返回 `"lpm"`
- 如果都没设置 → 返回 `None`

------

**三、逐行解析**

**第 111 行：检测匹配类型**

```python
match_type = match_field.WhichOneof("field_match_type")
```

**作用**：检查这个 `FieldMatch` 消息实际使用了哪种匹配类型。

**可能的返回值**：

| 返回值      | 对应的匹配类型 | 说明                         |
| :---------- | :------------- | :--------------------------- |
| `"exact"`   | EXACT          | 精确匹配                     |
| `"lpm"`     | LPM            | 最长前缀匹配                 |
| `"ternary"` | TERNARY        | 三态匹配                     |
| `"range"`   | RANGE          | 范围匹配                     |
| `"valid"`   | OPTIONAL       | 可选匹配（检查字段是否有效） |
| `None`      | (未设置)       | 没有匹配值                   |

**第 112-113 行：VALID（可选匹配）**

```python
if match_type == 'valid':
    return match_field.valid.value
```

**使用场景**：P4 中的 `optional` 匹配类型，检查字段是否存在（如 `hdr.ethernet` 是否有效）。

**示例**：

```python
# P4 定义：match_field = hdr.ethernet.etherType : optional
# 收到的 FieldMatch
FieldMatch {
    field_id: 3,
    valid: { value: True }  # 或者 False
}

# 返回值：True 或 False
```

**第 114-115 行：EXACT（精确匹配）**

```python
elif match_type == 'exact':
    return match_field.exact.value
```

**示例**：

```python
# 收到的 FieldMatch
FieldMatch {
    field_id: 1,
    exact: { value: b'\xaa\xbb\xcc\xdd\xee\xff' }
}

# 返回值：b'\xaa\xbb\xcc\xdd\xee\xff'（二进制字节串）
```

**第 116-117 行：LPM（最长前缀匹配）**

```python
elif match_type == 'lpm':
    return (match_field.lpm.value, match_field.lpm.prefix_len)
```

**示例**：

```python
# 收到的 FieldMatch
FieldMatch {
    field_id: 2,
    lpm: { 
        value: b'\xc0\xa8\x01\x00', 
        prefix_len: 24 
    }
}

# 返回值：(b'\xc0\xa8\x01\x00', 24)  # 元组
```





```python
110:     def get_match_field_value(self, match_field):
111:         match_type = match_field.WhichOneof("field_match_type")  # 看这个消息里实际是哪种
112:         if match_type == 'valid':
113:             return match_field.valid.value
114:         elif match_type == 'exact':
115:             return match_field.exact.value
116:         elif match_type == 'lpm':
117:             return (match_field.lpm.value, match_field.lpm.prefix_len)
118:         ...
```

- 这是编码的逆操作：把交换机**返回**的 `FieldMatch` 消息还原成 (值, 前缀) 这种人类可读形式。`readTableRules` 显示表项内容时用得上。
- `WhichOneof("field_match_type")` 是 protobuf 的方法：看这条消息到底填了 `exact`/`lpm`/`ternary`/`range` 里的哪一个。

---

## 7. 动作参数组（表项的 action 部分）

### 7.1 `get_action_param`（第 125-136 行）

```python
125:     def get_action_param(self, action_name, name=None, id=None):
126:         for a in self.p4info.actions:            # 遍历所有动作
127:             pre = a.preamble
128:             if pre.name == action_name:          # ① 找到动作
129:                 for p in a.params:               # ② 遍历动作的参数
130:                     if name is not None:
131:                         if p.name == name:       # 按参数名找
132:                             return p
133:                     elif id is not None:
134:                         if p.id == id:           # 按参数 ID 找
135:                             return p
136:         raise AttributeError(...)
```

和 `get_match_field` 结构一模一样：外层找动作，内层找参数。

### 7.2 `get_action_param_pb`：编码（第 144-149 行）

```python
144:     def get_action_param_pb(self, action_name, param_name, value):
145:         p4info_param = self.get_action_param(action_name, param_name)  # 找到参数定义
146:         p4runtime_param = p4runtime_pb2.Action.Param()    # 建运行时参数消息
147:         p4runtime_param.param_id = p4info_param.id        # 填参数 ID
148:         p4runtime_param.value = encode(value, p4info_param.bitwidth)  # 编码值
149:         return p4runtime_param
```

```protobuf
// p4runtime.proto 中的定义
// p4runtime_pb2.Action.Param()
message Action {
    // 动作 ID（对应 p4info 中的 action id）
    uint32 action_id = 1;
    
    // 动作参数列表
    repeated Param params = 2;
    
    // 参数消息定义（嵌套在 Action 内部）
    message Param {
        // 参数 ID（对应 p4info 中的 param id）
        uint32 param_id = 1;
        
        // 参数值（二进制格式）
        bytes value = 2;
    }
}
```

**一、给定的 p4info JSON 结构**

```json
{
  "tables": [
    {
      "preamble": { "id": 37375156, "name": "MyIngress.ipv4_lpm" },
      "match_fields": [
        { "id": 1, "name": "hdr.ipv4.dstAddr", "bitwidth": 32, "match_type": "LPM" }
      ],
      "action_ids": [28792405, 25652968, 21257015]
    }
  ],
  "actions": [
    {
      "preamble": { "id": 28792405, "name": "MyIngress.ipv4_forward" },
      "params": [
        { "id": 1, "name": "dstAddr", "bitwidth": 48 },
        { "id": 2, "name": "port", "bitwidth": 9 }
      ]
    },
    {
      "preamble": { "id": 25652968, "name": "MyIngress.drop" },
      "params": []
    },
    {
      "preamble": { "id": 21257015, "name": "NoAction" },
      "params": []
    }
  ]
}
```

------

**二、执行 get_action_param_pb() 的完整转换**

**调用代码**

```python
# 为 ipv4_forward 动作的 dstAddr 参数编码
param = helper.get_action_param_pb(
    action_name="MyIngress.ipv4_forward",
    param_name="dstAddr",
    value="aa:bb:cc:dd:ee:ff"
)
```

**第 145 行：从 JSON 中查找参数定义**

```python
p4info_param = self.get_action_param("MyIngress.ipv4_forward", "dstAddr")
```

**从 JSON 中查找到的内容**：

```json
// 在 actions 数组中找到 "MyIngress.ipv4_forward"
// 然后在它的 params 数组中找到 name="dstAddr" 的参数
{
    "id": 1,
    "name": "dstAddr",
    "bitwidth": 48
}
```

**p4info_param 变为**：

```python
p4info_param = {
    id: 1,
    name: "dstAddr",
    bitwidth: 48
}
```

**第 146 行：创建空的运行时参数对象**

```python
p4runtime_param = p4runtime_pb2.Action.Param()
```

**创建的空对象**：

```python
p4runtime_param = {
    param_id: 0,    # 默认值
    value: b''      # 空字节串
}
```

**第 147 行：填参数 ID**

```python
p4runtime_param.param_id = p4info_param.id  # p4info_param.id = 1
```

**p4runtime_param 变为**：

```python
p4runtime_param = {
    param_id: 1,    # ← 从 JSON 的 id 字段来
    value: b''
}
```

**第 148 行：编码参数值**

```python
p4runtime_param.value = encode(value, p4info_param.bitwidth)
# encode("aa:bb:cc:dd:ee:ff", 48)
```

**编码过程**：

```python
# 输入：value = "aa:bb:cc:dd:ee:ff", bitwidth = 48
# 48 bits = 6 bytes
# 解析 MAC 地址，转为 6 字节二进制

encoded_value = b'\xaa\xbb\xcc\xdd\xee\xff'
```

**p4runtime_param 变为**：

```python
p4runtime_param = {
    param_id: 1,
    value: b'\xaa\xbb\xcc\xdd\xee\xff'  # ← 从 value 参数编码而来
}
```

**第 149 行：返回最终结果**

```python
return p4runtime_param
```

**最终返回的 protobuf 对象**：

```python
Action.Param {
    param_id: 1,
    value: b'\xaa\xbb\xcc\xdd\xee\xff'
}
```



------

**三、完整的 JSON → protobuf 映射**

**输入（JSON 中的定义）**

```json
{
    "id": 1,
    "name": "dstAddr",
    "bitwidth": 48
}
```

用户输入的值

```
"aa:bb:cc:dd:ee:ff"
```

↓

**输出（P4Runtime protobuf）**

```python
Action.Param {
    param_id: 1,
    value: b'\xaa\xbb\xcc\xdd\xee\xff'
}
```

------

**四、另一个参数的完整转换**

**调用：port 参数**

```python
param = helper.get_action_param_pb(
    action_name="MyIngress.ipv4_forward",
    param_name="port",
    value=3
)
```

**第 145 行：从 JSON 查找**

```python
// 从 actions[0].params 中找到 name="port"
{
    "id": 2,
    "name": "port",
    "bitwidth": 9
}
```

**第 146-148 行：构建 protobuf**

```python
# p4info_param.id = 2
# encode(3, 9) → b'\x00\x03' (9位需要2字节)
```

**最终输出**

```protobuf
Action.Param {
    param_id: 2,
    value: b'\x00\x03'
}
```

basic 的例子：

```python
# 调用方：
action_params={'dstAddr': '08:00:00:00:02:22', 'port': 2}
# 内部：
#   'dstAddr' → p4info 里 bitwidth=48 → encode → 6 字节
#   'port'    → p4info 里 bitwidth=9  → encode → 2 字节 \x00\x02
```

---

## 8. `buildTableEntry`：组装工厂（第 151-185 行）—— 最重要的公共 API

```python
151:     def buildTableEntry(self,
152:                         table_name,          # 表名（字符串）
153:                         match_fields=None,   # 匹配键：dict，如 {'hdr.ipv4.dstAddr': ('10.0.1.1', 32)}
154:                         default_action=False,# 是否设置默认动作
155:                         action_name=None,    # 动作名
156:                         action_params=None,  # 动作参数：dict，如 {'dstAddr': ..., 'port': ...}
157:                         idle_timeout_ns=None,# 表项空闲超时
158:                         priority=None):      # 优先级（ternary 用）
159:         table_entry = p4runtime_pb2.TableEntry()          # ① 建一条空 TableEntry
160:         table_entry.table_id = self.get_tables_id(table_name)   # ② 表名 → 表 ID
161:
162:         if priority is not None:
163:             table_entry.priority = priority               # 可选：优先级
164:
165:         if idle_timeout_ns is not None:
166:             table_entry.idle_timeout_ns = idle_timeout_ns # 可选：空闲超时
167:
168:         if match_fields:
169:             table_entry.match.extend([
170:                 self.get_match_field_pb(table_name, match_field_name, value)
171:                 for match_field_name, value in match_fields.items()   # ③ 每个匹配键转成 FieldMatch
172:             ])
173:
174:         if default_action:
175:             table_entry.is_default_action = True           # ④ 标记为默认动作
176:
177:         if action_name:
178:             action = table_entry.action.action            # ⑤ 设置动作
179:             action.action_id = self.get_actions_id(action_name)   # 动作名 → 动作 ID
180:             if action_params:
181:                 action.params.extend([
182:                     self.get_action_param_pb(action_name, field_name, value)
183:                     for field_name, value in action_params.items()  # ⑥ 每个参数转成 Action.Param
184:                 ])
185:         return table_entry                              # ⑦ 返回组装好的消息
```

逐段：
- **第 159-160 行**：建空消息，填表 ID。`self.get_tables_id` 是靠第 5.3 节 `__getattr__` 自动生成的函数。
- **第 168-172 行**：`match_fields` 是个 dict，`items()` 逐个取出 `(字段名, 值)`，每个都调 `get_match_field_pb` 转成 `FieldMatch`，`extend` 追加到 `table_entry.match` 列表。**dict 的迭代顺序即匹配字段的先后顺序**。
- **第 174-175 行**：`default_action=True` 时只设标记（不设 match、priority），表示"这是表的默认动作"。
- **第 177-184 行**：填动作。`action.action_id` 是动作 ID；参数逐条编码进 `action.params`。
- **第 185 行**：返回完整的 `TableEntry` 消息，供 switch.py 的 `WriteTableEntry` 发送。

最终产出的消息（对应 basic 的 FwdTest）：

```protobuf
TableEntry {
  table_id: 37375156                       # MyIngress.ipv4_lpm
  match {
    field_id: 1                            # hdr.ipv4.dstAddr
    lpm { value: "\x0a\x00\x01\x01"        # 10.0.1.1
          prefix_len: 32 }
  }
  action {
    action_id: 28792405                    # MyIngress.ipv4_forward
    params { param_id: 1                   # dstAddr
             value: "\x08\x00\x00\x00\x02\x22" }
    params { param_id: 2                   # port
             value: "\x00\x02" }
  }
}
```



```json
{
  "tables": [
    {
      "preamble": { "id": 37375156, "name": "MyIngress.ipv4_lpm" },
      "match_fields": [
        { "id": 1, "name": "hdr.ipv4.dstAddr", "bitwidth": 32, "match_type": "LPM" }
      ],
      "action_ids": [28792405, 25652968, 21257015]
    }
  ],
  "actions": [
    {
      "preamble": { "id": 28792405, "name": "MyIngress.ipv4_forward" },
      "params": [
        { "id": 1, "name": "dstAddr", "bitwidth": 48 },
        { "id": 2, "name": "port", "bitwidth": 9 }
      ]
    },
    {
      "preamble": { "id": 25652968, "name": "MyIngress.drop" },
      "params": []
    },
    {
      "preamble": { "id": 21257015, "name": "NoAction" },
      "params": []
    }
  ]
}
```

**一、调用示例**

```python
# 添加一条 IPv4 路由：192.168.1.0/24 → 下一跳 MAC=aa:bb:cc:dd:ee:ff, 端口=3
entry = helper.buildTableEntry(
    table_name="MyIngress.ipv4_lpm",          # 表名
    match_fields={                            # 匹配字段
        "hdr.ipv4.dstAddr": ("192.168.1.0", 24)
    },
    action_name="MyIngress.ipv4_forward",     # 动作名
    action_params={                           # 动作参数
        "dstAddr": "aa:bb:cc:dd:ee:ff",
        "port": 3
    },
    priority=100,                             # 优先级
    idle_timeout_ns=60000000000               # 60秒超时
)
```

------

**二、逐步解析：每个参数如何转换**

**第 159-160 行：创建 TableEntry 并设置表 ID**

```python
table_entry = p4runtime_pb2.TableEntry()
table_entry.table_id = self.get_tables_id("MyIngress.ipv4_lpm")
```

**查找 JSON**：

```json
// 在 tables 数组中找到 name="MyIngress.ipv4_lpm"
{
    "preamble": { "id": 37375156, "name": "MyIngress.ipv4_lpm" },
    ...
}
```

**转换结果**：

```python
TableEntry {
    table_id: 37375156  // ← 从 JSON 的 preamble.id 来
}
```

------

**第 162-163 行：设置优先级**

```python
if priority is not None:
    table_entry.priority = 100
```

**转换结果**：

```python
TableEntry {
    table_id: 37375156
    priority: 100  // ← 直接来自输入参数
}
```

------

**第 165-166 行：设置空闲超时**

```python
if idle_timeout_ns is not None:
    table_entry.idle_timeout_ns = 60000000000
```

**转换结果**：

```protobuf
TableEntry {
    table_id: 37375156
    priority: 100
    idle_timeout_ns: 60000000000  // ← 直接来自输入参数
}
```

------

**第 168-172 行：设置匹配字段**

```python
if match_fields:
    table_entry.match.extend([
        self.get_match_field_pb("MyIngress.ipv4_lpm", "hdr.ipv4.dstAddr", ("192.168.1.0", 24))
    ])
```

**`get_match_field_pb()` 内部处理**：

1. **查找字段定义**（JSON）：

```json
// 在 tables[0].match_fields 中找到 name="hdr.ipv4.dstAddr"
{
    "id": 1,
    "name": "hdr.ipv4.dstAddr",
    "bitwidth": 32,
    "match_type": "LPM"
}
```

1. **编码值**：

```python
# match_type = LPM
# bitwidth = 32
# value = ("192.168.1.0", 24)
# encode("192.168.1.0", 32) → b'\xc0\xa8\x01\x00'
```

1. **生成 FieldMatch**：

```python
FieldMatch {
    field_id: 1                    // ← 从 JSON 的 match_fields[0].id 来
    lpm: {
        value: b'\xc0\xa8\x01\x00' // ← 从输入参数编码而来
        prefix_len: 24             // ← 从输入参数来
    }
}
```

**转换结果**：

```protobuf
TableEntry {
    table_id: 37375156
    priority: 100
    idle_timeout_ns: 60000000000
    match: [  // ← 从 match_fields 输入参数来
        {
            field_id: 1
            lpm: {
                value: b'\xc0\xa8\x01\x00'
                prefix_len: 24
            }
        }
    ]
}
```

------

**第 177-184 行：设置动作**

```python
if action_name:
    action = table_entry.action.action
    action.action_id = self.get_actions_id("MyIngress.ipv4_forward")
    if action_params:
        action.params.extend([
            self.get_action_param_pb("MyIngress.ipv4_forward", "dstAddr", "aa:bb:cc:dd:ee:ff"),
            self.get_action_param_pb("MyIngress.ipv4_forward", "port", 3)
        ])
```

**步骤 1：设置动作 ID**

**查找 JSON**：

```python
// 在 actions 数组中找到 name="MyIngress.ipv4_forward"
{
    "preamble": { "id": 28792405, "name": "MyIngress.ipv4_forward" },
    "params": [...]
}
```

**转换**：

```protobuf
action.action_id = 28792405  // ← 从 JSON 的 preamble.id 来
```

**步骤 2：处理参数 1（dstAddr）**

**`get_action_param_pb()` 内部**：

1. **查找参数定义**（JSON）：

```json
// 在 actions[0].params 中找到 name="dstAddr"
{
    "id": 1,
    "name": "dstAddr",
    "bitwidth": 48
}
```

1. **编码值**：

```python
# bitwidth = 48
# value = "aa:bb:cc:dd:ee:ff"
# encode("aa:bb:cc:dd:ee:ff", 48) → b'\xaa\xbb\xcc\xdd\xee\xff'
```

1. **生成 Action.Param**：

```protobuf
Action.Param {
    param_id: 1                    // ← 从 JSON 的 params[0].id 来
    value: b'\xaa\xbb\xcc\xdd\xee\xff'  // ← 从输入参数编码而来
}
```

**步骤 3：处理参数 2（port）**

**`get_action_param_pb()` 内部**：

1. **查找参数定义**（JSON）：

```json
// 在 actions[0].params 中找到 name="port"
{
    "id": 2,
    "name": "port",
    "bitwidth": 9
}
```

1. **编码值**：

```python
# bitwidth = 9
# value = 3
# encode(3, 9) → b'\x00\x03'
```

1. **生成 Action.Param**：

```protobuf
Action.Param {
    param_id: 2                    // ← 从 JSON 的 params[1].id 来
    value: b'\x00\x03'             // ← 从输入参数编码而来
}
```

**转换结果**：

```protobuf
TableEntry {
    table_id: 37375156
    priority: 100
    idle_timeout_ns: 60000000000
    match: [
        {
            field_id: 1
            lpm: {
                value: b'\xc0\xa8\x01\x00'
                prefix_len: 24
            }
        }
    ]
    action: {                        // ← 从 action_name 输入参数来
        action_id: 28792405          // ← 从 JSON 的 actions[0].preamble.id 来
        params: [                    // ← 从 action_params 输入参数来
            {
                param_id: 1          // ← 从 JSON 的 params[0].id 来
                value: b'\xaa\xbb\xcc\xdd\xee\xff'  // ← 从输入参数编码
            },
            {
                param_id: 2          // ← 从 JSON 的 params[1].id 来
                value: b'\x00\x03'   // ← 从输入参数编码
            }
        ]
    }
}
```





---

## 9. 复制引擎（PRE）：`buildMulticastGroupEntry` / `buildCloneSessionEntry`（第 187-207 行）

这两个和转发无关，属于**组播/镜像**（Packet Replication Engine）配置：

```python
187:     def buildMulticastGroupEntry(self, multicast_group_id, replicas):
188:         mc_entry = p4runtime_pb2.PacketReplicationEngineEntry()
189:         mc_entry.multicast_group_entry.multicast_group_id = multicast_group_id
190:         for replica in replicas:
191:             r = p4runtime_pb2.Replica()
192:             r.egress_port = replica['egress_port']   # 复制到哪个端口
193:             r.instance = replica['instance']         # 实例号（同一端口可多实例）
194:             mc_entry.multicast_group_entry.replicas.extend([r])
195:         return mc_entry
```

- **组播**：把一个包复制发给多个端口。`replicas` 是 `[{'egress_port':..,'instance':..}, ...]` 的列表。
- **镜像/克隆**：`buildCloneSessionEntry` 同理，加了一个 `packet_length_bytes`（克隆包只带前 N 字节）。
- 它们和 `buildTableEntry` 一样返回一条 PRE 消息，由 switch.py 的 `WritePREEntry` 发送。
- basic 练习用不到，mycontroller 等高级练习会用。

---

## 10. 调用链总结：一次 `buildTableEntry` 走遍全类

```
buildTableEntry('MyIngress.ipv4_lpm', match_fields, action_name, action_params)
 │
 ├─ self.get_tables_id(...)          ──► __getattr__ → get("tables", name) → table_id
 ├─ get_match_field_pb(...)          ──► get_match_field → 读 bitwidth/match_type
 │        └─ encode(value, bitwidth) ──► convert.encode：字符串/数字 → 字节
 └─ self.get_actions_id(...)         ──► __getattr__ → get("actions", name) → action_id
      └─ get_action_param_pb(...)    ──► get_action_param → 读 bitwidth
              └─ encode(...)         ──► 字节
返回 TableEntry ──► switch.py WriteTableEntry ──► gRPC ──► 交换机
```

---

## 11. 常见问题

### Q1：`get_tables_id` 这个方法在哪定义的？找不到啊
不在类里显式定义。访问 `get_tables_id` 时属性不存在，触发 `__getattr__`（第 49 行），
正则匹配 `get_(\w+)_id` 后**现场生成**。这是 Python 的动态特性。

### Q2：为什么有的 value 传单个值，有的传元组？
取决于匹配类型：
- `exact`：单个值 → `get_match_field_pb` 走 EXACT 分支
- `lpm`：`(值, 前缀长度)` → 走 LPM 分支
- `ternary`：`(值, 掩码)` → 走 TERNARY 分支
- 类型是 p4info 里定的，代码自动选分支。

### Q3：`encode` 在哪、怎么判断字符串是 IP 还是 MAC？
在 `convert.py`（第 71-92 行）：先看是不是 MAC 格式（正则 `xx:xx:xx:xx:xx:xx`），
再看是不是 IPv4（正则 `a.b.c.d`），然后分别用 `bytes.fromhex` / `socket.inet_aton` 转字节。
整数则按位宽补零转字节。末尾有 `assert(len==byte_len)` 兜底校验位宽。

### Q4：为什么要 `preamble`？
P4Runtime 规范里所有实体（表、动作、计数器…）都统一用一个 `preamble` 字段装 `id/name/alias`，
所以 `get` 可以只写一份通用遍历逻辑，不用为每种实体写一遍。

### Q5：`allow_unknown_field=True` 是干什么的？
p4info 文本里如果有当前 protobuf 版本不认识的新字段，`Merge` 默认会报错；
设为 True 后直接跳过未知字段，保证旧代码能读新编译器产出的文件。