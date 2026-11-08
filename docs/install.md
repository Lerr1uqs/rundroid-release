# 安装与升级

## 前置条件

| 项目 | 要求 |
|---|---|
| Host | Linux x86_64 |
| Python | CPython 3.13（`>=3.13,<3.14`） |
| glibc | >= 2.34（Ubuntu 22.04+ / RHEL 9+） |
| 包管理器 | `uv` |

当前 wheel tag 是 `cp313-cp313-manylinux_2_34_x86_64`。Python 3.12、Linux aarch64、
macOS 和 Windows 都需要另外构建的 wheel，不能直接安装这个产物。

## 直链安装

```bash
uv add "rundroid @ https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.5.1/rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl"
```

`uv add` 会把依赖名和 URL 分开写入：

```toml
[project]
dependencies = ["rundroid"]

[tool.uv.sources]
rundroid = { url = "https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.5.1/rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl" }
```

安装后验证：

```bash
uv run python -c "import rundroid, importlib.metadata as m; print(m.version('rundroid')); print(rundroid.Emulator)"
```

## 升级

直链是锁定式安装，没有 `latest`：

1. 从 [Releases](https://github.com/Lerr1uqs/rundroid-release/releases) 选择新版本；
2. 把命令中的 URL 换成新 wheel；
3. 重新执行 `uv add`。

```bash
uv add "rundroid @ https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.6.0/rundroid-0.6.0-cp313-cp313-manylinux_2_34_x86_64.whl"
```

`uv.lock` 会记录实际解析到的 URL 和版本。CI 使用 `uv sync --frozen` 时不会自行寻找新版本。

## 为什么没有 `latest` URL

wheel 文件名必须包含版本号（PEP 427），所以：

- `releases/latest/download/<文件名>` 只在该文件名恰好存在于 latest release 时有效；
- 一旦升版，旧文件名在 latest release 中不存在，URL 会 404；
- 发布固定名别名 wheel 会被 uv 的 filename/METADATA 版本一致性检查拒绝。

需要“跟随最新”时，应使用包索引（PEP 503）或 find-links 页面，而不是直链。

## 校验

```bash
curl -LO "https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.5.1/rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl"
curl -LO "https://github.com/Lerr1uqs/rundroid-release/releases/download/v0.5.1/rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl.sha256"
sha256sum -c rundroid-0.5.1-cp313-cp313-manylinux_2_34_x86_64.whl.sha256
```

也可以用 `gh` 通过 GitHub API 下载：

```bash
gh release download v0.5.1 -R Lerr1uqs/rundroid-release -p '*.whl'
```

## 常见问题

### `HTTP 504 Gateway Time-out`

GitHub release asset 的公共下载端点偶尔会返回 504。先重试，或改用 `gh release download`。
这通常是 GitHub/CDN 的临时问题，不是 wheel 损坏。

### `No matching distribution` / 版本不兼容

确认：

- `python --version` 是 3.13；
- `uname -m` 是 `x86_64`；
- `ldd --version` 的 glibc >= 2.34。

### 升级后还是旧版本

检查 `pyproject.toml` 的 `[tool.uv.sources]` 是否仍指向旧 URL，然后重新 `uv lock`。
