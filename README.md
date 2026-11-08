# rundroid

**在 Python 里运行 Android ARM64 native 代码。**

rundroid 是一个 Python 运行时。它把 `.so` 或 APK 里的 native 代码放进一个可控的 ARM64 guest，
让你用 Python 注册 Java shim、JNI、hook、断点和 trace，不需要真机。

适合这些场景：

- 逆向分析时复现某个 native 函数
- 在没有设备的环境里跑 ARM64 `.so`
- 用 Python 快速验证 JNI / Java 交互
- 给 native 代码加 breakpoint、watchpoint 和 trace

本仓库只提供**二进制和 Python 接口**，不包含 Rust 源码。

## 安装

Linux x86_64 + Python 3.13：

```bash
uv add "rundroid @ https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.5.1/rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl"
```

第一次运行前，把 guest 运行环境准备好：

```bash
rundroid-fetch-system-libs download
```

## 用起来

```python
from rundroid import AndroidEnvProfile, Emulator

profile = AndroidEnvProfile.from_mapping({
    "framework": {"system_properties": {"ro.product.model": "rundroid"}},
    "syscall": {"cwd": "/data/local/tmp"},
})

emu = Emulator("arm64", "unicorn", 42, profile=profile)
module = emu.load("libfoo.so")

print(emu.call("target_fn", 1, 2))
```

`load()` 返回一个 `ElfModule`。`call` 按导出符号调用，`call_address` 按绝对地址调用，
`invoke_native` 走完整的 JNI 参数编组。APK 作为构造参数传入：

```python
emu = Emulator("arm64", "unicorn", 42, profile=profile, apk="base.apk")
```

## 结构

![rundroid 结构](docs/diagrams/overview.svg)

## 包里有什么

![wheel 内容](docs/diagrams/wheel-contents.svg)

一个 wheel 里有编译好的 native 扩展、Python 接口、类型存根、命令行工具和 runtime lock。
guest `.so` 是独立资源，按需下载。

## 安装流程

![直链安装流程](docs/diagrams/install-flow.svg)

## 运行环境

![runtime assets 流程](docs/diagrams/runtime-assets-flow.svg)

## 平台

| 平台 | 状态 |
|---|---|
| Linux x86_64 · CPython 3.13 · glibc >= 2.34 | 可用（v0.5.1） |
| Linux aarch64 | 计划中 |
| macOS arm64 / x86_64 | 计划中 |

## 更多

- [安装与升级](docs/install.md)
- [wheel 内容与边界](docs/artifacts.md)
- [guest runtime assets](docs/runtime-assets.md)
- [发布流程](docs/release-process.md)（维护者）
- [仓库结构与同步策略](docs/repository-layout.md)

## Skill

```bash
npx skills add https://github.com/Lerr1uqs/rundroid-release/tree/main/skills/rundroid-python
```

skill 讲的是怎么用 rundroid 写 Python 脚本，不包含源码。

## 反馈

用的时候遇到问题、想要支持某个平台，或者有什么想法，都欢迎开 issue：

https://github.com/Lerr1uqs/rundroid-release/issues

带上 rundroid 版本、平台和能复现的命令，会更容易定位。

## 校验

每个 wheel 都附带同名 `.sha256`。v0.5.1：

```text
049769029fbc80c6f0bc92b0f6403658c97bcd9ab56e43b4ef6752cfcf6183f8
```

## 许可证

MIT，见 [LICENSE](LICENSE)。wheel 内的第三方组件见 dist-info 的 CycloneDX SBOM；
runtime image 的许可证见 ZIP 内的 `NOTICE/`。
