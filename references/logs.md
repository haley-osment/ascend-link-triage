# 日志收集与跨层关联

## 五类位置

1. **业务/代理日志**：从启动脚本重定向、容器日志、实际进程 stdout 位置发现。记录请求ID、P/D角色、DP rank、PID、远端session、字节数、开始结束、重试和输出长度。
2. **进程 plog**：root 历史默认目录 `/root/ascend/log/debug/plog`；实际用户、日志级别及 `ASCEND_PROCESS_LOG_PATH` 可改变路径。检查目标进程环境中的该项，不要打印完整 environ。文件按 PID 和启动时间关联，不能只取“最新文件”。
3. **Host 系统日志**：宿主机 `/var/log/messages`、`journalctl -k` 或 `dmesg`，查驱动、复位、设备异常、链路事件。
4. **Device OS 日志**：进卡后的 `dmesg` 与 `/var/log/messages`，与 Host 的同名文件不是同一份。回传日志中可能出现 `device-<PID>`，需结合物理卡和命名空间识别。
5. **MSN/msnpureport 设备诊断包**：Host 的工具导出，包含哪些文件以实际版本输出为准。“msn日志”先确认用户指的是导出包还是某个具体文件。

Host 复现后及时导出到空间充足的唯一目录：

```bash
capture_dir="$EVIDENCE_ROOT/device-report-$(hostname)-$(date +%Y%m%d_%H%M%S)"
mkdir -p "$capture_dir"
cd "$capture_dir"
/usr/local/Ascend/driver/tools/msnpureport -f
```

`-t` 各类型、容器导出与按卡参数需查看该版帮助，不能把历史 `-t 1` 或用户提到的 `-t 5` 当成通用全量导出。保留工具输出、文件清单、大小与时间，零字节或空目录不算成功。

## 关联流程

同时记录两端 `date -Ins` 与时区；主机时钟可能有分钟级偏差，日志也可能UTC。优先请求ID→rank/PID→session/channel→stream/任务，最后用修正后的时间窗口关联交换机和Device。不同进程的 slot、stream、Notify 编号可重复。

```bash
# 在目标进程所在命名空间查询
tr '\0' '\n' < "/proc/$WORKER_PID/environ" | rg '^ASCEND_PROCESS_LOG_PATH='
rg -n 'KV cache transfer|KV load failure|Retry transfer' "$SERVICE_LOG"
rg -n -i 'AutoConnect|CreateChannel|WaitChannelConnected|Channel Ready|HixlBatchGet|NOTIFY_WAIT|stream synchronize timeout' "$WORKER_PLOG"
```

没有 rg 用 grep；同时读取错误前后上下文和对端同次事件，不只汇总关键字计数。一个请求可有多桶KV、多次尝试，桶数、尝试数、请求数分开统计。

## 如何下判断

- `WaitChannelConnected` / `is_connected:0`：建链未完成。查实际设备地址、端口、转发、聚合、TLS和对端状态，不能直接判硬件坏。
- Channel Ready 后提交 HixlBatchGet，但完成慢：取同一窗口 TX/RX、丢包/PFC 增量与设备SQ/完成进度；有其他业务时端口计数只作辅助。
- `NOTIFY_WAIT`：当前流在等事件，不证明数据已经搬完，也不证明问题一定在网络。
- `halMemAlloc`、大页申请失败：先核对目标卡空闲HBM及申请量；对端随后的 closed/reset 可能只是结果。
- plog 显示成功但业务失败：再查请求路由、缓存元数据生命周期、输出和错误处理，不反向推导“网络必坏”。

需提高日志级别时，可在受控复现进程启动前短时设 `ASCEND_GLOBAL_LOG_LEVEL=0`（历史定义DEBUG=0）；先确认本版规则、日志空间和恢复方法。修改当前 shell 不会自动改变已运行worker的环境。不要为开debug擅自重启服务。
