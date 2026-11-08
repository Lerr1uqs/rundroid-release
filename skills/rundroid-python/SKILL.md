---
name: rundroid-python
description: 帮助使用者安装和准备 Rundroid 资源，并编写 Python 仿真脚本；适用于 system libs/runtime assets/test assets 下载与缺失排障，以及 emulator、rjg APK/DEX 离线生成 Java shim、JNI、调试、guest 线程冻结控制和真实 ELF/APK case。
metadata:
  short-description: Rundroid resource preparation and Python case authoring
  version: "1.4"
---

# Rundroid Python

把 Rundroid 当成 Python 里的 Android ARM64 native 仿真运行时。使用者用它加载 ELF/APK、
构造 Java shim、注册 hook、补齐 Android 环境、执行 native，并观察 guest 行为。

## 使用边界

- 当前 guest 只支持 ARM64 + Android + Unicorn；host 支持 Windows/Linux/macOS 不等于
  guest 支持这些平台。
- breakpoint/watchpoint 支持两种 handler：继承 `Breakpoint`/`Watchpoint` 并覆盖
  `on_hit`，或直接传入 callable。breakpoint callable 的签名是
  `BreakpointHitCallback = Callable[[CpuContext, int], bool]`，watchpoint callable 的签名是
  `WatchpointHitCallback = Callable[[CpuContext, WatchpointEvent], bool]`；两者都必须返回
  真正的 `bool`。
- breakpoint/watchpoint、syscall 与 libc/native callback 中的 `CpuContext` 只在当前
  callback 内有效；机器调试统一通过 `cpu.cdt`，不要保存到 callback 外继续使用。
- 寄存器只有 typed 属性入口 `cpu.regs.xN` / `cpu.regs.wN`（外加 `lr/fp/ip0/ip1/sp/pc/
  tpidr_el0`）：`wN` 是 `xN` 低 32 位的 view，读零扩展、写清零高 32 位。旧的
  `cpu.reg(name)` / `cpu.set_reg(name, value)` 字符串入口已删除。
- Java interceptor callback 可以通过 `ctx.art` 读写 Java 状态，但没有 live CPU context
  或 `cdt`，也不能在 callback 中再调用 guest 函数或另一个 native execution。
- guest 线程冻结控制（`emu.threads`，需 `thread_scheduler=True`）只在 host 空闲、即
  两次 guest 调用之间可用；guest 执行中或 callback 内调用会直接报错。冻结是**摘除 guest
  调度资格**，不是挂起 host 后台线程；依赖被冻结线程的调度无进展会使 session 进入
  不可恢复的 Fatal。
- 参数错误、找不到符号、未实现的 guest 能力会直接报错；脚本应根据错误补环境或调整
  参数，不要依赖隐式降级。
- APK 数据只作为可选的构造配置 `Emulator(..., apk=path)` 绑定；ART/builtin/JNI 在构造期
  建立，Java declaration 直接经 `emu.art.register(...)` 注册，不存在手工 init 入口。
- 源码仓库不隐式下载或提交大型二进制；运行真实 ELF/APK/case 前，先用正式资源 CLI
  对 system/runtime image 与 test assets 分别执行只读 preflight。

## 按任务查阅

只读取当前任务需要的 reference，不要默认加载全部文档：

- emulator 初始化、加载 `.so`/APK、导出调用、`JNI_OnLoad`、native 调用、生命周期，或
  guest 线程查询 / 冻结控制（`emu.threads`）：
  读 [references/python-runtime.md](references/python-runtime.md)。
- 安装、首次环境准备、system libs/runtime assets/test assets 下载、缓存位置、离线校验或
  资源缺失排障：读 [references/resource-preparation.md](references/resource-preparation.md)。
- 编写 `JavaClass`、`JavaInterface`、构造器、方法、字段、重载或 Java 值包装：
  读 [references/java-shim-authoring.md](references/java-shim-authoring.md)。
- 从 APK/DEX 生成可编辑 Java 声明、使用 `rjg`、筛选 class/package 或处理生成诊断：
  读 [references/java-shim-generation.md](references/java-shim-generation.md)；补写行为和注册时
  结合 [references/java-shim-authoring.md](references/java-shim-authoring.md)。
- 查询 STATIC/DYNAMIC native、`RegisterNatives`、JNI ABI、参数编组或返回值：
  读 [references/jni-native-invocation.md](references/jni-native-invocation.md)。
- Java method、syscall 或 raw syscall 拦截：读
  [references/interception.md](references/interception.md)。
- breakpoint、watchpoint、trace、crashdump、内存、ELF 或 JNI inspection：读
  [references/debugging-and-inspection.md](references/debugging-and-inspection.md)。
- profile、system property、filesystem、virtual device、framework stub、stdio 或
  logcat 补环境：读 [references/environment-and-virtualization.md](references/environment-and-virtualization.md)。
  更细的 profile TOML/schema 说明可继续读 [references/android-profile.md](references/android-profile.md)。
- GuestEvent、runtime log、logcat/stdio 输出或 case CLI：读
  [references/observability-and-cli.md](references/observability-and-cli.md)。
- 编写完整的真实运行脚本：结合阅读
  [references/python-runtime.md](references/python-runtime.md) 和
  [references/observability-and-cli.md](references/observability-and-cli.md)。
