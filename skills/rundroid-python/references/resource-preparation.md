# 外部资源准备

Rundroid 的源码仓库只保存资源 lock、manifest 约束和 fixture 源码，不保存下载后的 APK、
`.so` 或 runtime bundle。运行真实 ELF/APK/case 或完整测试前，分别检查下面两类资源。

> **使用发行 wheel 时**：只需运行 `rundroid-fetch-system-libs download`（或
> `rundroid-fetch-runtime-assets download`）。发行包不包含 test assets；本文其余的
> `uv run --no-project`、test assets 命令和源码仓库路径仅适用于源码开发。

| 资源 | Authority | 用途 |
|---|---|---|
| API33 ARM64 runtime image/system libs | `resources/runtime-assets.lock.toml` 固定 Release、manifest 与 SHA-256 | Android linker、bionic 和 runtime |
| test assets | `resources/test-assets.lock.toml` 固定 `rundroid-test` commit 与 manifest SHA-256 | Rust/Python 测试 fixture |

`system_libs` 是兼容命令名，与 `runtime_assets` 指向同一份 Android 13/API33 ARM64 runtime
image。test assets 是另一套独立资源；准备其中一套不会隐式准备另一套。

## 首次准备

从源码仓库根目录执行：

```bash
cd python
uv run --no-project python -m rundroid_resource_cli.system_libs download
uv run --no-project python -m rundroid_resource_cli.test_assets download
```

`--no-project` 让首次下载不依赖本地 PyO3 extension 或 editable build。不要手写 Release URL，
也不要把下载后的二进制复制进源码 `resources/`。

安装 wheel 或成功执行 bootstrap/maturin develop 后，也可以使用 console scripts：

```bash
rundroid-fetch-system-libs download
rundroid-fetch-runtime-assets download
rundroid-fetch-test-assets download
```

前两个命令是同一 runtime image 的兼容入口，不需要重复执行。

## 离线检查与路径

运行真实 case 或测试前优先执行无副作用 preflight：

```bash
cd python
uv run --no-project python -m rundroid_resource_cli.system_libs check
uv run --no-project python -m rundroid_resource_cli.test_assets check
uv run --no-project python -m rundroid_resource_cli.system_libs print-root
uv run --no-project python -m rundroid_resource_cli.test_assets print-root
```

`check` 不联网、不写入，验证 lock identity、manifest 和文件摘要；`print-root` 只打印当前
解析出的版本化缓存目录。只有确认资源缺失时才执行 `download`。

默认缓存根：

```text
Linux/macOS: $XDG_CACHE_HOME/rundroid/assets
未设置 XDG_CACHE_HOME: ~/.cache/rundroid/assets
Windows: %LOCALAPPDATA%/rundroid/assets
```

环境变量 `RUNDROID_ASSETS_DIR` 可显式覆盖缓存根，适用于离线 CI 或预填充缓存。runtime 与
test assets 在根目录下使用独立、由 version/commit 隔离的子目录。

## Bootstrap 与全量测试

```bash
uv run --no-project --python 3.13 python scripts/bootstrap_dev.py --check-only
uv run python scripts/full_test.py --keep-logs
```

`bootstrap_dev.py --check-only` 检查 host 工具链以及两类已下载资源，不执行下载或构建。
不带 `--check-only` 的 bootstrap 才会下载资源、同步依赖并构建 Python FFI。
`full_test.py` 会在测试 stage 前执行两类资源 `check`，但不会自动下载。

## 常见错误

### Runtime manifest、hash 或 system library 不匹配

- 症状：提示 runtime manifest 缺失、SHA-256 错误、runtime image 不完整或 guest path 未声明。
- 检查：运行 `system_libs check` 和 `system_libs print-root`。
- 修复：把 `print-root` 指向的损坏版本目录移到工作区外，再运行 `system_libs download`。
- 禁止：不要从 host `/system`、其他 Android 版本或旧 SDK23 bundle 手工拼 `.so`。

### Test asset 缺失、hash 不匹配或仍是 Git LFS pointer

- 症状：提示 test assets 不完整，或文件是约百字节的 Git LFS pointer。
- 检查：运行 `test_assets check` 和 `test_assets print-root`。
- 修复：确认 `git lfs` 可用，把损坏的 commit 缓存目录移到工作区外，再运行
  `test_assets download`。
- 禁止：不要把 fixture 缺失归因于 loader/JNI，也不要把缓存文件复制回源码仓库。

### 下载失败

- Runtime Release 失败时检查 GitHub 网络/代理后重试正式 `system_libs download`；不要绕过
  lock 使用临时 URL。
- Test assets 下载失败时确认 Git、Git LFS 和对 `rundroid-test` 的访问权限，再重试正式
  `test_assets download`。

### 缓存根或执行目录错误

- 若资源下载过但当前环境找不到，先检查 `RUNDROID_ASSETS_DIR`、`XDG_CACHE_HOME` 或
  `LOCALAPPDATA`，再用 `print-root` 对照实际目录。
- `python -m rundroid_resource_cli...` 的源码形式必须在仓库 `python/` 目录执行；在仓库根
  运行会因 package path 不正确而失败。已安装环境可直接使用 console script。

资源 CLI 的 public contract 变化时，同步核对 `README.md`、`python/pyproject.toml`、
`python/rundroid_resource_cli/`、两份资源 lock、bootstrap/full-test 脚本与本文。
