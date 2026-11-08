# release-distribution Specification

## Purpose

定义 rundroid 二进制发行仓库的发布契约：wheel 与 runtime image 的产物边界、平台标签、
校验和、安装方式，以及文档的自包含性。源码仓库不发布 Rust 源码，本仓库只发布可安装产物。

## Requirements

### Requirement: Release artifacts are immutable and versioned

每个发布资产 SHALL 由不可变 URL 定位，URL SHALL 包含版本或 asset tag。已发布资产的字节
MUST NOT 被同名覆盖；修复发布缺陷时 SHALL 递增版本。

#### Scenario: Published wheel is never overwritten

- **WHEN** 维护者需要修复一个已发布 wheel 的打包问题
- **THEN** 维护者 SHALL 递增 `python/pyproject.toml` 的版本并发布新 tag
- **AND** SHALL NOT 删除或覆盖旧 tag 的同名资产

#### Scenario: Direct URL encodes version and platform

- **WHEN** 使用者通过 `uv add "<wheel URL>"` 安装
- **THEN** URL SHALL 包含版本、Python ABI 和平台 tag
- **AND** `uv.lock` SHALL 固定该 URL 对应的产物

### Requirement: Wheel is the complete Python install artifact

wheel SHALL 包含编译后的 native 扩展、Python 接口层、类型存根、runtime 资源 CLI 与
runtime lock；MUST NOT 包含 Rust 源码。

#### Scenario: Import and typing work without a Rust toolchain

- **WHEN** 使用者在干净环境安装 wheel 并执行 `import rundroid`
- **THEN** native 扩展 SHALL 可直接加载
- **AND** `rundroid/_rundroid.pyi` 与 `rundroid/py.typed` SHALL 提供类型信息
- **AND** 安装过程 SHALL NOT 要求 Rust、cmake 或 ninja

#### Scenario: Wheel does not expose source

- **WHEN** 使用者解压 wheel
- **THEN** 其中 SHALL 只有编译产物和 Python 接口层
- **AND** SHALL NOT 包含 Rust 源码

### Requirement: Guest runtime image is a separate immutable asset

Android API 33 ARM64 runtime image SHALL 作为独立 release 资产发布，并由 wheel 内的
`rundroid-fetch-system-libs` / `rundroid-fetch-runtime-assets` 显式下载。

#### Scenario: Wheel installation does not download guest binaries

- **WHEN** 使用者只执行 `uv add`
- **THEN** wheel 安装 SHALL NOT 隐式下载 guest `.so`
- **AND** 使用者需要显式运行 `rundroid-fetch-system-libs download`

#### Scenario: Runtime image integrity is verified before publication

- **WHEN** `download` 安装 runtime image
- **THEN** CLI SHALL 校验 bundle SHA-256、manifest SHA-256 和逐文件摘要
- **AND** 只有全部校验通过后才原子发布缓存目录

### Requirement: Wheel embeds runtime lock but excludes test fixtures

wheel SHALL 打包 `runtime-assets.lock.toml`，使真实运行所需的 runtime CLI 在安装环境中可用；
MUST NOT 打包 `test-assets.lock.toml` 或任何测试 fixture。

#### Scenario: Runtime CLI works from an installed wheel

- **WHEN** 使用者在没有源码 checkout 的环境运行 `rundroid-fetch-system-libs print-root`
- **THEN** CLI SHALL 从包内 `rundroid_resource_cli/resources/` 读取 runtime lock
- **AND** SHALL 输出版本化缓存目录

#### Scenario: Test asset CLI is explicitly development-only

- **WHEN** 使用者在发行包环境运行 `rundroid-fetch-test-assets`
- **THEN** CLI SHALL 提示该命令仅供源码仓库测试使用
- **AND** SHALL NOT 暗示 wheel 包含测试 fixture

### Requirement: Platform and ABI tags are explicit

每个 wheel SHALL 使用与构建产物一致的 Python ABI、平台和 libc tag。发行范围当前只包含 Unix；
其他平台 MUST 单独构建、单独验证、单独发布。

#### Scenario: Linux x86_64 wheel declares glibc requirement

- **WHEN** 维护者发布 Linux x86_64 wheel
- **THEN** wheel tag SHALL 反映实际使用的 libc 基线
- **AND** 文档 SHALL 说明最低 glibc 版本

#### Scenario: Unsupported platform fails clearly

- **WHEN** 使用者在 Python 3.12、aarch64 或 macOS 上安装当前 wheel
- **THEN** 包管理器 SHALL 报告没有兼容的 wheel
- **AND** 文档 SHALL 说明需要等待对应平台构建

### Requirement: Checksums accompany every binary asset

每个 wheel 与 runtime ZIP SHALL 提供 SHA-256 校验值。wheel 的校验值 SHALL 以同名
`.sha256` 资产发布；runtime 的校验值 SHALL 固定在 lock 中并可在文档中复核。

#### Scenario: User verifies a downloaded wheel

- **WHEN** 使用者下载 wheel 和 `.sha256`
- **THEN** `sha256sum -c` SHALL 通过
- **AND** 文档 SHALL 给出该版本的摘要

### Requirement: Direct URL installation is pinned, not "latest"

直链安装 SHALL 被视为锁定式安装。文档 MUST NOT 宣称直链会跟随最新版本；需要“跟随最新”时
SHALL 使用包索引或 find-links 页面。

#### Scenario: Upgrade replaces the URL

- **WHEN** 使用者升级到新版本
- **THEN** 使用者 SHALL 用新版本的 wheel URL 重新执行 `uv add`
- **AND** `uv.lock` SHALL 更新为新的不可变 URL

### Requirement: Release documentation is self-contained

本仓库的 README、文档、图和 openspec MUST 以真实文件存在，MUST NOT 依赖跨仓库符号链接或
子模块才能阅读。

#### Scenario: Clone the release repository alone

- **WHEN** 使用者只 clone `rundroid-release`
- **THEN** README 中的所有相对链接 SHALL 可解析
- **AND** 图和文档 SHALL 可直接渲染

### Requirement: Public skill is distributable without source

面向使用者的 `rundroid-python` skill MAY 在本仓库公开；它 MUST 只描述使用方式，不包含源码。

#### Scenario: Install the skill from the release repository

- **WHEN** 使用者执行
  `npx skills add https://github.com/Lerr1uqs/rundroid-release/tree/main/skills/rundroid-python`
- **THEN** skill SHALL 提供 Python 使用方式、资源准备、JNI、调试和 trace 指引
- **AND** SHALL NOT 要求访问 Rust 源码
