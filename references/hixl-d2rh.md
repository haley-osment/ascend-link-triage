# 双机 UBoE D2rH 分段带宽

Agent 在两端调用 `bash scripts/hixl-d2rh`。入口从 skill 内的整包或分片解出预编译的 C++ `hixl_comm_bench`。发起机的 NPU HBM 通过 UBoE 同步写入目标机的 Host DRAM。Python 负责启动进程和生成报告。将完整 skill 复制到现场即可离线使用，无需 Git、编译器、pip 或外网。

## 现场提供

- 已验证的 Host 环境为 Linux aarch64、openEuler 24.03 LTS-SP4、glibc 2.38、CANN 9.2.1 和 Ascend950DT。其他环境需要兼容的 CANN 用户态库（ACL、runtime、HCOMM 等）、libstdc++、NPU 驱动、固件和实际可用的 UBoE 链路。容器还需能访问两端各自的 NPU、驱动库、拓扑和控制网。
- 现场提供 Python 3（参考 3.12）、Bash、GNU coreutils（含 timeout、sha256sum）、tar、gzip 和 ldd。默认入口能加载标准路径下的 CANN 9.2.1。自定义路径请先加载现场的 CANN 环境。TMPDIR（默认 /tmp）需可写、可执行，结果目录需可写。
- 每端选一张空闲卡。单卡对默认使用目标机的 16000、26000 和发起机的 16001，请确认对应端口空闲且两端互通。

## 包内依赖

包内有同批构建的 `bin/hixl_comm_bench`、`lib/libcann_hixl.so`、AICPU 算子描述 `libcann_hixl_kernel.json`、设备算子包 `cann-hixl-compat.tar.gz`、启动与报告脚本及上游许可。上游构建来源为 cann/hixl 提交 `885b294fadf2126263e7e9f4588edabd15a9715e`。

启动时，若现场描述包含所需算子且存在配套 tar，入口优先使用现场内核（日志显示 `installed`）。否则它会在临时目录建立仅本次进程可见的 CANN 路径，引用现场其余文件，并放入随包的描述及内核（`bundled`）。此过程使用普通文件和符号链接，无需 mount、sudo 或写系统目录。设备访问权限和现场签名策略仍然适用。若设备拒绝内核签名，应使用现场认可的匹配签名包并保留报错。

## 两端运行

Agent 将同一版本的完整 skill 复制或只读挂载到两台机器，在两端的 skill 根目录预检。

```bash
bash scripts/hixl-d2rh check
```

检查涵盖资源哈希、架构、动态库与重定位、启动器、设备算子描述和配套包。动态库缺项或 undefined symbol 请按输出核对现场 CANN 和驱动。检查通过后，用一组小流量传输确认设备能加载内核且两端能建链。

两端用同一个目标机控制 IP 和同一个 BLOCK（十进制字节），CARD 分别填本机卡号。结果目录使用各自可写的全新绝对路径。先在目标机运行并等 26000 开始监听，再在发起机运行。

```bash
# 目标机 Host
bash scripts/hixl-d2rh target "$TARGET_IP" "$CARD" "$BLOCK" "/tmp/hixl-target-$BLOCK-$(date +%s)"
# 发起机 Host
bash scripts/hixl-d2rh initiator "$TARGET_IP" "$CARD" "$BLOCK" "/tmp/hixl-initiator-$BLOCK-$(date +%s)"
```

每档两端退出后再换下一档。可依次测 54728、65536、131072、262144、1048576 字节。两端缓冲区都是 8 MiB，各自收到相同的 transfer_size，取不超过 8 MiB 的最大整块倍数。预算 6 轮，报告首轮、后五轮中位数和范围、实际传输字节数与失败数。记录两端内核来源、console.log 及发起端逐轮 CSV。超时后核对本次进程和端口是否释放。

CSV 应含 `benchmark=hixl_comm_bench`、`direction=D2rH`、`transport=uboe`、`initiator_memory=device` 和 `target_memory=host`。带宽单位为十进制 GB/s。此版写方向的内容校验状态是 `not_checked`。如需证明写入内容正确，还需在目标端读回比对。
