# Debugging 与 Inspection

## 适用场景

需要下断点、监听内存、查看寄存器/内存/调用栈、追踪指令，或分析 ELF/JNI/crashdump 时
查阅本篇。

## 执行控制

- `emu.breakpoint.add(target, handler)` 注册 code hook；target 可以是地址或已解析符号。
  `handler` 可以是 `Breakpoint` 子类实例，也可以是
  `BreakpointHitCallback`：`Callable[[CpuContext, int], bool]`。直接 callable 与
  `Breakpoint.on_hit(cpu, address)` 收到完全相同的参数。
- `emu.breakpoint.remove(handle)` / `clear()` 管理 breakpoint 生命周期。
- `emu.watch_read((begin, end), handler)` 和 `watch_write(...)` 监听半开区间
  `[begin, end)`；`handler` 可以是 `Watchpoint` 子类实例，也可以是
  `WatchpointHitCallback`：`Callable[[CpuContext, WatchpointEvent], bool]`。直接 callable
  与 `Watchpoint.on_hit(cpu, event)` 收到完全相同的参数；`watch_fetch` 当前是显式
  `NotImplementedError`，不能假装支持。
- 两类 handler 都必须返回 `True` 或 `False`，返回其他 truthy/falsy 值会直接报错。callback
  返回后 `CpuContext` 立即失效；需要留存的数据应在 callback 内复制出来。
- trace 使用 context manager：`trace`、`trace_range`、`trace_call`、`trace_symbol`。
  hook 在 enter 安装，在 exit flush/remove；scope 外的 guest 执行不应写入 trace。
- `trace_call(once=True)` 的重复命中会按当前 contract fail-fast；`gum`/`unidbg` 是公开
  文本格式。

## CpuDebugTool

`cpu.cdt` 是唯一的 callback-scoped 机器调试入口，不提供交互式 console：

```python
def on_hit(cpu: CpuContext, hit_address: int) -> bool:
    dbg = cpu.cdt
    print(f"x0={cpu.regs.x0:#x} pc={cpu.pc:#x}")
    print(dbg.hexdump("x0", 32))
    print(dbg.read_utf8("x1"))
    print(dbg.backtrace())
    return True
```

常用能力：寄存器 dump、地址/module/symbol 解析、frame-pointer backtrace、bytes/int
读取、hexdump、stack/telescope、vtable、C string、Android libc++ string，以及 JNI
string/object/byte-array/object-array/boxed Integer 和 `jmethodID` 解析。ARM64 整数参数使用
`dbg.args()` / `dbg.arg(index)`，返回值使用 `dbg.retval()` / `dbg.set_retval(value)`。

`dbg.jni.read_boxed_int(handle)` 只接受 `java/lang/Integer` 对象 handle；寄存器中的 raw
`jint` 应使用 `dbg.arg(index) & 0xffff_ffff` 或 `dbg.read_i32(address)`，不能当作 object
handle 传入。所有 JNI debug API 都是只读，不会执行 Java 或 guest 代码。

地址参数只接受整数或受支持的 ARM64 寄存器名。复杂表达式如 `x1 + 0x20` 应在脚本中
显式计算，不要扩展成一套隐式表达式语言。

## 寄存器访问

寄存器只有 `cpu.regs` 一个入口，全部是 typed 属性，IDE 能补全、mypy 能检查：

```python
def on_hit(cpu: CpuContext, hit_address: int) -> bool:
    # 反汇编里的 wN 直接对应 cpu.regs.wN，不需要手动 & 0xffffffff。
    if cpu.regs.w0 == 0x2A:
        cpu.regs.w0 = 0              # 同时清零 x0 高 32 位
    cpu.regs.x1 = cpu.regs.lr        # x30 的别名
    return True
```

- `cpu.regs.x0..x30` 是 64-bit 原样读写；`cpu.regs.w0..w30` 是同一份寄存器存储的
  低 32 位 view：读返回零扩展的非负 int，写只保留低 32 位并把对应 `xN` 的高 32 位
  清零。写 `0x0000000189abcdef` 到 `w2` 得到 `x2 == 0x89abcdef`。
- setter 与 X 使用同一输入域：接受 `0..=u64::MAX` 的 Python int；负数或超界抛
  `OverflowError`，非整数抛 `TypeError`，失败时后端寄存器不被修改。
- 别名 `lr`/`fp`/`ip0`/`ip1` 就是 `x30`/`x29`/`x16`/`x17`；`sp`/`pc`/`tpidr_el0` 是
  独立特殊寄存器。`cpu.pc` / `cpu.sp` / `cpu.lr` 与 `cpu.regs.pc/sp/lr` 同源，
  只是 convenience property。
- **没有** `w31` / `wsp` / `wzr` / `xzr`，也不做大小写或表达式重解释；未声明的属性抛
  `AttributeError`。
- **旧字符串入口 `cpu.reg(name)` / `cpu.set_reg(name, value)` 已删除**（运行时
  `AttributeError`，mypy 报成员不存在）。迁移方式：固定名字直接换成属性
  （`cpu.reg("x2")` → `cpu.regs.x2`、`cpu.reg("sp")` → `cpu.regs.sp`）。确实需要按
  动态下标取值时，在脚本内用局部 `Literal`/枚举或显式有限元组建表，不要把
  `getattr(cpu.regs, f"x{index}")` 当作新的通用寄存器 API。
- callback 返回（或抛错）后，已保存的 `regs` 对象对 X/W/别名/特殊寄存器统一抛
  `RuntimeError`；需要留存的值要在 callback 内复制成普通 int。

## Memory 与 map

`emu.memory` 是普通 emulator memory inspector；callback 内的 guest memory、VMA、符号和
pointer inspection 统一使用 `cpu.cdt`。`emu.vmmap` 只查询 VMA，不执行 guest bytes。

- memory read 跨 VMA gap 或遇到 unreadable region 时 fail-fast。
- snapshot 是显式 span 的 immutable bytes 快照，diff 依赖 span 兼容性。
- search 有 limit；不要为了“尽量返回”绕过边界或吞掉 decode error。
- pointer/telescope 会区分 null、module/symbol、string、integer、unmapped、cycle 和
  depth；这些是诊断结果，不应被改成不透明的字符串。

## ELF、JNI 与 crashdump

- `emu.elf` 查询当前 linker/parser catalog：modules、sections、segments、dynsym；
  未找到和 ambiguous 都有专用异常。
- `cpu.cdt.jni` 只查看当前 callback 可见的 JNI object/array/string 和 method token。
- `emu.crashdump` 保存最近一次 guest fault 的只读诊断 facade。fault 发生后，异常应
  保留原始执行错误并附带寄存器、PC/LR、module/VMA、反汇编和栈信息；辅助读取失败
  只能标记 unreadable/unmapped。

## 使用限制

CpuContext 在 callback 返回后失效；不得保存到全局、异步任务或下一次命中。调试 facade
只做当前现场的读取和格式化，不在 callback 中启动新的 guest execution。Java method
interceptor 的 `JavaMethodContext` 是 ART-only，不含 `cpu` 或 `cdt`。
