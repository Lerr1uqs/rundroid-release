# wheel 内容与边界

## 一个 wheel 覆盖三件事

| 需求 | wheel 内路径 |
|---|---|
| 二进制 | `rundroid/_rundroid.cpython-313-x86_64-linux-gnu.so` |
| Python 接口 / stub 层 | `rundroid/*.py` |
| 类型存根（SDK） | `rundroid/_rundroid.pyi`、`rundroid/py.typed` |
| runtime 资源 CLI | `rundroid_resource_cli/` + console scripts |
| runtime 资源锁 | `rundroid_resource_cli/resources/runtime-assets.lock.toml` |

v0.5.1 的 wheel 约 9.7 MB。native `.so` 解压后约 31 MB，构建时已 strip 调试符号。

## 包含

```text
rundroid/
  __init__.py
  _rundroid.cpython-313-x86_64-linux-gnu.so
  _rundroid.pyi
  py.typed
  apk.py
  art.py
  args_parser.py
  breakpoint.py
  cpu_context.py
  drivers.py
  elf_inspector.py
  emulator.py
  env_profile.py
  jni.py
  javashim/
  memory_inspector.py
  output.py
  syscall.py
  threads.py
  watchpoint.py
  ...
rundroid_resource_cli/
  __init__.py
  common.py
  runtime_assets.py
  runtime_bundle.py
  runtime_update.py
  adb_runtime_extract.py
  system_libs.py
  test_assets.py
  resources/runtime-assets.lock.toml
```

## 不包含

- **test fixtures**。`test-assets.lock.toml`、测试 APK、测试 `.so` 不在 wheel 中。
- **runtime image ZIP**。guest `/system/lib64/*.so` 等通过 `rundroid-fetch-system-libs download`
  单独下载。
- **其他平台二进制**。当前只有 Linux x86_64 / CPython 3.13。

## 为什么 runtime lock 在 wheel 里，test lock 不在

`rundroid-fetch-system-libs` 与 `rundroid-fetch-runtime-assets` 是同一实现，真实运行需要
`runtime-assets.lock.toml` 来确定 runtime image 的 release URL 和版本。

`test-assets.lock.toml` 只服务于源码仓库的测试 fixture（来自 `rundroid-test` Git LFS），
普通使用者不需要。发行包只打包 runtime lock。

## SBOM 与第三方组件

- wheel 的 `rundroid-0.5.1.dist-info/sboms/` 下有 CycloneDX SBOM，列出 native 依赖。
- runtime image ZIP 的 `NOTICE/` 包含 guest 组件的许可证信息。
- 本仓库的文档与 Python 接口层使用 MIT，见 [LICENSE](../LICENSE)。
