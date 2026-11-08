# guest runtime assets

## 这是什么

`android-runtime-api33-arm64` 是 guest 侧运行环境：Android 13 / API 33、ARM64 的
`/system/lib64`、APEX bionic 与相关 `.so`。它是独立于 wheel 的 release 资产，ZIP 约 3.3 MB。

真实 ELF/APK 运行前需要它；仅仅 `import rundroid` 不需要。

## 准备

```bash
# 首次使用前：下载、校验并安装
rundroid-fetch-system-libs download

# 等价入口
rundroid-fetch-runtime-assets download
```

`check` 与 `print-root` 不联网、不写入：

```bash
rundroid-fetch-system-libs check
rundroid-fetch-system-libs print-root
```

只有 `download` 会联网并写入缓存。最终目录完成全量校验后才会原子发布；已存在但损坏的目录
不会被覆盖。

## 缓存位置

```text
Linux/macOS: $XDG_CACHE_HOME/rundroid/assets
未设置 XDG_CACHE_HOME: ~/.cache/rundroid/assets
Windows: %LOCALAPPDATA%/rundroid/assets
```

可显式覆盖：

```bash
RUNDROID_ASSETS_DIR=/path/to/cache rundroid-fetch-system-libs download
```

runtime 与 test 资源使用独立的版本化子目录和 lock，互不隐式准备。

## 版本与校验

wheel 内的 `rundroid_resource_cli/resources/runtime-assets.lock.toml` 固定：

- `asset_id`: `android-runtime-api33-arm64`
- `version`: `TQ1A.230205.001.D2-9471403`
- `release_url`: 本仓库对应 release 的 ZIP
- `bundle_sha256`: `927b47236dfb91d580d98ab1e8293f4cbdb9aa430020da504692cdc8f6c62a5e`
- `manifest_sha256`: `6b6cff8c8b2a82a4e95018e8c9ec1f31eac27039e6b593fb04a28c10d05cdeeb`

下载流程会校验 ZIP 的 SHA-256，再按 manifest 校验每个文件，最后把完整目录原子替换到缓存。

## `update` 是开发工具

`rundroid-fetch-system-libs update` 只用于在源码仓库本地重算 runtime manifest 与 lock，
需要本地 payload 和 `llvm-readelf`。发行包使用者不应运行它。

## 离线 CI

预先填充缓存后，用 `RUNDROID_ASSETS_DIR` 指向该目录，然后只运行 `check`：

```bash
RUNDROID_ASSETS_DIR=/opt/rundroid-assets rundroid-fetch-system-libs check
```

`check` 不会发起网络请求，也不会修改缓存。
