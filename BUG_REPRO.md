# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

海关申报的超时形同虚设：给了 120ms 的超时，实际等满 5 秒才返回，而且还报成功、把申报单号发出去了。

```
$ ./oilctl customs submit --timeout 120ms --customs-latency 5s
{
  "alive": true,
  "channel": "海关单一窗口",
  "elapsed_ms": 5000,
  "ok": true,
  "serial_no": "CN-000001",
  "shipment_id": "S-001",
  "timeout_ms": 120
}
$ echo $?
0
```

`timeout_ms` 是 120，`elapsed_ms` 却是 5000。按 README 的约定，调用方设定的超时到达或调用被取消时应该立即返回对应错误、退出码 4。现在等于超时之后申报照做，还拿到了单号 —— 上游以为我们撤单了，实际单子已经报上去。

`POST /api/customs/submit?timeout_ms=120` 表现一样，等满通道延时才返回 200。

对照现象：

- `./oilctl customs submit --shipment S-001 --mass 9525.4 --timeout 3s`（通道延时远小于超时）正常成功，`elapsed_ms` 是 0，这个没问题。
- `./oilctl customs probe --timeout 100ms --customs-latency 5s` 也是等满 5 秒才回来，`ok` 仍是 true。

所以只要通道延时超过调用方给的超时，超时就完全不生效，探测和申报都一样。

帮我修好，让调用方设定的超时与取消真正生效。已有测试跑一遍不要有回归。

## 含 Bug 版本

- 仓库：VanceMichael/go-annotation-21
- 仓库地址：https://github.com/VanceMichael/go-annotation-21.git
- parent SHA：7bb1b5f844e0bbc42f2a4b9c0cf3639ec84ef1e9

## 复现步骤

```bash
git clone -- https://github.com/VanceMichael/go-annotation-21.git bug-repro
cd bug-repro
git checkout --detach 7bb1b5f844e0bbc42f2a4b9c0cf3639ec84ef1e9
go test ./internal/gateway/ -run "TestSubmitHonoursCallerTimeout|TestSubmitHonoursCallerCancel|TestQueryHonoursCallerTimeout|TestProbeHonoursCallerTimeout|TestSubmitAliveClientNotAffectedByOtherCallTimeout" -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/gateway/ -run "TestSubmitHonoursCallerTimeout|TestSubmitHonoursCallerCancel|TestQueryHonoursCallerTimeout|TestProbeHonoursCallerTimeout|TestSubmitAliveClientNotAffectedByOtherCallTimeout" -count=1
--- FAIL: TestSubmitHonoursCallerTimeout (5.02s)
    gateway_test.go:41: 超时后 Submit 应返回错误
--- FAIL: TestSubmitHonoursCallerCancel (5.01s)
    gateway_test.go:66: errors.Is(err, context.Canceled) = false, 错误为 <nil>
--- FAIL: TestSubmitAliveClientNotAffectedByOtherCallTimeout (0.31s)
    gateway_test.go:81: 短超时申报应失败
--- FAIL: TestQueryHonoursCallerTimeout (5.01s)
    gateway_test.go:105: errors.Is(err, context.DeadlineExceeded) = false, 错误为 <nil>
--- FAIL: TestProbeHonoursCallerTimeout (5.01s)
    gateway_test.go:123: 超时应返回错误
FAIL
FAIL	wasteoil/internal/gateway	20.393s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/gateway/ -run "TestSubmitHonoursCallerTimeout|TestSubmitHonoursCallerCancel|TestQueryHonoursCallerTimeout|TestProbeHonoursCallerTimeout|TestSubmitAliveClientNotAffectedByOtherCallTimeout" -count=1
--- FAIL: TestSubmitHonoursCallerTimeout (5.01s)
    gateway_test.go:41: 超时后 Submit 应返回错误
--- FAIL: TestSubmitHonoursCallerCancel (5.01s)
    gateway_test.go:66: errors.Is(err, context.Canceled) = false, 错误为 <nil>
--- FAIL: TestSubmitAliveClientNotAffectedByOtherCallTimeout (0.30s)
    gateway_test.go:81: 短超时申报应失败
--- FAIL: TestQueryHonoursCallerTimeout (5.01s)
    gateway_test.go:105: errors.Is(err, context.DeadlineExceeded) = false, 错误为 <nil>
--- FAIL: TestProbeHonoursCallerTimeout (5.01s)
    gateway_test.go:123: 超时应返回错误
FAIL
FAIL	wasteoil/internal/gateway	20.345s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

定向测试与全量回归在 linux/amd64、linux/arm64 双架构下均通过。
调用方设定的超时到达时，申报与探测都在超时时刻附近返回非 nil 错误，可通过 errors.Is 判定为 context.DeadlineExceeded；调用被取消时可判定为 context.Canceled。
CLI 退出码为 4，且 elapsed_ms 接近所设超时而不是通道延时；HTTP 侧返回 503。
超时未到达时的正常申报结果不变：仍然成功并返回申报单号。
一次调用超时不影响同一客户端后续在充足超时下的调用。
