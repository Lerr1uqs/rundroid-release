# Environment Profile 与 Virtualization

## 适用场景

需要模拟设备属性、补 system property/Settings、挂载虚拟文件或设备、安装 framework stub、
读取 APK，或替换 stdin/stdout/logcat 时查阅本篇。

## Profile

profile 是一次 session 的设备配置，在创建 emulator 时固定，运行期间不能热替换。

```python
profile = AndroidEnvProfile.from_mapping(
    {
        "framework": {
            "system_properties": {
                "ro.build.version.sdk": "35",
                "ro.product.model": "rundroid",
            },
            "settings": {"secure": {"android_id": "test-id"}},
        },
        "syscall": {
            "cwd": "/data/local/tmp",
            "pid": 1234,
            "uid": 2000,
            "uname": {"machine": "aarch64"},
        },
        "filesystem": {"root": "/tmp/guest-root"},
    }
)
```

- 未知 section/key、错误类型、范围错误、重复 property 都应立即报错。
- `None` 表示未配置/显式 absent，不能和空字符串或运行时默认值混淆。
- 当前支持 framework properties、secure settings、cwd/uid/euid/gid/egid/pid/ppid、
  uname、CPU mask、priority、stack rlimit 和 filesystem root。
- `filesystem` 只接受 `root`（rootfs 基线，默认 `cwd/.rundroid/rootfs`）与
  启动前冻结的 `writable_paths`（规范 guest 绝对目录，空集合表示全局只读）。
  overlay 由 Emulator 在内部 `.rundroid/overlay/session-*` 新建，不再作为公开参数；
  guest 写入只落在当前 session overlay，rootfs 基线不会被直接改写。
- 一次 emulator 的 guest 主线程身份必须来自冻结 profile，不要在补环境代码中硬编码
  `pid == 1`。

## 文件和设备补环境

`VirtualDevice` 子类实现 `open/read/write/close`，通过 `@device(path)` 声明路径；
`@file_node(path, kind=...)` 声明动态文件节点。decorator 只写 metadata，import 时不会
自动挂载。显式使用 `register(runtime, classes)` 或 `runtime.fs.map_device(...)`。

设备/文件 provider 只实现目标实际需要的读写语义，未覆盖的 op 会直接报错。不要在
Python provider 中自行维护文件描述符表。

## Framework 与 APK

- `Emulator` 构造期一次发布完整 builtin catalog；不再选择或安装 framework class group。
- 自定义 class/interface 经 `emu.art.register([...])` 注册；builtin 补充走
  `emu.art.extend([...])` + `JavaClassExtension` / `JavaInterfaceExtension`。
  替换已有 member 时还必须显式写 `override=True`。
- APK 是可选数据能力，不是 DEX VM 或 class 安装开关。只能作为构造参数
  `Emulator(..., apk=path)` 绑定，当前只处理单一 base APK、ARM64 native library、metadata、
  signature certificate、assets；不执行 DEX。构造后不存在绑定/替换入口。
- `emu.apk` 提供 metadata、assets 和 native library 查询。
- `Context.getSystemService` 返回按 service name 固定的精确 manager singleton；
  `ServiceManager` 返回独立的 binder identity，不要把两条 API 当成同一种对象。

## Stdio、logcat 与环境的关系

`set_stdin_source`、`set_stdout_sink`、`set_stderr_sink`、`set_logcat_sink` 都是实例级
注入，须在 guest execution 前完成。stdio source/sink 应符合明确 callable shape；空 bytes
表达 EOF。logcat sink 的 `emit(priority, tag, message)` 不应和 runtime human log 或
GuestEvent 混为同一类输出。
