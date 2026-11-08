# 资产元数据

这里的 TOML 只记录**元数据**：文件名、URL、大小和 SHA-256。二进制放在 GitHub Releases，
不提交进 git。

| 文件 | 内容 |
|---|---|
| `v0.5.1.toml` | Linux x86_64 / CPython 3.13 wheel |
| `v0.5.0.toml` | 历史 wheel；resource CLI 缺少 runtime lock |
| `android-runtime-api33-arm64.toml` | guest API 33 ARM64 runtime image |

发版时新增一个 `v<version>.toml`，不要修改历史文件。

校验时使用 Release 上的同名 `.sha256`：

```bash
curl -LO "https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.5.1/rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl"
curl -LO "https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.5.1/rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl.sha256"
sha256sum -c rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl.sha256
```
