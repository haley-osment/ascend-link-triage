# 从 URMA 到 HIXL 再到真实 KV

各层分别验证不同能力：设备 ping→URMA数据搬运→HIXL建链/注册内存/校验→真实业务KV。按失败层选测试，不要求每次都跑满全套。UDMA设备/EID、Host控制IP、设备数据IP不能混用。

## URMA：先一对卡

先查实际版本帮助。下面是历史 A5 UBoE Host 包装器的10秒 read_bw模板，变量均需绑定到目标卡。启动server并确认就绪后再启动client，两端参数保持一致。

```bash
# server Host；CARD/DEVICE/SIP/DIP 为本端卡、设备、本端与对端设备IP
hccn_tool -t -urma_perftest -i "$CARD" -d "$DEVICE" \
  --eid_idx 0 -s 16384 -D 10 -type read_bw --tp_aware \
  -C 12800 -J 1 -Q 256 -T 4096 -l 256 --hugepage_size 2MB -w \
  --sip "$SIP" --dip "$DIP"

# client Host；变量改为client视角，SERVER_IP仍指server设备IP
hccn_tool -t -urma_perftest -i "$CARD" -d "$DEVICE" \
  --eid_idx 0 -s 16384 -D 10 -type read_bw --tp_aware \
  -C 12800 -J 1 -Q 256 -T 4096 -l 256 --hugepage_size 2MB -w \
  --sip "$SIP" --dip "$DIP" -S "$SERVER_IP"
```

历史包装器要求 `-i/-d` 紧跟 `-urma_perftest`。Device 内直接调用则形如 `urma_perftest read_bw -d ... --uboe ...`，不是Host的 `-type read_bw`；以 Device 版本帮助确认持续时间参数。原生UB的 `--bind_ip`/EID流程须从本版帮助与已核对拓扑构造，不把上述IPv4 UBoE命令换个设备名就运行。

read/write、单向/双向（历史 `-B`）、16KiB/1MiB、持续打印（历史 `-y`）与定时（`-D`）分别记录。`iterations=0` 无效；超时、poll jfc失败、零带宽均需看双方首错。控制连接失败先查server是否仍活着和实际命名空间监听；不要先调QoS。

若工具残留，先定位本次进程与资源。`-type reset` 不是无害查询，不能放在循环开头无条件执行。只在影响范围和授权明确后使用本版支持的定向清理。

## HIXL：版本固定与数据校验

双机 D2rH 分段带宽默认执行[离线 C++ benchmark](hixl-d2rh.md)。下面的 Python 示例用于对应版本的其他对照实验。

复用现场已验证容器，确认Python/CANN/HIXL、设备映射及空闲HBM。历史 benchmark 固定源提交 `4f523ccd51a926ac9ec7af3ea7e116616b10ff0d`，来自 cann/hixl 的 `benchmarks/hixl_benchmark`；配套文件为 hixl_benchmark.py、bench_lib.py、acl_compat.py。不要只复制主脚本。读取随包 README/源码及 `--help`，记录来源、commit与文件hash。该历史版用过Python3.12和CANN9.2；其他版本先验证兼容性。

下例为该历史接口的小规模 rD2D（读远端设备到本地设备）模板。容器两端先设置已确认存在的 CANN 环境，核对现有协议配置；实验用UBoE时曾显式设置：

```bash
export ASCEND_LOCAL_COMM_RES='{"version":"1.3"}'
export ASCEND_GLOBAL_RESOURCE_CONFIG='{"comm_resource_config.protocol_desc":["uboe:device"]}'

# target容器
python3 -u hixl_benchmark.py --device "$CARD" --direction rD2D --transport uboe \
  --block_sizes 1M --transfer_size 64M --loops 5 --listen "$ENGINE_PORT"

# initiator容器，target ready后运行；REMOTE_HOST是Host控制地址
python3 -u hixl_benchmark.py --device "$CARD" --direction rD2D --transport uboe \
  --block_sizes 1M --transfer_size 64M --loops 5 --remote "$REMOTE_HOST:$ENGINE_PORT"
```

