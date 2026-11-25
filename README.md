# ascend-link-triage

一套面向昇腾多机环境的 **UB / UBoE 通信排障 skill**，帮助 AI Agent 从硬件、设备端口、交换机、拓扑配置到日志和真实业务传输逐层定位问题。

经验主要来自 A5 / 650 / 650E 环境的实际排查。核心流程是：**确认环境与映射 → 收集证据 → 最小卡对复现 → 正常/异常对照 → 修复与回归验证**。

这是一套指令和参考文档，不是安装后自动运行的诊断程序；仓库不包含设备驱动、打流工具、账号密码或自动修改网络的脚本。

## 能排查什么

- **硬件与设备端口**：卡不可见、健康告警、端口 down、bond 成员异常、LLDP 邻居与实际接线不符；核对物理卡、容器可见卡、rank、PID、EID/IP 和交换机端口映射。
- **UBoE 与交换机**：设备 ping 丢包、建链超时、ARP 映射漂移、MAC/Trunk 不匹配、LACP PortNo 重复，以及 MTU、VLAN、QoS/PFC 等配置对照。
- **UB 与 HCCL**：fullmesh 部分卡对不通、控制网选错、rootinfo/topo 与实际组网不一致、集合通信启动异常。
- **URMA / HIXL**：ping 通但打流失败、读写方向差异、异常低带宽、注册内存或 HBM 分配失败、对端退出导致的连接错误。
- **PD / KV 传输**：部分 rank 失败、KV 建链超时、重试、TTFT 长尾，及“HTTP 成功但 KV 实际失败”的情况。
- **日志取证**：业务日志、plog、msnpureport 设备诊断包、Host 与 Device 的 `/var/log/messages`、内核日志的跨层关联。

适合通信故障定位与修复验证；不以单纯的模型精度分析、吞吐调参或固件升级为主要用途。

## 安装前准备

1. 使用支持 `SKILL.md` 的 Agent。以下以 Codex 为例。
2. 安装 Git，并确保能访问 GitHub。
3. 以下示例使用 HTTPS 克隆；若仓库为私有仓库，需使用拥有读取权限的 GitHub 账号认证。
4. 实际排障时，Agent 所在环境需要能够访问目标服务器；需要查交换机时还需相应访问权限。安装 skill 不会自动获得这些权限。

可先验证仓库访问：

```bash
git ls-remote https://github.com/haley-osment/ascend-link-triage.git
```

若访问失败，检查 GitHub 网络连接、仓库地址及账号权限。

## 安装

### 方式一：克隆到用户技能目录（推荐）

