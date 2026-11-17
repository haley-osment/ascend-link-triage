# 交换机与 UBoE

## 从端点反查转发路径

建立每个 bind IP 的 IP→本卡 MAC→VLAN→Eth-Trunk→物理成员→LLDP 邻居映射。两台交换机分别核对；不能只根据 IP 尾号推端口。区分 Eth-Trunk 接口与 E-Trunk 协议配置，用户口语可能混用。

Huawei 现场只读命令形态（使用实际接口/IP替换示例占位）：

```text
display current-configuration interface Eth-Trunk<id>
display eth-trunk <id>
display lldp neighbor interface <physical-interface>
display arp | include <device-ip>
display arp history | include <device-ip>
display mac-address | include <mac>
display interface <physical-interface>
```

命令随版本变化，先用 CLI `?` 校验。抓取 ARP history 时保留事件时间、原因、旧/新 MAC、旧/新 Trunk。一次 ARP 快照正确不能否定历史错学。若记录因 Packet 更新而切换到另一个在线物理设备的 MAC，优先追查重复 IP/错误源端配置和实际报文；不要只凭漂移认定交换机软件故障。ping 可能触发重新学习，应记录测试前后状态。

控制面 TCP 可达不能证明 UBoE 可达。设备 ping 用已核对的本卡 IP：

```bash
# Host
hccn_tool -t -ping -i "$CARD" -src_address "$LOCAL_DEVICE_IP" -dst_address "$PEER_DEVICE_IP" -pkt 64
```

正常卡与异常卡同时对比 bond、MTU、VLAN、网关、DSCP/TC/PFC、buffer、水线、TLS。正常与异常相同的配置不能单独作为根因；QoS 数值也不能按历史案例盲改。

## LACP 端口编号重复

跨交换机聚合若共享 LACP System ID，需核对对端看到的成员 PortNo 是否唯一。现场两台 leaf 对同一设备的 PortNo 都为75；在一台设置偏移后变为2075，另一台保留75。

某 XH9000 V300R026 版本 CLI 支持 `lacp port-id-extension` 的 `enable` 和数值参数；帮助显示数值为0、2000、4000…20000。其他版本可能只支持 enable。不要将2000作为全网默认值，也不要两台同时加相同偏移而仍保留重复编号。已有 enable/偏移配置要先检查实际 PortNo，不能无条件覆盖。

经授权的单接口修改形态：

```text
system-view
interface Eth-Trunk<id>
lacp port-id-extension <verified-offset>
quit
commit
return
display eth-trunk <id>
```

修改前保存原值，回滚恢复原值；原先无此项才考虑 `undo lacp port-id-extension`，并先检查语法。变更可能触发 LACP 重新协商，先确认目标业务状态。验证所有目标成员 Selected、实际编号已区分，再做业务传输验证。

## 静态 ARP 与保存

静态 ARP 可在已确认 IP/MAC/VLAN/接口后作为定向绕过，不能证明动态学习根因。现场支持的命令形态为：

```text
arp static <ip> <mac> vlan <vlan> interface Eth-Trunk<id>
```

准备对应 undo 或原配置恢复；不要批量清空 ARP。恢复动态学习是另一项可能影响业务的实验，按已授权范围执行，监控正确/错误 MAC 的来源和变化。

`commit` 提交运行配置；`save` 写入下次启动配置，**不等于 reboot**。save 通常保存整份当前配置，不只本次命令，需留意并发管理员的变更。正式变更验证后执行保存并处理交互确认，通过 `display startup` 与 `display saved-configuration` 核对目标接口实际存在于启动配置。不要只看到“Now saving”就宣称已保存成功。
