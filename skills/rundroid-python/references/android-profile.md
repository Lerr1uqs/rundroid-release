# Android 环境 Profile

## 适用场景

目标代码读取 `Build`、`Settings.Secure`、system property、进程身份或当前工作目录时，
使用 `AndroidEnvProfile` 描述这台虚拟 Android 设备。

每个 `Emulator` 都必须传入 profile。profile 在 session 创建时固定，运行期间不能热替换；
需要另一套设备环境时，创建新的 profile 和 emulator。

## 最小配置

```python
from rundroid import AndroidEnvProfile, Emulator

profile = AndroidEnvProfile.from_mapping(
    {
        "framework": {
            "system_properties": {
                "ro.product.model": "Pixel 7",
                "ro.build.version.sdk": "35",
            },
        },
    }
)
emu = Emulator("arm64", "unicorn", 42, profile=profile)
```

`framework.system_properties` 至少需要一个属性。属性值必须是字符串；需要表达属性
不存在时，可以使用 `absent_system_properties`。

## 完整配置

```python
profile = AndroidEnvProfile.from_mapping(
    {
        "framework": {
            "system_properties": {
                "ro.product.model": "rundroid",
                "ro.build.version.sdk": "35",
            },
            "settings": {
                "secure": {
                    "android_id": "test-device-id",
                },
            },
        },
        "syscall": {
            "cwd": "/data/local/tmp",
            "uid": 2000,
            "gid": 2000,
            "pid": 1234,
            "ppid": 1,
            "uname": {
                "sysname": "Linux",
                "release": "5.15.0-rundroid",
                "machine": "aarch64",
            },
            "cpu_mask": 1,
            "priority": 0,
            "rlimit_stack": 8388608,
        },
        "filesystem": {
            "root": "/tmp/rundroid-guest-root",
        },
    }
)
```

可配置内容包括：

- `framework.system_properties`、`framework.absent_system_properties`
- `framework.settings.secure`
- `syscall` 下的 cwd、uid/euid、gid/egid、pid/ppid、uname、CPU mask、priority 和 stack
  limit
- `filesystem.root`
- 可选的 `process_files.status` 与 `network.interfaces`，用于同一会话的进程文件和网卡身份

`pid` 会影响 guest 看到的主线程身份；需要模拟特定进程时，应在 profile 中显式填写，
不要在脚本其他位置再硬编码另一份 pid。

## 进程文件与网卡身份

```python
profile = AndroidEnvProfile.from_mapping({
    "framework": {"system_properties": {"ro.build.version.sdk": "33"}},
    "syscall": {"pid": 1234, "uid": 2000, "gid": 2000},
    "process_files": {"status": {"name": "rundroid", "tracer_pid": 0}},
    "network": {"interfaces": {"wlan0": {"mac": "02:42:ac:11:00:02"}}},
})
mac = profile.network.interfaces["wlan0"].mac  # Java extension 使用同一输入
```

启用后，Kernel 自动挂载 `/proc/self/status` 与 `/sys/class/net/wlan0/address`。status 的
PID/UID/GID 来自冻结的 syscall profile；`Threads` 在每次底层 read 时采样 guest 线程数。
未声明的路径仍返回 ENOENT。`fstat` 对动态 status 报告 `st_size=0`；它仍可正常读取。
`network` 只提供网卡身份数据，Java `NetworkInterface`、`WifiInfo` 的权限和隐私策略
由具体 Java 实现决定，不能假定它们一定返回文件里的 MAC。

## TOML 文件

`from_toml()` 使用与 mapping 相同的结构：

```toml
[framework.system_properties]
ro.product.model = "rundroid"
ro.build.version.sdk = "35"

[framework.settings.secure]
android_id = "test-device-id"

[syscall]
cwd = "/data/local/tmp"
pid = 1234

[clock]
mode = "anchored_host"
# 可选：覆盖 anchored_host 初值；省略时回落 Rust core 的冻结默认 epoch / start。
# realtime_epoch_ns = 1700000000000000000
# monotonic_start_ns = 0

[filesystem]
root = "/tmp/rundroid-guest-root"
```

```python
profile = AndroidEnvProfile.from_toml("device.toml")
emu = Emulator("arm64", "unicorn", 42, profile=profile)
```

旧版脚本使用的顶层 `[system_properties]` 和 `[settings]` 结构仍可读取，但新脚本优先
使用 `[framework]` 分组。

## 校验与缺失值

- 未知 section、未知字段、空 property key、错误类型和数值越界会直接报错。
- profile 中的 `None` 表示没有提供该项，不等同于空字符串。
- `[clock]` 默认 `mode = "anchored_host"`（Kernel 唯一时钟 authority：固定 epoch 锚定，
  两域随宿主真实 elapsed 同步推进；读取时钟不改变时间值）。`mode = "host"` 表示直接使用
  宿主 wall clock / elapsed，且不得同时提供 `realtime_epoch_ns` / `monotonic_start_ns`
  （构造期报错）。旧 `deterministic` 已移除并给出迁移提示，不作为兼容别名。
  结构化读取使用 `emu.clock.snapshot()`、`cpu.clock.snapshot()` 或
  `JavaMethodContext.clock.snapshot()`；`emu.clock_summary` 仅作只读诊断。
- guest `nanosleep` / futex timeout 在无其它 runnable task 时真实等待宿主到 deadline；
  调试脚本会因此实际耗时变长，时间只保证同源、单调和 deadline 关系，不承诺绝对重现。
- 缺少 system property 时，依赖该 property 的 Java/native 读取可能直接报错。
- `Settings.Secure.getInt` 可以使用调用方提供的 default；其他缺失环境不应假定会自动
  补齐。
