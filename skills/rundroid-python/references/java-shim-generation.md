# APK/DEX Java Shim Generation

## 入口与安装

`rjg` 是独立发行包 `rundroid-java-gen` 的离线 CLI，读取 class definition，输出可编辑
Python javashim。需要 Python 3.13；生成阶段无需 rundroid native extension、JDK 或 NDK。
生成源码由已安装 rundroid 的脚本环境消费。当前源码实现位于仓库
`tools/rundroid-java-gen/`，不要假定已经发布到 PyPI。

在仓库根目录直接运行，不必安装全局命令：

```sh
uv run --project tools/rundroid-java-gen rjg --help
uv run --project tools/rundroid-java-gen rjg -a app.apk -c com/example/NativeBridge -o shims.py
uv run --project tools/rundroid-java-gen rjg --dex classes.dex --prefix com/example/ --mode skeleton -o skeleton.py
```

也可 `uv tool install --python 3.13 ./tools/rundroid-java-gen`，之后使用 `rjg`。
项目环境要求 RTK 时给命令加相应 `rtk proxy` 前缀。

## 选择与模式

- `-a/--apk` 和 `--dex` 二选一；APK 支持单 base APK 的多个根目录 DEX，按数字编号扫描。
- 至少指定一个 `-c/--class` 或 `--prefix`，两者可重复、取并集。class 用
  `com/example/NativeBridge` 内部名，包前缀必须以 `/` 结尾。
- 父类/interface 须另行选中、已是 builtin，或用 `--provided-class com/example/Base`
  声明由用户提前提供。工具不自动扩大选择，也不把缺失父类改成 Object。
- 默认 `native` 模式生成选中类壳、关系和 native 方法；无 native 的选中类仍保留壳。
- `--mode skeleton` 另外生成构造器、普通方法与字段。字段是 `<clinit>` 执行前的
  encoded 初值或零值；工具不执行 `<clinit>`，也不恢复 Java 方法体。

例如 DEX 声明 `static native byte[] sign(byte[])`，输出成员为：

```python
@java_method('sign([B)[B', static=True, native=True)
def sign(arg0: JByteArray | None) -> JByteArray | None:  # type: ignore[misc]
    'native 声明；实际实现由 so 提供。'
    raise AssertionError("native body must not execute")
```

这是生成文件的成员节选，完整文件含 import、class decorator、来源摘要和发布列表。
Java 原名/descriptor 保留在 decorator；重载、混淆名、keyword 和构造器会获得稳定唯一的
Python 名称，不能假定 `<init>` 会生成 Python `__init__`。
普通方法与 Java 构造器的生成函数抛 `NotImplementedError`，应审查并补写需要执行的行为。
native body 的 AssertionError 应保留，真实调用走 native dispatch。

## 审查与显式注册

生成后先检查 class 选择、父类/interface、static/native flags、字段初值和生成诊断。
将生成文件放入用户脚本可导入的位置；补写构造器/普通方法后，显式发布：

```python
from shims import JAVA_CLASSES, JAVA_EXTENSIONS, REQUIRED_CLASSES

# 先确保 REQUIRED_CLASSES 中的 builtin / 用户外部依赖已经存在。
emu.art.register(JAVA_CLASSES)
emu.art.extend(JAVA_EXTENSIONS)
```

import 本身不注册、不创建 Emulator。`REQUIRED_CLASSES` 只声明依赖，不负责注册。
注册完成后，继续正常的装载 so、JNI_OnLoad、native 调用流程，参阅
[jni-native-invocation.md](jni-native-invocation.md)。

builtin 同名 definition 默认失败。只有明确需要扩展该 builtin 时，才同时指定选择器和
`--extension-class java/lang/String`；输出进入 `JAVA_EXTENSIONS`，走 `emu.art.extend`。
工具不自动设置 `override=True`。遇到既有成员冲突，按
[java-shim-authoring.md](java-shim-authoring.md) 的 extension 规则逐项审查。
`JAVA_CLASSES` 和 `JAVA_EXTENSIONS` 不可混用。

## 失败与能力边界

- 输出文件已存在时失败，不覆盖用户改过的文件；重新生成应使用新文件名并审查差异。
  参数错误退出 2，输入/生成错误退出 1，成功退出 0。
- 支持 little-endian 标准 DEX 035/037/038/039/040；不接受 041 container、CompactDex、
  ODEX、split APK、XAPK 或 AAB。非法摘要、字符串、索引、重复定义和资源超限会直接失败。
- skeleton 可以表达静态 primitive/string/null 常量；需要制造对象或数组初值时明确失败。
  可在只需 native 声明时选择 native 模式，它不输出字段。
- 框架类只有引用不代表 APK 含其 definition，不能依赖 rjg 恢复完整 Android framework。
- 当前只接受 APK/DEX；没有 audit report 输入、`metadata_audit()` 或
  `register_dex_metadata()` API，不解析运行日志生成声明。
- 生成声明减少手写 metadata，不保证目标 so 可以直接运行；仍需用户补环境和实现行为。

源码输出固定 UTF-8/LF，不含时间戳/绝对路径。详细输入上限、parser 决策和维护命令见
仓库 `tools/rundroid-java-gen/README.md`。builtin 目录当前对应 rundroid 0.5.0 / API33；
runtime authority 改动后需同步该目录并运行 catalog 一致性测试。
