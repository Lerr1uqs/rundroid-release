# 创建运行时与执行 Native

## 适用场景

需要创建 emulator、加载 `.so`/APK、调用导出函数、初始化 JNI、执行 native 或使用
线程 scheduler 时查阅本篇。

主要入口是 [python/rundroid/emulator.py](/mnt/f/coding-workspace/rundroid/python/rundroid/emulator.py)
和 [python/rundroid/apk.py](/mnt/f/coding-workspace/rundroid/python/rundroid/apk.py)。

## Session 初始化

```python
from rundroid import AndroidEnvProfile, Emulator

profile = AndroidEnvProfile.from_mapping(
    {
        "framework": {
            "system_properties": {"ro.product.model": "rundroid"},
        },
        "syscall": {"cwd": "/data/local/tmp"},
    }
)
emu = Emulator("arm64", "unicorn", 42, profile=profile)
# 需要 APK 时作为构造配置传入（keyword-only）：
# emu = Emulator("arm64", "unicorn", 42, profile=profile, apk="path/to/base.apk")
```

构造成功即 ART/JNI ready：`emu.art` 已发布完整 builtin catalog，`emu.jni_env_pointer` /
`emu.java_vm_pointer` 必定非空。`art.init()` / `init_art()` / `init_jni()` / `load_apk()`
都不存在——不提供 lazy init 或兼容 alias。

- `arch` 当前必须是 `"arm64"`，`backend` 当前必须是 `"unicorn"`。
- `profile` 是 keyword-only 必填参数；运行时会在创建 session 时固定这份设备配置。
- `seed` 用于确定性 syscall runtime。不要在其他模块再次硬编码 pid、TLS 或随机值。

## 推荐执行顺序

```text
构造 Emulator（构造期完成 ART/JNI，可选 apk= 绑定 APK）
    -> 配置 fs / framework / hooks / output
    -> 可选 emu.art.register([...]) / emu.art.extend([...])
    -> load(path) / load_bytes(name) / emu.apk.load_library(name)  -> ElfModule
    -> 可选 jni_onload(module)
    -> call / call_address / invoke_native
    -> 读取结果或关闭 emulator
```

`load(path)`、`load_bytes(name, data)` 与 `APKPackage.load_library(name)` 是 root ELF `.so`
装载入口，统一返回当前 session 的 `ElfModule`（ELF constructor 已完成、`JNI_OnLoad`
未执行）；host 侧 public `dlopen` 已移除，guest 内部的 `dlopen` 仍由 virtual libdl
bridge 处理。APK 只在构造期经 `apk=` 绑定单一 ARM64 base APK，不执行 DEX 或 Java
bytecode；构造后不存在绑定/替换入口。不加载 APK 也可以使用 ART，此时只有读取
package/signature/assets/Application 数据的调用会报告缺少 `ApkContext`。APK native
library 通过统一 linker chain 装载。

ART/builtin/JNI 发布发生在 `Emulator` 构造期：完整 builtin catalog、固定 system-service
singleton 与 JNI ABI 都在返回实例前装好。`emu.art.register([...])`（definition）与
`emu.art.extend([...])`（既有 builtin 的 extension）都直接写入当前 runtime，不排队、
不隐式初始化。

`ElfModule` 是唯一公开 module identity：loader 返回值可直接交给 `jni_onload(module)`、
`dlsym(module, symbol)`、`dlclose(module)`、`emu.elf.section/segment/segments/symbols/base_of`、
`call_module(module, offset)` 与 offset breakpoint/watchpoint。裸 `int` / 字符串 selector
一律 `TypeError`；跨 session 或已卸载 module 在任何机器操作前 fail-fast。

## 三种调用面

- `emu.call(name, *args)`：按导出符号名调用，参数是最多八个 AArch64 整数参数。
- `emu.call_address(addr, *args)`：按绝对 guest 地址进行 raw ABI 调用，不自动注入 JNI
  参数。
