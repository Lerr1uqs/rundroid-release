# 资产元数据

这里的 TOML 只记录元数据：文件名、URL 和大小。二进制放在 GitHub Releases，不提交进 git。

| 文件 | 内容 |
|---|---|
| `v0.5.1.toml` | Linux x86_64 / CPython 3.13 wheel |
| `v0.5.0.toml` | 历史 wheel；resource CLI 缺少 runtime lock |
| `android-runtime-api33-arm64.toml` | guest API 33 ARM64 runtime image |

发版时新增一个 `v<version>.toml`，不要修改历史文件。
