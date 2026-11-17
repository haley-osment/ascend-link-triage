# UB 与 HCCL 拓扑

原生 UB 的 EID、ipourma 接口和 UBoE 的设备 IPv4/bond 不是同一套选路依据。先用 `hccn_tool -g -dev_info` 和 `urma_admin show` 对照完整 EID、设备、平面与物理口。历史双机 fullmesh 使用过 udmac0d0e3 的跨机 EID，另一现场使用不同前缀；不复制别机 EID 或固定字符串匹配规则。

Host UB ping 模板（先核对本机支持参数）：

```bash
hccn_tool -t -ub_ping -i "$CARD" -src_eid "$LOCAL_EID" -dst_eid "$PEER_EID" -pkt 128 -cnt 6 -interval 1
```

按实卡数生成源卡×目标卡矩阵并标方向。同卡号测试不等于8×8，正向也不等于反向。规律性成片失败优先核对平面/接线/拓扑，零散失败优先对比具体端口与设备状态；这只是优先级，不是根因结论。

## rootinfo/topo 核对

从实际进程启动参数、容器挂载和日志找到正在读取的 rootinfo/topo；对 Host 文件和容器内文件分别计算 SHA256，防止“改了宿主机文件但容器仍读旧文件”。检查 JSON 语法及实际 schema，不假定不同版本字段完全一致。

核对机型、机器数、卡数、die/平面、rank 映射、server/device 地址、超节点配置和实际布线。`netLayer[1] not exist in topo` 提示拓扑层不匹配，应结合当前配置和生成来源定位。

历史某双机 fm16 环境使用 `/usr/local/Ascend/driver/topo/950/atlas_850_2.json`，iBMC 超节点需与实际16p组网一致；不能将16p或该文件推广到所有650/650E环境。BMC配置、固件与重启都不属于只读检查。

现有环境若已安装并验证 mindcluster-tools，可先看帮助，再生成到新的证据目录：

```bash
mindcluster-tools rootinfo --help
mindcluster-tools rootinfo -t "$VERIFIED_TOPO" --output "$NEW_ROOTINFO"
```

这是历史工具接口，不自动安装 unofficial 包，不直接覆盖 `/etc/hccl_rootinfo.json`。记录工具来源/版本、输入 topo hash 与生成结果；核对后按授权替换并保留回滚。

## HCCL 控制网与集合通信

`FindLocalHostIp` / `there is no valid host ip`：先看 Host IPv4、选网卡与实际进程的 `HCCL_SOCKET_IFNAME`；不要用 ipourma 设备地址替代 Host 控制地址。`HrtRaSocketGetSockets` timeout：核对实际IP/端口监听、路由和对端进程首错。Gloo、TP 控制网变量与 HCCL 的用途分别确认。

在已验证的本地 HCCL test 工具上检查版本、`--help`、MPI 实际路径、依赖库、hostfile、rank 分配、环境变量传递及结果校验，再小规模复现原算子。集合通信测试需空闲资源，不在未知在线服务上直接扩满卡。

`AIV task exception`、`AIX LINK ERROR` 可能是前面建链/拓扑错误后的结果，也可能有其他原因；查同 PID/同窗口的第一条错误，不能将字符串直接映射为 rootinfo 根因。