环境变量可能优先于 `--transport`，实际协议还须从日志验证。该版协调端口为引擎端口+10000，换版先看源码；选未占用且在有效范围内的两个端口。target因HBM申请失败退出时，client可能仅显示coordination peer closed，先看target首错。

`rD2D`用于D主动拉P的方向；`D2rD`是本地设备写远端设备，`D2rH`是写远端Host内存。不要把写成功当读成功。先确认 `Data verify : pass`，再报告GB/s、实际字节数、块大小、迭代数、预热、方向。历史某对卡46.6GB/s只是单场景观测，不是硬件验收阈值。

## 真实业务 KV

固定一端换另一端，异常卡与正常卡做 A→B→A。只有后端或代理源码明确支持时才使用目标rank请求头；历史后端支持 `X-data-parallel-rank`，普通代理可能丢掉该头。必须从worker日志确认实际落点。

PD调试协议随connector变化：先核对当前实现的prefill请求、kv_transfer_params及decode请求格式，再复用现场脚本；不要伪造通用接口。每次唯一请求ID和小输入，检查实际输出、KV成功日志、错误/重试，并保存卡对矩阵。历史某版connector在异常后的finally仍标记任务done，故“所有HTTP成功”不能单独证明KV成功。

涉及慢读/同卡计算干扰时，在空闲实验窗口比较无计算与相同传输期间插入一次计算，确认时间重叠、字节数、协议与数据校验；连续Mul和单次Mul是不同负载，不能混为一组。每轮使用新目录，避免上次done/stop标记导致提前退出。

## 性能异常：覆盖方向与分段形态

先做短时、小规模对照，不直接展开所有参数的笛卡尔积。明确成功、失败、未测、初始化失败四种状态，初始化失败不算性能结果。

- 按内存落点覆盖相关方向：D2rH（本地 HBM→远端 Host DRAM）、rH2D/RH2D（远端 Host DRAM→本地 HBM），以及相关 D2rD/rD2D、同机 Host↔Device。方向名称以本版工具源码为准。读成功不能排除写异常，反之亦然。target 绑定某张卡不代表缓冲区在该卡 HBM，须核实实际内存类型。
- 分段大小包含业务实际字节数及小/大块、对齐/非对齐对照。历史候选为 54,728 B、64 KiB、1 MiB，不是固定业务参数。既做固定总字节数改变段大小，也做固定描述符数量改变段大小；记录 key 数、每 key 段数、地址 stride、连续/离散布局及对齐，避免把大小、数量、布局混为一因。
- 基础矩阵后，对异常组合比较单/多 worker、冷/热连接及适用的 UB/UBoE；保持节点对、卡、来源和版本一致，从日志核实生效协议。其他节点对的成功不能替代同节点对的受控比较。
- 记录整批耗时、实际字节数、带宽单位、各轮分布、失败/重试及校验方式。检查计时源码：按描述符摊销的 latency/p99 不是整批延迟或真实尾延迟；API 总耗时不是纯 DMA 耗时。
- 核实校验实际执行。有的 benchmark 即使设置 data_verify=true，写方向仍为 not_checked；返回码成功不能写成数据校验通过。需要接收端读回，明确完整校验或抽样位置/数量。
- 每轮确认上一轮进程退出、端口释放；原生调用可能不响应 SIGTERM，超时后核查并仅清理本轮残留进程。与残留流量重叠的结果排除，不混入正常统计。

## MMC 池化：保留真实接口与控制路径

HIXL 用于隔离底层传输，不能替代同版本 MMC 最小测试。写入使用实际 batch_put_from_layers，读取使用 batch_get_into_layers；保留 Meta、默认 lease、分配与位置查询。确认每个 key 的源/落点 service、rank→节点/物理卡映射及 DRAM/HBM 类型，再验证接收数据。先核对真实业务 key 数、每层字节数和地址布局；合成布局只能称为近似。

拆分 Meta 请求发出/收到、lease 截止、任务提交/实际开始、搬运提交/完成、最终检查；探针须评估扰动，无法观测的阶段明确留空。缩短 lease 导致过期是故障注入，只能证明错误机制，不能称为原业务根因复现。快写/慢写不能直接解释慢读或 lease 过期，需在同一业务时间线上关联。
