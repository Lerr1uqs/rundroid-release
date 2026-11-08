# 发布流程

本文描述如何从源码仓库构建并发布一个二进制 wheel 到本仓库。发布只由维护者执行。

## 版本权威

`python/pyproject.toml` 的 `[project].version` 是 Python 包版本 authority。发版时：

1. 修改 `version`，例如 `0.5.1`；
2. 在 `python/` 执行 `uv lock`，让 `uv.lock` 的 editable 项目版本同步；
3. 构建产物版本必须与 release tag `v<version>` 一致。

Cargo workspace 的版本不决定 wheel 版本。

## 构建

```bash
cd python
uv run --no-sync maturin build --release --strip --compatibility manylinux_2_34 --out ../dist
```

关键参数：

| 参数 | 原因 |
|---|---|
| `--release` | 发布优化构建 |
| `--strip` | 去掉调试符号；`_rundroid.so` 从约 183 MB 降到约 31 MB |
| `--compatibility manylinux_2_34` | 二进制实际使用 `GLIBC_2.34` 符号，`manylinux_2_28` 会被 maturin 拒绝 |

产物：

```text
dist/rundroid-<version>-cp313-cp313-manylinux_2_34_x86_64.whl
```

## 构建后检查

```bash
python - <<'PY'
import zipfile, glob
p = sorted(glob.glob("dist/rundroid-*.whl"))[-1]
z = zipfile.ZipFile(p)
names = z.namelist()
for required in (
    "rundroid/py.typed",
    "rundroid/_rundroid.pyi",
    "rundroid_resource_cli/resources/runtime-assets.lock.toml",
):
    assert required in names, required
assert not any("test-assets.lock.toml" in n for n in names)
print(p, len(names), "entries")
PY
```

确认：

- `.so` 是 `stripped`；
- `_rundroid.pyi` 与 `py.typed` 存在；
- 只有 runtime lock，没有 `test-assets.lock.toml`；
- `METADATA` 的 `Version` 与文件名一致。

## 校验和

```bash
cd dist
sha256sum rundroid-<version>-cp313-cp313-manylinux_2_34_x86_64.whl \
  > rundroid-<version>-cp313-cp313-manylinux_2_34_x86_64.whl.sha256
```

## 干净环境验证

在全新临时项目中用**公共 URL**安装：

```bash
WORK=$(mktemp -d)
cd "$WORK"
uv init --no-workspace
uv add "rundroid @ https://github.com/Lerr1uqs/rundroid-release/releases/download/v<version>/rundroid-<version>-cp313-cp313-manylinux_2_34_x86_64.whl"
.venv/bin/python -c "import rundroid, importlib.metadata as m; print(m.version('rundroid'))"
.venv/bin/rundroid-fetch-system-libs print-root
```

如果公共端点临时 504，可先用 `gh release download` 拉取同一资产，校验 SHA-256 后在本地安装；
确认产物无误后再重试公共 URL。

## 上传

```bash
gh release create v<version> \
  -R Lerr1uqs/rundroid-release \
  --target main \
  --title "rundroid <version> (Linux x86_64)" \
  --notes-file /tmp/rundroid-v<version>-notes.md \
  dist/rundroid-<version>-cp313-cp313-manylinux_2_34_x86_64.whl \
  dist/rundroid-<version>-cp313-cp313-manylinux_2_34_x86_64.whl.sha256
```

## 不可变规则

- **不要覆盖同名资产**。uv 和 CI 会按 URL 缓存；同名不同字节会造成校验和/缓存问题。
- 修复发布问题时 **bump 版本**，例如 `0.5.1` → `0.5.2`，不要删除旧 tag 后重传。
- wheel release 使用 `v<version>` tag；runtime image 使用独立的 asset tag。不要把两类资产
  混在同一个 tag 里。
- 单文件上限 2 GiB；当前 wheel 约 9.7 MB，远低于限制。

## 未来平台

需要分别构建并上传：

| 平台 | wheel tag 示例 | 构建机 |
|---|---|---|
| Linux x86_64 | `cp313-cp313-manylinux_2_34_x86_64` | Linux（manylinux 容器更稳） |
| Linux aarch64 | `cp313-cp313-manylinux_2_34_aarch64` | Linux aarch64 / QEMU |
| macOS arm64 | `cp313-cp313-macosx_11_0_arm64` | macOS arm64 |
| macOS x86_64 | `cp313-cp313-macosx_10_12_x86_64` | macOS x86_64 |

不要把 `linux_x86_64` 裸 tag 的 wheel 当作跨平台产物。每个平台都需要独立验证。