当前官方文档列出的 Codex 用户级技能目录是 `~/.agents/skills/`，项目级目录是 `.agents/skills/`。参见 [OpenAI 官方安装说明中的技能目录](https://developers.openai.com/api/reference/cli)。

**Windows PowerShell：**

```powershell
$skillRoot = Join-Path $env:USERPROFILE '.agents/skills'
New-Item -ItemType Directory -Force -Path $skillRoot | Out-Null
git clone https://github.com/haley-osment/ascend-link-triage.git (Join-Path $skillRoot 'ascend-link-triage')
```

**Linux / macOS：**

```bash
mkdir -p "$HOME/.agents/skills"
git clone https://github.com/haley-osment/ascend-link-triage.git "$HOME/.agents/skills/ascend-link-triage"
```

如果该目录已经存在，不要覆盖，先确认是不是本仓库的已有安装，再按下文更新。

部分已有环境使用 `~/.codex/skills/` 或自定义技能根目录；以当前客户端实际加载的路径为准。本项目最初在 `~/.codex/skills/ascend-link-triage/` 创建。已有安装可继续维护，不要在多个被扫描的目录保留不同版本的同名 skill。

### 方式二：下载并手动安装

从仓库下载压缩包，将完整内容解压到技能根目录下的 `ascend-link-triage` 文件夹。确认目录结构如下，避免多嵌套一层，也不要只复制 `SKILL.md`：

```text
<技能根目录>/ascend-link-triage/
├── SKILL.md
├── README.md
├── agents/openai.yaml
├── references/
└── assets/evidence-template.md
```

安装后开启新会话并显式调用下面的 skill。若未被识别，检查技能目录、文件嵌套层级及客户端是否需要重新加载。其他 Agent 的安装路径以其自身文档为准。

## 如何使用

测试双机 HIXL D2rH 分段带宽时，skill 自带适用于 Linux ARM64、CANN 9.2.1 参考环境的预编译工具，可离线解压。见[客户现场前提及操作说明](references/hixl-d2rh.md)。

在 Codex 对话中使用 `$ascend-link-triage` 显式调用；`$` 前缀调用方式参见 [OpenAI Skills 使用示例](https://developers.openai.com/blog/eval-skills)。

### UBoE 建链超时

```text
使用 $ascend-link-triage 排查两台服务器之间的 UBoE 建链超时。
P 主机：<主机地址或 SSH 别名>，容器：<P 容器>
D 主机：<主机地址或 SSH 别名>，容器：<D 容器>
现象：<错误信息及发生时间>
先只读检查设备 IP、bond、LLDP、交换机 ARP/LACP 和两端日志。
允许在指定空闲卡上做小流量验证；不要重启服务或修改交换机配置。
```

### UB / HCCL / topo 问题

```text
使用 $ascend-link-triage 排查 UB fullmesh 部分卡对不通的问题。
启动脚本：<路径>，rootinfo/topo：<路径>，失败日志：<路径>
先核对实际组网、卡与 EID 映射、Host 控制网和容器实际读取的文件，
再给出最小复现方案，不直接覆盖现有 rootinfo。
```

### ping 通但 HIXL 或 KV 传输失败

```text
使用 $ascend-link-triage 排查 ping 正常但 KV 传输超时的问题。
服务日志：<路径>，发生时间：<时间和时区>，请求 ID：<ID>
请关联实际 rank/PID、plog、Host/Device 日志，区分建链失败、
传输完成慢和对端退出；必要时用同一卡对做 URMA/HIXL 对照。
```

首次排查尽量提供：目标机器与连接方式、容器/角色、症状和时间、日志/脚本位置、可用空闲卡，以及允许的测试和变更范围。缺失信息会先从现有环境发现；密码通过安全渠道提供，不写入仓库。

也可以直接阅读 [SKILL.md](SKILL.md) 和参考页，按文档人工排查。

## 预期输出与判断边界

一次排查应留下：受影响机器/卡对、证据位置、已证实事实、根因或待验证假设、变更与回滚、配置是否保存，以及复测的方向/规模/成功失败数。可使用 [取证模板](assets/evidence-template.md)。

- Host 网络可达、设备 ping 成功、HTTP 200 都不能单独证明实际 KV 传输正常。
- 8×8 卡对验证不等于大包满载或长期稳定性验证；同卡号测试也不等于 fullmesh。
- 静态 ARP 下恢复正常，不代表动态 ARP 学习问题已经解决。
- 历史端口、EID、topo、LACP 偏移和工具参数需按当前版本验证，不能直接照抄。
- skill 会区分只读取证与可能影响业务的操作；重启、reset、网络配置等变更必须在用户授权范围内。

## 文档导航

- [硬件、端口与 Device 入口](references/hardware-device.md)
- [交换机、UBoE、ARP 与 LACP](references/switch-uboe.md)
- [UB、HCCL 与 rootinfo/topo](references/ub-hccl-topology.md)
- [URMA、HIXL 与真实 KV 分层验证](references/traffic-tests.md)
- [业务、plog、MSN 与 Host/Device 日志](references/logs.md)
- [历史案例与证据等级](references/cases.md)

## 更新已安装版本

使用 Git 安装的用户，在实际安装目录执行：

```bash
git status --short
git pull --ff-only
```

有本地修改时先保留并处理差异；快进更新失败时不要使用强制覆盖。手动解压安装的用户重新下载最新包，核对差异后替换。更新后在新会话中调用。

若开发仓库与安装目录分开，推送开发仓库不会自动更新已安装副本；需要在安装目录拉取或同步文件。

## 如何沉淀新经验

在对应参考页补充方法，在案例页记录具体事实。每次说明环境版本、症状、最小复现、证据、排除项、修复与回滚、复测覆盖，并明确哪些仍是假设。

提交前检查 Markdown 链接和命令执行位置，不提交凭据、私钥、令牌或未经脱敏的原始日志。未经当前环境验证的历史命令应标注适用条件，不能把单次成功写成通用结论。

