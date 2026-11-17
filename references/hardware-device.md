# 硬件、端口与 Device 入口

## Host 与容器基线

Host 先运行 `hostname`、`date -Ins`、`npu-smi info`、`ip -br addr`、`ip route`、`ss -lntp`。记录卡型号、健康/告警、占卡 PID、内存及实际监听地址。容器用 `docker ps` 和定向 `docker inspect` 查询镜像、设备映射、挂载、Entrypoint/Cmd；不要公开完整环境变量或认证配置。结合容器内进程与 `/proc/<pid>/status` 的 NSpid 核对 PID 命名空间。

Host 历史工具路径为 `/usr/local/Ascend/driver/tools/hccn_tool`。下面是已记录的查询形态，变量须先从本机信息填写；`-g -dev_info` 输出可能很大，应保存后筛选。

```bash
hccn_tool -g -dev_info -i "$CARD"
hccn_tool -g -ip -i "$CARD" -d "$BOND"
hccn_tool -g -gateway -i "$CARD" -d "$BOND"
hccn_tool -g -bond_attr -i "$CARD" -d "$BOND"
hccn_tool -g -lldp -i "$CARD" -d "$BOND"
hccn_tool -g -link_stat -i "$CARD" -u "$UDIE" -p "$PORT"
```

部分工具要求 `-i` 放在主参数前，以本机帮助为准。记录设备名、完整 EID、IP/掩码/网关、UDie、ETH 口及 bond 成员。650E 某现场是 UDie1 的端口8/9、udmac0d1e3、bond<卡号>；这是例子，不是默认映射。

检查当前 link、历史 down/复位、协商速率、活动成员数、聚合组 ID、partner System ID/Port ID。双成员存在不等于双成员都 Selected。LLDP 的交换机/物理口必须能和交换机反向邻居对应。链路 up 也不能排除 CRC、丢包、PFC 或转发错误；取同一测试窗口前后计数增量。

## 进入 Device

Host 的 root shell、容器 root shell、Device OS shell 是三种不同位置。先从本机 `dev_info`、网卡/EID 信息和现有运维配置找到本机 Host 侧 UB 地址与目标卡可管理地址；不要用 UBoE 数据 IP 或猜测地址替代。

历史使用过绑定本地 IPv6 地址的 SSH 入口，通用形态为：

```bash
# Host 执行；两地址必须属于本次主机和目标卡的已确认映射
ssh -b "$HOST_UB_SOURCE_IPV6" "${DEVICE_USER}@${DEVICE_MGMT_IPV6}"
```

历史账号名是 HwHiAiUser，但新环境以实际配置为准。密码和提权方式经授权的安全渠道取得；不附带默认密码，不通过本 skill 修改 SSH/安全策略。登录后先用 `hostname`、`id`、`ip -br addr`、`urma_admin show` 确认卡身份；工具可用性以实际镜像为准。需要权限时按现场授权使用 `su`，不要假设已是 root。

Device 可查询 `urma_admin show`、`dmesg`、`/var/log/messages`。`ubctl` 等诊断接口先看本机帮助，只执行确认含义的只读子命令。没有可靠的登录映射时优先使用 Host 的 hccn_tool 包装命令及 msnpureport 导出，并明确尚未直接进卡。

退出时如曾提权，分别退出提权 shell 和 SSH；回 Host 后再次检查 `hostname`。不要在未知提示符下粘贴 Host/交换机命令。
