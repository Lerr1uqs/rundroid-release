# Interception

## 适用场景

需要让 Java method 或 syscall 被 Python callback 观察、修改或短路时查阅本篇。

## Java method interceptor

Java method interceptor 的入口是 `Interceptor.java_method(...)`：

```python
from rundroid import Interceptor, JavaMethodContext
from rundroid.javashim import JavaObject

@Interceptor.java_method("test/Signer->sign([B)[B", mode="override")
def override(
    ctx: JavaMethodContext,
    self_obj: JavaObject,
    payload: bytes,
) -> bytes:
    return b"mock"

handles = emu.interceptor.register([override])
emu.interceptor.remove(handles[0])
```

context 提供 class/method/descriptor、static/instance、call kind、mode 和 callback ART
facade。

支持的语义包括：

- hook：观察调用，可继续原始实现。
- override：阻止原始实现并返回 host 值。
- supplement：原始实现缺失时提供补充实现。

### 和 class extension 的分工

interceptor 是 **runtime overlay**，适合临时/实验性的单 method 改动；它不能声明 field
或 interface 关系，也不成为 Python blueprint 成员。要领长期补一组成员（method +
field + 关系）请继承 `JavaClassExtension` / `JavaInterfaceExtension` 并调用
`emu.art.extend([...])`（见 `java-shim-authoring.md` 的
「什么时候用 declaration / extension / interceptor」）。

两者不隐式叠层：目标 member 已有 active `hook` / `override` / `supplement` 时，
extension 的 `override=True` 会在注册期被拒绝，必须先 `emu.interceptor.remove(...)`。
extension 新增的 method 与 `supplement` placeholder 占用同一个 canonical key，
后注册者 fail-fast 并报告双方来源。

注册时会校验 signature、callback shape、返回策略和 duplicate binding。使用
`remove(handle_or_func)` 或 `clear(signature=None)` 清理拦截器。

## Syscall interceptor

syscall interceptor 的入口是 `Interceptor.syscall(...)` 和 `Interceptor.raw_syscall(...)`：

- `Interceptor.syscall(nr)`：按已知 syscall arity 注册 decorator。
- `Interceptor.raw_syscall(nr, argc=...)`：对 raw syscall 显式声明参数数量。
- `Interceptor.remove(...)`、`Interceptor.clear(syscall_nr=None)`：按 handle/function
  或 syscall 维度清理。

Syscall callback 的参数数量必须和 syscall ABI 对齐；不要通过 `*args` 隐藏错误 arity。
host 返回值需要遵守现有 syscall result/error 投影，不能把异常静默转换成成功返回。

## Callback 限制

- callback 只持有当前调用所需的短生命周期 context。
- Java interceptor callback 可以使用 `ctx.art`，但不能嵌套 `call`、`call_address` 或
  `invoke_native`。
- syscall callback 使用的 `CpuContext` 只在当前 callback 内有效，机器调试统一通过
  `cpu.cdt`，不要保存到全局。
- callback 必须返回当前 API 要求的真实类型；不要用 `None` 或任意整数代替控制返回值。
