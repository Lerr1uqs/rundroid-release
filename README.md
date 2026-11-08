# rundroid

**用 Python 驱动 Android ARM64 native 代码，在同一个脚本里补环境、执行、观察和改写行为。**

rundroid 把 APK 或 `.so` 放进可控的 Android guest：Rust 负责 ARM64 执行、ELF 链接、Android syscall 与 JNI；Python 负责 Java shim、设备环境、拦截和调试。你可以围绕一个 native 入口反复运行分析脚本，不必为每次验证准备真机或完整 Android 系统。

适合复现 JNI 算法、分析依赖 Android 环境的 `.so`、定位 native 与 Java 交互，以及把一次性逆向步骤整理成可重复运行的 Python 脚本。

## 它怎样工作

![Python 控制层经 PyO3 驱动 Rust 执行核心，装载并运行 Android ARM64 guest](docs/diagrams/overview.svg)

你的脚本给出输入和环境，rundroid 在 guest 中完成 `load → JNI_OnLoad → native call`，再把返回值、事件和调试现场交还给 Python。Java 方法可以由 Python shim 提供；guest 调用的 libc、syscall 和内存访问也有对应的观察或拦截入口。

## 能做什么

| 能力 | 对分析工作的价值 |
| --- | --- |
| **装载真实 Android native 库** | 加载独立 `.so` 或 base APK 中的 ARM64 库，处理 ELF 依赖、重定位、constructor 和 guest `dlopen`；可选择执行 `JNI_OnLoad`。 |
| **从 Java 声明调用 native** | 通过 JNI descriptor 查找 `RegisterNatives` 或 `Java_*` 入口；`invoke_native` 自动处理 `JNIEnv*`、对象引用、参数编组和返回值。 |
| **用 Python 补 Java 行为** | 声明 Java class、方法和字段，为已有 framework 类扩展成员，或临时 hook、override、supplement 一个 Java 方法。无需运行 DEX 字节码。 |
| **给目标构造设备环境** | 配置系统属性、Settings、进程身份和文件系统；提供虚拟文件/设备、stdio 与 logcat，让目标读到脚本指定的值。 |
| **在执行现场观察和改写** | 下断点、监听读写、拦截 syscall/libc 入口；在回调中查看寄存器、内存、栈、符号与 JNI 对象，按需修改执行状态。 |
| **控制 guest 线程** | 按需启用协作式线程调度，查询并冻结 guest 子线程，围绕多线程 native 路径定位问题。 |
| **留下可分析的证据** | 按范围或符号采集指令 trace，筛选 guest 事件；执行出错时查看 crashdump。固定 seed 便于复现依赖随机数的路径。 |

这些能力可以组合在一条脚本里：先补齐目标依赖的 Java/Android 环境，再装载库、命中断点、调整输入，最后检查 native 返回值和 trace。

## 最小调用链

下面的 `libfoo.so` 和 `target_fn` 是你自己的 ARM64 库与导出符号。运行时会在创建 session 时固定 `profile`，入口参数错误会直接报错，便于定位缺失的环境或不匹配的签名。

```python
from rundroid import AndroidEnvProfile, Emulator

profile = AndroidEnvProfile.from_mapping({
    "framework": {"system_properties": {"ro.product.model": "rundroid"}},
    "syscall": {"cwd": "/data/local/tmp"},
})

emu = Emulator("arm64", "unicorn", 42, profile=profile)
module = emu.load("libfoo.so")

# 如果目标导出 JNI_OnLoad，先执行 emu.jni_onload(module)。
print(emu.call("target_fn", 1, 2))
```

针对 JNI native，可在装载并执行 `JNI_OnLoad` 后查询 Java 签名，再让 rundroid 完成 JNI 参数编组：

```python
emu.jni_onload(module)
native = emu.jni.find_native(
    java_class="com/example/Signer",
    java_method="sign([B)[B",
)
result = emu.invoke_native(native, b"payload")
```

第二段适用于实际导出 `JNI_OnLoad` 并注册了该方法的库。Java shim 与断点的完整写法见 [Python 使用 Skill](skills/rundroid-python/SKILL.md)。

## 安装

当前提供 **Linux x86_64 · CPython 3.13 · glibc ≥ 2.34** 的 wheel：

```bash
uv add "rundroid @ https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.5.1/rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl"
rundroid-fetch-system-libs download
```

第二条命令准备锁定版本的 Android 13 / API 33 ARM64 guest runtime。wheel 包含 Rust 扩展、Python API、类型存根与资源工具；guest 系统库按需下载。安装条件、缓存位置和升级方法见 [安装指南](docs/install.md)。

| Host 平台 | 发行状态 |
| --- | --- |
| Linux x86_64 · CPython 3.13 | 已有 v0.5.1 wheel |
| Linux aarch64、macOS、Windows | 尚无对应 wheel |

当前 guest 固定为 **Android ARM64 / API 33**。APK 提供 native 库、manifest、签名证书和 assets；rundroid 不执行 DEX 字节码，也不是完整 Android 系统。单一 base APK 可在构造 `Emulator(..., apk="base.apk")` 时绑定。

## 深入使用

- [Python 使用 Skill](skills/rundroid-python/SKILL.md)：从 Java shim、JNI 到拦截、调试和真实脚本的任务索引。
- [安装与升级](docs/install.md) · [wheel 内容](docs/artifacts.md) · [guest runtime 资源](docs/runtime-assets.md)。
- [源码仓库](https://github.com/Lerr1uqs/rundroid) · [反馈问题](https://github.com/Lerr1uqs/rundroid-release/issues)。

也可以安装 Skill，让支持 Skill 的工具按 rundroid 的公开 API 辅助写脚本：

```bash
npx skills add https://github.com/Lerr1uqs/rundroid-release/tree/main/skills/rundroid-python
```

## 许可证

MIT，见 [LICENSE](LICENSE)。
