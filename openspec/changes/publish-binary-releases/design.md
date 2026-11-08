## Context

rundroid 的 Python 扩展由 PyO3 + maturin 构建。源码安装路径需要 Rust、cmake、ninja 和可用的
C/C++ 工具链；发布目标则是让使用者只安装二进制 wheel。当前 release 仓库只有 runtime image
ZIP 和一份占位 README，没有 Python 安装产物。

约束：

- 使用者继续通过 `uv add` 安装，不要求额外的私有索引服务。
- runtime image 已有不可变 lock 和 release URL 契约，不能因 wheel 发布而改变。
- 发布资产不可变；同名文件不覆盖。
- 文档必须在单独 clone release 仓库时可读。

## Goals / Non-Goals

**Goals:**

- 发布 Linux x86_64 / CPython 3.13 wheel，包含 native 扩展、Python 接口层、类型存根和 CLI。
- 修复 wheel 内 runtime lock 缺失，使 runtime CLI 在干净安装环境可用。
- 用直链 URL 提供锁定式安装，并在文档中解释其与 “latest” 的差异。
- 让 release 仓库自包含 README、文档、SVG、openspec 和公开 skill。

**Non-Goals:**

- 不发布 PyPI 包或 PEP 503 索引。
- 不构建 macOS / aarch64 / Windows wheel。
- 不把 runtime image 打进 wheel。
- 不使用跨仓库符号链接共享文档。
- 不改变 runtime lock/manifest schema。

## Decisions

### 直链 wheel URL，而不是包索引

直链满足最小可用目标：无需额外基础设施，`uv add` 能直接解析，`uv.lock` 固定产物。
代价是 URL 绑定版本和平台，升级需要替换 URL。需要“跟随最新”时再引入 PEP 503 索引或
find-links 页面。

### manylinux_2_34

实际二进制使用到 `GLIBC_2.34` 符号；maturin 拒绝 `manylinux_2_28`。因此发布 tag 使用
`manylinux_2_34_x86_64`，并在文档中声明最低 glibc。更宽的兼容性需要在较老的 manylinux
容器中重新构建和验证，不能仅改 tag。

### 发布构建 strip 调试符号

`[profile.release]` 保留 `debug = true` 供开发调试；发布构建使用 `maturin --strip`。
`_rundroid.so` 从约 183 MB 降到约 31 MB，wheel 约 9.7 MB。

### runtime lock 进 wheel，test lock 不进

`rundroid-fetch-system-libs` 与 `rundroid-fetch-runtime-assets` 是真实运行入口，需要
`runtime-assets.lock.toml`。`test-assets.lock.toml` 只服务于源码仓库测试 fixture。
包内副本放在 `rundroid_resource_cli/resources/`，并增加与源码 lock 的字节一致性测试，避免漂移。
`lock_path()` 优先源码 checkout，回退到包内路径。

### runtime image 保持独立资产

guest `.so` 约 3.3 MB，且已有独立的不可变 tag、lock 和下载链路。把它并入 wheel
会放大 wheel、破坏既有 lock 契约，也会让“仅 import SDK”的用户下载不必要的内容。

### release 文档自包含，不用符号链接

GitHub 和单独 clone 都不会解析跨仓库符号链接。release 仓库内的 README、文档、SVG、
openspec 和 skill 必须是真实文件。主仓库若需要单一事实来源，应通过同步脚本/CI 复制，
而不是 symlink；子模块只适合维护者在主仓库本地同时 checkout 两个仓库。

## Risks / Trade-offs

- **只有 Linux x86_64**：aarch64 / macOS 用户暂时不可用；需要独立构建机。
- **glibc 2.34 基线**：更老的发行版不能安装；需要 manylinux 容器重建才能放宽。
- **直链更新成本**：升级需要替换 URL；未来应补充索引。
- **包内 lock 副本漂移**：通过字节一致性测试约束。
- **GitHub release-assets 端点偶发 504**：公共下载偶发失败；文档建议重试或用 `gh release download`。
