# UB 与 HCCL 拓扑

原生 UB 的 EID、ipourma 接口和 UBoE 的设备 IPv4/bond 不是同一套选路依据。先用 `hccn_tool -g -dev_info` 和 `urma_admin show` 对照完整 EID、设备、平面与物理口。历史双机 fullmesh 使用过 udmac0d0e3 的跨机 EID，另一现场使用不同前缀；不复制别机 EID 或固定字符串匹配规则。

## 原生 UB CTP 的两种路径

`ub_ctp` 是协议名，框内 fullmesh 直连和经出框口的 UB CLOS 都可以使用。排查建链日志时先记录实际选中的本地、远端 EID 和物理卡号，再对照进程使用的 endpoint 配置（常见于 `/etc/hixlep/ub_endpoint_npu_<物理卡号>.json`）与现场拓扑。

**Fullmesh 直连按卡对选。** 每张卡连向不同对端卡的直连端口各有自己的 EID。从卡 A 到卡 B，在 A 文件中找带 `dst_eid` 的 `ub_ctp` 条目，其 `comm_id` 是源 EID，`dst_eid` 是目的 EID。在 B 文件中核实两者恰好对调。即使源卡相同，换对端也要重新查这一对。可在 Host 上用下面的命令从两份配置中找互指记录，输出依次为源、目的 EID。

```bash
SRC=6; DST=2
jq -rs '.[0].endpoint_list[] as $a | .[1].endpoint_list[] | select($a.protocol=="ub_ctp" and .protocol==$a.protocol and $a.dst_eid==.comm_id and .dst_eid==$a.comm_id) | [$a.comm_id,.comm_id] | @tsv' "/etc/hixlep/ub_endpoint_npu_${SRC}.json" "/etc/hixlep/ub_endpoint_npu_${DST}.json"
```

**经出框口的 UB CLOS 按平面选。** 在生效的 rootinfo/topo 中确认出框端口接入原生 UB fabric，并对应到两端的同一平面。在源端和对端分别从本卡文件取该平面的 `comm_id`，作为两端 EID。平面条目可以只有本端地址。下面这条在各端 Host 分别执行，`CARD` 取本端物理卡号，`PLANE` 取现场确认的平面名。

```bash
jq -r --arg p "$PLANE" '.endpoint_list[] | select(.protocol=="ub_ctp" and .plane==$p) | .comm_id' "/etc/hixlep/ub_endpoint_npu_${CARD}.json"
```

若查不到或查出多条候选，按 topo 的端口映射和业务日志里的完整 EID 核对。设备枚举的 PG（Port Group）EID 标识端口组，仅凭 EID index 或名字 `plane_pg_0` 无法确定走哪条物理链路。若 rootinfo 的 CLOS 条目标记为 `plane_uboe` 且地址为 IPv4，应按 UBoE 核查。复现业务日志中的 PG EID 超时，可报告这对 EID 的可达性；确认原生 UB CLOS 故障还需要该 PG 到出框端口的映射。

**现场例子。** 某台八卡机的物理卡 6 到卡 2 使用源 EID 尾段 `d306`，到卡 3 使用 `d206`。同卡枚举的 `db06` 是 mesh 侧端口组 EID，实际 endpoint 文件中 `plane_pg_0` 的本端 EID 尾段是 `cb06`。这些尾段只用于说明这个例子，命令始终使用现场完整 EID。

**执行与判读。** 在两端用 `hccn_tool -g -dev_info -i "$CARD"` 核对完整 EID 归属。从源 Host 发起时，`CARD` 是源物理卡号，两端 EID 均按八组四位十六进制填写。反向复测时在原目的端发起，交换两端 EID 并更新源卡号。按实际发送、接收和丢包数判读，即使全丢包也可能显示 `Cmd executed successfully!`。

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
