# Java Shim Authoring

## 适用场景

需要创建 Java class/interface、补方法或字段、构造对象、处理重载、继承或 Java value
时查阅本篇。

已有 APK/DEX 时，可先用 `rjg` 生成声明，再审查、补写行为并注册；安装、过滤和模式区别见
[java-shim-generation.md](java-shim-generation.md)。生成器不会实现 Java 业务逻辑。

主要入口是 [python/rundroid/javashim/](/mnt/f/coding-workspace/rundroid/python/rundroid/javashim/)
和 [python/rundroid/art.py](/mnt/f/coding-workspace/rundroid/python/rundroid/art.py)。

## 声明与注册

```python
from rundroid.javashim import JavaClass, JByteArray, JInt, java_class, java_method

@java_class("android/content/pm/Signature")
class Signature(JavaClass):
    def __init__(self) -> None:
        self._value = b""

    @java_method("<init>([B)V")
    def init(self, value: JByteArray) -> None:
        self._value = value

    @java_method("hashCode()I")
    def hash_code(self) -> JInt:
        return 0

emu.art.register([Signature])
```

- `@java_class` / `@java_interface` / `@java_method` / `@java_field` 用于声明 Java
  结构；import 时不会自动注册。
- `emu.art.register([...])` 可直接在 `Emulator` 构造成功后调用（构造期已发布 builtin
  bootstrap）；它不会排队或隐式初始化，并会校验 class 是否属于当前 runtime。
- 扩展已存在的 builtin class 要继承 `JavaClassExtension` / `JavaInterfaceExtension` 并
  走 `emu.art.extend([...])`；与 builtin 同 descriptor 的 method/field 还必须声明
  `override=True`，普通同名 definition 会 fail-fast。decorator 没有 `extend=True`
  参数——definition 与 extension 的区分只在**基类**上。
- `JavaInterface` 不能直接实例化；应构造实现类。
- Java descriptor 决定 guest 看到的方法签名。Python 方法名可以不同，但同一个 class
  内的 descriptor 必须唯一且无歧义。

### 什么时候用 declaration / extension / interceptor

| 需求 | 入口 | 生命周期 |
| --- | --- | --- |
| 全新 Java class / interface | `JavaClass` / `JavaInterface` + `emu.art.register([...])` | 长期生效 |
| 给 builtin 补一组 method / field / 关系 | `JavaClassExtension` / `JavaInterfaceExtension` + `emu.art.extend([...])` | 长期生效 |
| 临时改单个 method 行为 | `Interceptor.java_method(..., mode="hook"/"override"/"supplement")` | 到 `emu.interceptor.remove(...)` 为止 |

```python
from rundroid.javashim import JavaClassExtension, JavaField, java_class, java_method

@java_class("android/content/Context")
class ContextExtension(JavaClassExtension):
    extension_state: JavaField[int] = JavaField(
        name="extensionState", sig="I", default=11
    )

    @java_method("extensionValue()I")
    def extension_value(self: "ContextExtension") -> int:
        return self.extension_state

    @java_method("extensionStaticValue()I", static=True)
    def extension_static_value() -> int:
        return 77

emu.art.extend([ContextExtension])
```

入口边界：

- `register([...])` 只接受普通 `JavaClass` / `JavaInterface`；`extend([...])` 只接受
  `JavaClassExtension` / `JavaInterfaceExtension`。交叉传参在任何 runtime mutation 前抛
  `TypeError`，错误会点名正确入口。
- extension 的目标必须在 `emu.art.extend(...)` 之前已经存在，且 `type` 为 `VmCore` /
  `FrameworkBuiltin`；`PythonStub` / `RustManual` / `Placeholder` 会被拒绝。extension 的
  member descriptor 若引用同一个脚本里新建的 class，要先 `register()` 该 definition。

extension 自身的边界：

- extension 保留目标的 class identity、origin、framework session state 与既有成员，
  只追加 method / field / interface 关系；它不改 superclass，也不让目标变得可构造。
