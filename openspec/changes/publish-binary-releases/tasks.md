## 1. wheel 构建与打包

- [x] 1.1 版本 bump 到 `0.5.1` 并同步 `python/uv.lock`。
- [x] 1.2 用 `maturin build --release --strip --compatibility manylinux_2_34` 构建 wheel。
- [x] 1.3 把 `runtime-assets.lock.toml` 作为包数据打入 `rundroid_resource_cli/resources/`。
- [x] 1.4 确认 wheel 不含 `test-assets.lock.toml`，且 `_rundroid.pyi`、`py.typed` 存在。

## 2. 安装环境 lock 定位

- [x] 2.1 `lock_path()` 支持源码 checkout 优先、包内 resources 回退。
- [x] 2.2 新增包内 lock 与源码 lock 的字节一致性测试。
- [x] 2.3 新增源码缺失时回退到包内 lock 的测试。
- [x] 2.4 `rundroid-fetch-test-assets` 在发行包环境输出“仅源码测试”提示。

## 3. 验证

- [x] 3.1 干净 uv 项目通过直链安装，`import rundroid` 与版本检查通过。
- [x] 3.2 `rundroid-fetch-system-libs print-root` 在安装环境可读取包内 lock。
- [x] 3.3 用本地 TLS 镜像验证 `download → SHA-256 → 解压 → check` 全链路。
- [x] 3.4 发布前确认 `.so` 已 strip，wheel 约 9.7 MB。

## 4. 发布

- [x] 4.1 发布 v0.5.0（Linux x86_64）。
- [x] 4.2 发布 v0.5.1（修复 runtime lock 打包），上传 wheel 与 `.sha256`。
- [x] 4.3 用 `gh release download` 校验远端字节与本地构建一致。

## 5. 发行仓库文档

- [x] 5.1 重写 README：安装、平台、产物、runtime、校验、skill。
- [x] 5.2 新增安装、产物、runtime assets、发布流程、仓库结构文档。
- [x] 5.3 生成 4 张 SVG：直链安装、wheel 内容、runtime 流程、发布流程。
- [x] 5.4 新增 `release-distribution` spec 与 `publish-binary-releases` change。
- [x] 5.5 新增资产元数据 `assets/*.toml`。
- [x] 5.6 公开 `rundroid-python` skill 副本并标注 wheel 用法。

## 6. 待办

- [ ] 6.1 构建并发布 Linux aarch64 wheel。
- [ ] 6.2 构建并发布 macOS arm64 / x86_64 wheel。
- [ ] 6.3 评估 PEP 503 索引以支持“跟随最新”安装。
- [ ] 6.4 增加发版同步脚本或 GitHub Action，自动生成 wheel 与资产元数据。
