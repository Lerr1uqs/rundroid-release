# 仓库结构与同步策略

## 目录

```text
.
├── README.md
├── LICENSE
├── docs/
│   ├── install.md
│   ├── artifacts.md
│   ├── runtime-assets.md
│   ├── release-process.md
│   ├── repository-layout.md
│   └── diagrams/
│       ├── install-flow.svg
│       ├── wheel-contents.svg
│       ├── runtime-assets-flow.svg
│       └── release-process.svg
├── assets/
│   ├── README.md
│   ├── v0.5.0.toml
│   ├── v0.5.1.toml
│   └── android-runtime-api33-arm64.toml
├── openspec/
│   ├── specs/release-distribution/spec.md
│   └── changes/publish-binary-releases/
│       ├── proposal.md
│       ├── design.md
│       └── tasks.md
└── skills/
    └── rundroid-python/
```

## 为什么文档是自包含的

release 仓库会被单独 clone，也会被 GitHub 直接渲染。因此：

- **不要使用跨仓库符号链接**。GitHub 不会把指向 `../rundroid/...` 的链接解析成另一个仓库；
  单独 clone 后它们就是断链。
- **不要把主仓库设成子模块来“共享”文档**。子模块只能把本仓库挂到主仓库的某个目录下，
  不会让本仓库的 README/文档自动指向主仓库内容。
- 文档、图、openspec 和 skill 都应是本仓库内真实存在的文件。

## 如果想保持单一事实来源

推荐在**主仓库**里加一个同步脚本或 GitHub Action，在发版时把公开内容复制到本仓库：

```text
main repo: docs/release/**  +  skills/rundroid-python/**
        └── sync (CI / script) ──► rundroid-release: docs/**, skills/**
```

复制而不是 symlink，原因同上：消费者和 GitHub 看到的是真实文件。

## 子模块的适用场景

如果维护者希望在主仓库本地直接编辑 release 仓库，可以把本仓库作为子模块挂到主仓库：

```bash
git submodule add https://github.com/Lerr1uqs/rundroid-release.git release
```

它解决的是“本地同时 checkout 两个仓库”，**不解决** release 仓库单独 clone 时的文档完整性。
因此 release 仓库仍应自包含。

## skill 的发布

`skills/rundroid-python` 是面向使用者的 skill，不含源码，可以公开。本仓库保留一份副本：

```bash
npx skills add https://github.com/Lerr1uqs/rundroid-release/tree/main/skills/rundroid-python
```

如果主仓库的 skill 有更新，发版时重新复制并提交。注意：skill 里的源码仓库命令
（如 `uv run --no-project`）仅适用于开发；wheel 使用者直接使用 console scripts。

## 资产放在哪里

wheel、runtime ZIP 和 `.sha256` 放在 **GitHub Releases**，不提交进 git：

- 单文件上限 2 GiB；
- 避免仓库膨胀；
- 保持 release 资产的不可变性；
- `assets/*.toml` 只记录元数据（URL、大小、SHA-256），不保存二进制。