- `emu.invoke_native(native, *args)`：走正式 JNI Native invocation，自动处理
  `JNIEnv*`、receiver/reference、descriptor 参数和 typed return。

需要调用 `JNI_OnLoad` 时把 loader 返回的 `ElfModule` 显式传给 `emu.jni_onload(module)`；
JNI_OnLoad 不是普通 library load 的隐式副作用。每个 module 的 active guard 独立——缺
导出、guest 失败或 JNI version 非法都不提交，且一个 module 成功不放行另一个。

## Callback 限制

外层 guest 正在执行时，callback 中不能再次开始 guest execution。也就是说，Java
interceptor callback 里不要调用 `emu.call()`、`call_address()` 或 `invoke_native()`。

callback 需要读写 Java 状态时使用 `ctx.art`；机器 callback 的调试使用 `cpu.cdt`，不要
保存 callback 中的 `CpuContext` 或把 callback context 用到调用返回之后。

## Guest 线程冻结控制

启用 `thread_scheduler=True` 后，`emu.threads` 提供 guest child thread 的查询与冻结控制。
冻结只是把某个 guest 线程从**调度资格**中临时摘除，**不是**挂起 host 后台线程：该线程
仍留在 scheduler 的 task 表中，栈、waiter、TLS 等内部资源保留，只是不再获得 machine
turn，直到 `unfreeze`。

```python
emu = Emulator(
    "arm64", "unicorn", 42, profile=profile,
    thread_scheduler=True, instruction_quantum=257,
)

snapshot = emu.thread_scheduler_snapshot()  # 只读，不推进 guest
thread = emu.threads[100]                   # GuestThreadSnapshot
thread.frozen                               # bool：是否已被冻结

emu.threads.freeze(100)                     # 摘除调度资格
emu.threads.unfreeze(100)                   # 恢复调度资格
```

- `emu.threads[tid]` 返回 `GuestThreadSnapshot`（`tid` / `is_main` / `queue` / `state` /
  `waiter` / `deadline_ns` / `frozen`）；`thread_scheduler_snapshot()` 的每个 thread
  快照同样携带 `frozen`，两者都不执行 guest 指令、不改变 scheduler 状态。
- `emu.threads.freeze(tid)` / `unfreeze(tid)` 返回 `None`，只改 frozen overlay。`freeze`
  仅对 guest child thread 生效；**没有** `list` / `kill` / `drain` / `step` 等入口。
- 三个入口都只能在 **host 空闲**（两次 guest 调用之间）时调用：guest 正在执行，或在
  breakpoint / interceptor / syscall callback 内调用，都会报 `RuntimeError`；非 owner
  host thread、session Fatal、`close()` 之后同样是 `RuntimeError`。
- 参数错误：`bool` / `str` / `None` / `float` → `TypeError`；超出 `i32` 范围 →
  `OverflowError`；未知 tid、以及 host 空闲时已不存在的 main tid → `KeyError`（main task
  只在一次 root 调用存续期间存在，不假造）。
- **依赖被冻结线程的调度无进展是终止性错误**：当所有 runnable 消失且 snapshot 中存在
  frozen task（典型场景：root 在 `pthread_join` 一个被冻结的 child），coordinator 报
  `SchedulerDeadlock` 并把诊断记入 **Fatal**。此后该 session **不可恢复**——任何 guest
  入口与线程控制都立即 `RuntimeError`，只有 `close()` 仍可清理；需要继续运行请新建
  `Emulator`。不含 frozen 的普通 `SchedulerDeadlock` 不会把 session 置为 Fatal。

## 生命周期与限制

- `close()` 用于释放当前 emulator 资源。
- `init_test_memory()` 是测试/探针专用区域，不能当作目标 heap 或正式 process layout；
  它与 ELF 装载双向互斥，但与已就绪的 ART/JNI 不互斥。
- `jni_env_pointer`、`java_vm_pointer` 构造成功后必定非空，`close()` 后为 `None`。
- guest 执行 fault 仍然 fail-fast；crashdump 是诊断上下文，不是恢复机制。
