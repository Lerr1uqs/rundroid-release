# Guest Event、Logging 与 CLI

## 适用场景

需要筛选 guest 输出、保存事件、接收 logcat/stdio，或给脚本增加统一命令行参数时查阅
本篇。

## 三条输出线

```text
GuestEvent       guest 行为的顺序文本事件，可按 Event 过滤
Runtime log      host runtime 的人类诊断日志，可设 level/terminal
Android logcat   guest __android_log_* 的 sink
```

它们职责正交。不要用 runtime log 替代 GuestEvent，也不要把 telemetry 或 fail-fast
退出报告伪装成 GuestEvent。

## GuestEvent facade

入口是 [python/rundroid/output.py](/mnt/f/coding-workspace/rundroid/python/rundroid/output.py)：

```python
from rundroid import Event

emu.guest_event.enable([Event.JNICALL, Event.SYSCALL])
emu.guest_event.disable([Event.STDIO])
emu.guest_event.to_file("guest-events.txt")
```

当前事件类型包括 `JNICALL`、`SYSCALL`、`COMMAND`、`ERROR`、`LOGCAT`、`STDIO`、
`SYSPROP`。每个元素都必须是 `Event`，事件列表会整体应用；不要部分应用
一个非法 sequence。

## Runtime log 与 sink

- `emu.log.set_level("off" | "fatal" | "error" | "info" | "debug")` 只改变 runtime
  human log。
- `emu.log.to_console(on)` 只改变终端镜像，不改变 telemetry 或 logcat。
- `emu.set_logcat_sink(sink)` 替换 guest logcat sink。
- `set_stdin_source` / `set_stdout_sink` / `set_stderr_sink` 属于 stdio transport，
  细节见 environment reference。

## Case CLI

脚本 CLI 使用 `rundroid_main`：

```python
from rundroid import rundroid_main

@rundroid_main
def main() -> int:
    # Emulator 会读取当前 CLI scope 的 GuestEvent selection。
    return 0

if __name__ == "__main__":
    main()
```

运行时可以重复传入 `--events/-e`，例如 `-e JNICALL -e SYSCALL`；也支持 `all` 或
`none`，二者不能和普通 token 混用。未传 option 时 selection 是 `None`，不能覆盖脚本
默认事件集合。

`rundroid_main` 用 ContextVar 将 selection 限定在当前 decorated main 作用域，并在正常
返回、异常和显式退出路径 reset。不要让 module global 保存上一轮 CLI 参数。

## 使用限制

- 非法 token、重复 token、未知事件和错误 main 返回类型都应显示为参数/类型错误并
  fail-fast。
