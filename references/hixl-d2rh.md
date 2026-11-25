# 双机 UBoE D2rH 分段带宽

默认调用 skill 自带的 `scripts/hixl-d2rh`。测试通过 C++ 工具同步地从发起机的 NPU HBM 写到目标机的 Host DRAM。Python 只负责启动两端进程和整理报告。离线包可直接复制到两端，现场无需 Git、编译器、pip 或公网。

## 现场提供

- 两端均为 Linux aarch64，系统 glibc、libstdc++ 及 CANN 用户态与构建版本二进制兼容。参考运行镜像是 openEuler 24.03 LTS-SP4、glibc 2.38、Python 3.12.16、CANN 9.2.1。可复用满足这些条件的业务容器，给包只读挂载并提供单独可写的结果目录。
- 兼容的昇腾 NPU、驱动和固件、可用测试卡及 UBoE 链路。两端的目标进程都需能访问各自的 NPU、通信资源、驱动库和现场拓扑。目标端控制 IP 可从发起端访问。此版单卡配置默认使用 16000 和 26000 两个端口，确认空闲可达。
- Python 3（参考版本 3.12）、Bash、GNU coreutils（含 timeout、sha256sum）、tar 和 ldd。加载现场的 CANN 环境。自定义安装路径时在运行前设置 ASCEND_HOME_PATH 和所需驱动库搜索路径。
- 包内提供同批构建的 hixl_comm_bench 与 libcann_hixl.so，以及启动、平台检测、报告脚本、上游许可证和声明。CANN 的 ACL、runtime、HCOMM、metadef、日志和 profiler 等库，以及驱动的 libascend_hal.so，由现场安装提供。具体加载路径见两端 check 输出。驱动与固件按现场版本配套。CPU 型号相同仍需逐端验证。

## 运行

由 Agent 将本机已安装的 skill 目录通过现有连接复制或只读挂载到两台测试机（已部署则复用），在各端的 skill 根目录运行下列命令。入口自动从随附资源加载已编译工具。运行期间使用可执行的临时目录，如 /tmp 挂载为 noexec，请设置 TMPDIR 到可执行的可写目录。全程无需连接外部仓库或安装软件。

```bash
bash scripts/hixl-d2rh check
```

check 检查包内文件哈希、架构、动态库和重定位、启动器以及二进制帮助入口。若有 missing library、undefined symbol 或版本错误，保存输出，比对现场 CANN、驱动、libstdc++ 和 glibc；使用匹配环境或提供对应构建。

两端各设置 TARGET_IP 为**目标机同一个控制 IP**，CARD 为**本机**可用卡号，BLOCK 为十进制字节数。先在目标机运行第一条，等待它监听，再在发起机运行第二条。结果目录需用各自可写文件系统的全新绝对路径。

```bash
# 目标机
bash scripts/hixl-d2rh target "$TARGET_IP" "$CARD" "$BLOCK" "/tmp/hixl-target-$BLOCK-$(date +%s)"
# 发起机
bash scripts/hixl-d2rh initiator "$TARGET_IP" "$CARD" "$BLOCK" "/tmp/hixl-initiator-$BLOCK-$(date +%s)"
```

每次取一档，两端 BLOCK 相同。可依次测 54728、65536、131072、262144、1048576。每轮传输量为不超过 8 MiB 的最大整块倍数，启动时打印实际字节数与描述符数量。默认 6 轮，分别报告首轮和后五轮的中位数、范围与失败数。8 MiB 是短时对照配置，复测业务时按需调整两端 buffer_size、transfer_size 与超时。固定描述符数量的对照应另列一组。完成一档、两端进程退出后再启动下一档。超时后检查本次进程与端口是否释放。

核对 initiator CSV 中 benchmark=hixl_comm_bench、transport=uboe、direction=D2rH、initiator_memory=device、target_memory=host，结合实际配置和两端日志确认通路。带宽采用十进制 GB/s。保留两端 console.log 与逐轮 CSV，记录异常或未测档位。此版内置内容比对覆盖读方向，D2rH 写结果的 not_checked 应如实报告；需要证明写入正确时，另做目标端读回验证。

## 版本与证据

上游源码 cann/hixl 提交 `885b294fadf2126263e7e9f4588edabd15a9715e`。参考镜像 ID `sha256:1900f2e158a2887966daae07b04cddcdab939867936ca1a881a180083f386497`。历史双机 C++ CSV 覆盖 UBoE D2rH 的 64 KiB 和 128 KiB，写数据均未校验。归档中的二进制和同批库来自该环境的构建目录。归档已在参考镜像中断网解压并检查加载。客户端真实 D2rH 带宽以该环境的双端结果为准。