- extension 是既有 ObjectId 的 **behavior view**：不能声明 Java `<init>` / `<clinit>`，
  不能定义 Python `__init__`，也不能经 `art.new_object(...)` 或 `ExtensionType(art, ...)`
  实例化（都在 ObjectId 分配前抛 `TypeError`）。需要 framework 实例时只能通过现有
  framework method/factory 取得 session-owned 对象。
- member 默认 add-only：与既有成员或 `supplement` placeholder 冲突时 fail-fast，错误会打印
  双方的 canonical signature 与来源。要替换就对该 member 单独写 `override=True`，且
  native method / `<clinit>` 永远不可覆盖。
- 同一 target 只允许一个 active extension blueprint。
- `register()` / `extend()` 各自是整批操作，任一声明失败时**本批**不会留下部分状态；
  两个调用之间没有跨调用回滚。

### 为什么没有 `new_stub_object()` / `alloc_object()`

曾经考虑（issue #33）新增一个 Python 侧 convenience API，为「framework 缺 method/field 的
对象」直接造空壳。这个方案不采用：

- 空壳对象没有 canonical typed payload，会让 `ObjectStore` 里出现行为不完整的
  `StubInstance`，之后任何 framework method 都会拿到半成品 identity；
- 它把「补齐行为」和「创建 identity」混成一个动作，诱导绕过 `<init>`、伪造 singleton。

正确映射是：缺 method/field 用 **extension**（附着到既有 ObjectId）；需要 framework 实例
时用现有 framework method/factory 拿 session-owned 对象；正常可构造 class 用
`art.new_object(cls, ...)` 匹配 `<init>`。guest JNI 的 `AllocObject` 标准 ABI 不受影响。

## 构造对象

`emu.art.new_object(JavaClass, *args)` 的顺序是：

1. 先按参数解析唯一 `<init>`，任何 Python handler 执行前 fail-fast。
2. 运行时建立对象身份、字段存储和 guest reference。
3. 绑定 Python proxy/cache。
4. 执行 Python `__init__`（如果 blueprint 显式定义）。
5. 严格调用已经解析出的 Java `<init>` descriptor。

构造器只接受位置参数；不支持 kwargs。继承链的默认构造器和 `superclass` 分派由
共享 constructor resolver 处理，不要在每个 class 中手写一套父类调用规则。

字符串 class name 的 `new_object("name")` 是较低级的无参路径：当前只支持已注册 class
的 `<init>()V`，返回通用 `JavaObject`。完整的 Python blueprint 应优先使用 class type。

## 方法、字段和重载

- `@java_method` 支持 instance/static、Python implementation 和 `native=True` 声明。
- 实例方法通过 `JavaObject.method(...)` 或 ART typed dispatch 调用；static 方法可用
  `emu.art.bind_class(Class).static(...)`。
- `@java_field(name=..., sig=..., initial=..., static=...)` 的值由 descriptor 驱动；
  instance/static field 应通过 Java field 声明访问，不要用普通 Python 属性代替 guest 字段。
- overload resolver 根据 descriptor、参数数量、primitive/reference 兼容性和 wrapper
  类型选择唯一候选；并列候选必须报错，不能按定义顺序猜测。
- Java `String`、`byte[]`、primitive wrapper、object array 和 primitive array 有明确
  的 Python wrapper。需要 guest reference identity 时使用 wrapper 的 `opaque_ref`，
  不要把内部 ObjectId 当作 guest jobject。

## 使用限制

- Python `__init__` 的异常会直接抛出；构造失败后应修正 class 或参数再重试。
- `native=True` 的方法只是声明 native 入口，具体调用方式见 JNI native 文档。
- Java 方法 callback 内允许重入 ART 纯状态，例如读写 field、创建 shim 对象；不能再
  调 `emu.call()`、`call_address()` 或新的 guest native execution。
