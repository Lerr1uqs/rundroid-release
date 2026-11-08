## Why

使用者需要在**没有 Rust、cmake、ninja 工具链**的情况下把 rundroid 当作 Python 运行时使用。
当前主仓库只提供源码安装（`uv add git+ssh://...`），会在本机通过 maturin 构建 PyO3 扩展；
这既要求源码可见，也把构建复杂度转移给了使用者。

需要一条只发布二进制的路径：wheel 包含编译扩展、Python 接口层和类型存根，使用者用
`uv add "<wheel URL>"` 即可安装。

## What Changes

- 在 `Lerr1uqs/rundroid-release` 发布 Linux x86_64 / CPython 3.13 的 `rundroid` wheel：
  `rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl`。
- 修正 wheel 的 runtime 资源打包：把 `runtime-assets.lock.toml` 作为包数据放入
  `rundroid_resource_cli/resources/`，使 `rundroid-fetch-system-libs` 在没有源码 checkout 的
  环境也能工作；不打包 `test-assets.lock.toml`。
- `lock_path()` 在源码 checkout 缺失时回退到包内 lock；新增包内 lock 与源码 lock 的字节一致性测试。
- `rundroid-fetch-test-assets` 在发行包环境给出“仅供源码仓库测试”的明确提示。
- 版本 bump 到 `0.5.1`，同步 `python/uv.lock`。
- 在 release 仓库补齐 README、安装/产物/runtime/发布流程文档、4 张 SVG 图、资产清单、
  openspec spec/change，以及可公开的 `rundroid-python` skill 副本。

## Capabilities

### New Capabilities

- `release-distribution`：二进制发行产物的边界、不可变性、平台标签、直链安装和
  文档自包含性。

### Modified Capabilities

- `runtime-resource-cli`：wheel 安装环境从包内 resources 读取 runtime lock；test-assets lock
  不进入发行包。

## Impact

- 主仓库：`python/pyproject.toml`（版本、maturin include）、
  `python/rundroid_resource_cli/common.py`（lock 路径回退）、
  `python/rundroid_resource_cli/test_assets.py`（发行包提示）、新增测试与包内 lock 副本。
- 发行仓库：文档、图、openspec、skill、资产元数据。
- 使用者：从“需要编译源码”变为“下载 wheel 并显式准备 runtime image”。
- 不改变 Rust 资源加载契约、guest 运行行为或现有 lock schema。
