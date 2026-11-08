# JNI Native Inspection 与 Invocation

## 适用场景

需要查找 Java native、调用 `RegisterNatives`、调用 static/instance native 方法，或处理
JNI 参数和返回值时查阅本篇。

主要入口是 [python/rundroid/jni.py](/mnt/f/coding-workspace/rundroid/python/rundroid/jni.py)
和 [python/rundroid/emulator.py](/mnt/f/coding-workspace/rundroid/python/rundroid/emulator.py)。

## Native 的三层模型

```text
Java 声明
    -> Native method metadata（native=True）
    -> DYNAMIC RegisterNatives binding 或 STATIC Java_* export
    -> NativeInvocationPlan 编组
    -> ARM64 guest execution
```

`@java_method(..., native=True)` 只声明 Native method，不保存 Python host handler。
`RegisterNatives` 只能绑定 Native 声明；host implementation 与 native 声明冲突时应
fail-fast。

地址解析顺序固定为：

1. DYNAMIC binding。
2. STATIC short `Java_package_Class_method`。
3. STATIC long symbol。
4. 未命中抛 `JniNativeNotFoundError`；跨模块重复抛 ambiguity error。

每次调用都会按当前 runtime 状态解析；如果 library 被重新加载或 binding 发生变化，旧的
`JniNativeFn` 可能失效，需要重新查询。

## 查询 API

```python
native = emu.jni.find_native(
    java_class="com/example/Signer",
    java_method="sign([B)[B",
)
result = emu.invoke_native(native, b"payload")
```

- `native_functions(kind=None)` 枚举 DYNAMIC/STATIC 快照；顺序必须确定性。
- `natives(kind=..., class_prefix=..., module=...)` 是过滤 helper，不建立第二份地址表。
- `find_native(full_signature)` 或 keyword 形式二选一；参数组合错误直接 `TypeError`。
- `JniNativeQuery` 可用于调用尚未绑定的声明，让错误在正式 invocation boundary 暴露。

## ARM64 JNI 编组

- guest ABI 自动注入 `JNIEnv*` 和 `jclass`/`jobject`。
- static method 的 receiver 是有效 jclass reference；instance method 的第一个 Java
  参数是有效 jobject receiver。
- integer/reference 使用独立的 X register 序列；float/double 使用独立的 S/D register
  序列，不能因为整数参数耗尽就错误消耗浮点寄存器。
- **栈参数**：`x0/x1` 之后的 X 序列与 `d0..d7` 分别最多 8 个；某一类别耗尽后，该类别
  的后续参数按声明顺序写入以 callee entry SP 为起点的 memory argument area，每个
  scalar 占 8 字节 slot，整段 allocation 向上补齐到 16 字节。两类 spilled 参数共享同一个
  area，因此顺序仍是原始 descriptor 顺序。合法 typed JNI signature 不再因为"栈传参未
  实现"失败。
- 支持范围仅限 JNI descriptor 可表达的 boolean/byte/char/short/int/long/float/double/
  object/array。C struct/union 按值、HFA/HVA、SIMD/vector、SVE、variadic、ARM32 以及
  raw `call_address` 的无类型超 8 参数都不在范围内。
- ObjectId 是 VM 内部 identity，不能直接写入 guest jobject/jclass/array ABI；对象参数
  必须经本次 local frame 的 RefTable 转换成 guest reference handle——无论它落在寄存器
  还是 stack slot。
- object/array 返回值必须先把 x0 当作 reference handle 解析回 ObjectId，再关闭 local
  frame，然后由 storage-aware marshaller 还原为 Python wrapper/value。
- 超出上述 typed JNI scalar 范围的 ABI 路径应明确 fail-fast，不要截断、丢弃或猜测参数。

## 嵌套数组

`new_object_array` 的 `element_class` 接受普通内部类名或数组 JNI 描述符：

```python
matrix = emu.art.new_object_array([b"first", b"second", None], element_class="[B")
cube = emu.art.new_object_array([matrix, None], element_class="[[B")
```

以上分别是真正的 `byte[][]`（`[[B`）和 `byte[][][]`（`[[[B`）。每次构造增加一维，
最多255维；空数组/null元素仍保留声明类型。非法描述符为 `ValueError`，
不兼容元素为 `TypeError`；不把错误类型降级为 Object[]。作为 typed JNI 参数/返回值
传递时保留数组维数，Java callback 解组后为嵌套 list，byte[] 叶节点为 bytes。

## 使用限制

Java callback 中再次独立 `invoke_native` 属于嵌套 guest execution，会被拒绝；callback
可通过 `ctx.art` 读取或修改 Java 状态。

执行错误会直接抛出；guest 自己设置的 pending exception 按当前 JPE policy 传播。
