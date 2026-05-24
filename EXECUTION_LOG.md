# 操作记录 (Execution Log)

本文档记录了基于 `README.md` 中跨平台编译移植计划的实际执行过程、遇到的坑（Troubleshooting）以及指令的演变。

---

## 阶段一：在 Main PC 上搭建交叉编译兵工厂
### 1.1 安装交叉编译 C/C++ 工具链
我们首先在 Main PC (`10.0.1.173`, x86_64) 上安装了用于编译 RISC-V 架构 C/C++ 代码的 GCC/G++ 工具链。
**执行命令：**
```bash
sudo apt-get update
sudo apt-get install -y gcc-riscv64-linux-gnu g++-riscv64-linux-gnu
```
**结果：** 成功安装。

### 1.2 配置 Rust 工具链与目标架构
**执行命令：**
```bash
# 安装/更新 Rustup
curl --proto "=https" --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env
# 添加 RISC-V 目标架构
rustup target add riscv64gc-unknown-linux-gnu
```
**结果：** Rust 环境更新至 `1.95.0`，目标架构添加成功。

### 1.3 配置 Cargo 交叉编译器
**执行命令：**
```bash
mkdir -p ~/.cargo
echo -e "\n[target.riscv64gc-unknown-linux-gnu]\nlinker = \"riscv64-linux-gnu-gcc\"" >> ~/.cargo/config.toml
```
**结果：** 成功将 `riscv64-linux-gnu-gcc` 映射为编译 RISC-V 时的 Linker。

---

## 阶段二：阉割版 Bootstrapping 交叉编译 (Host -> Target)
### 2.1 拉取 Codex 官方源码
**尝试一（Git Clone）：**
由于 GitHub 访问受限，直接 `git clone` 失败；更换了 `mirror.ghproxy.com` 后因仓库太大再次超时阻塞。
**尝试二（直接下载压缩包并解压）：**
**执行命令：**
```bash
curl -L -o /tmp/codex.zip https://github.com/openai/codex/archive/refs/heads/main.zip
scp -o StrictHostKeyChecking=no /tmp/codex.zip gateman@10.0.1.173:/home/gateman/projects/codex-riscv-port/codex.zip
# 在 Main PC 上解压
cd /home/gateman/projects/codex-riscv-port/ && unzip -q codex.zip && mv codex-main codex
```
**结果：** 源码成功落盘。

### 2.2 首次尝试交叉编译（遇到网络阻塞）
**执行命令：**
```bash
cd codex/codex-rs && source $HOME/.cargo/env && cargo build --target riscv64gc-unknown-linux-gnu --release
```
**遇到的问题（Troubleshooting）：**
CPU 占用为 0。查看日志发现卡在：`Updating git submodule https://chromium.googlesource.com/libyuv/libyuv`。
原因是宿主机直连 Google 代码库网络不通。
**解决方案（注入内网代理）：**
利用局域网内的代理节点（Moon, IP `10.0.1.105` 上运行的 `mihomo`，端口 `7890`）：
```bash
export http_proxy=http://10.0.1.105:7890
export https_proxy=http://10.0.1.105:7890
```

### 2.3 第二次尝试交叉编译（遇到系统库链接错误）
在注入代理后，Cargo 开始疯狂下载依赖并成功拉满 CPU 进行编译，但随后遇到致命阻断。
**执行命令：**
```bash
cd codex/codex-rs && export http_proxy=... && cargo build --target riscv64gc-unknown-linux-gnu --release
```
**遇到的问题（Troubleshooting）：**
编译抛出异常：
```
Could not find directory of OpenSSL installation...
pkg-config has not been configured to support cross-compilation.
```
**根本原因：**
交叉编译器在宿主机（x86）上找不到专供目标架构（RISC-V）链接的 C 语言系统库（如 `libssl-dev`）。纯粹依靠配置 Sysroot 交叉编译大型 C/C++ 依赖工程极度不可靠。

---

## 阶段方案变更：Docker with QEMU 模拟原生环境
由于纯交叉编译（Cross-Compilation）遇到严重的 C 系统库链接风暴，我们决定改变策略：利用 Main PC 强大的 x86 算力，通过 `QEMU` 实时指令集翻译，在本地运行一个“原生的” RISC-V Docker 容器进行编译。

### 3.1 确认环境准备状态
1. 确认 Docker 已安装且正在运行（`Docker version 29.1.3`）。
2. **确认内核兼容性（核心排查）：**
   老板的 Main PC 运行的是经过精简编译的定制内核 (`Linux gateman-MoreFine-S500 7.0.10`)。跨架构 Docker 容器强依赖内核的 `binfmt_misc` 模块，而精简内核默认可能未加载。
   **排查与修复指令：**
   ```bash
   zcat /proc/config.gz | grep -i binfmt_misc  # 确认编译了模块：CONFIG_BINFMT_MISC=m
   sudo modprobe binfmt_misc                   # 手动挂载该核心模块
   ```

### 3.2 尝试注册 QEMU 翻译器（遇到网络阻塞）
**执行命令：**
```bash
sudo docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```
**遇到的问题（Troubleshooting）：**
执行失败，报错：`failed to resolve reference "docker.io/multiarch/qemu-user-static:latest"`。
由于 Docker Daemon 本身拉取镜像依然受到网络墙的限制，该指令超时失败。

**下一步计划（Pending）：**
需要为 Main PC 的 Docker Daemon 守护进程配置网络代理（指向 `10.0.1.105:7890`），以便成功拉取镜像并启动 RISC-V 容器环境。
### 3.3 为 Docker 配置代理并重启
**执行命令：**
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo sh -c 'echo -e "[Service]\nEnvironment=\"HTTP_PROXY=http://10.0.1.105:7890\"\nEnvironment=\"HTTPS_PROXY=http://10.0.1.105:7890\"\nEnvironment=\"NO_PROXY=localhost,127.0.0.1,10.0.1.0/24\"" > /etc/systemd/system/docker.service.d/http-proxy.conf'
sudo systemctl daemon-reload
sudo systemctl restart docker
```
**结果：** 成功为 Docker Daemon 注入了代理环境变量。

### 3.4 再次尝试注册 QEMU 翻译器（遇到内核网络栈特性缺失）
**执行命令：**
```bash
sudo docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```
**遇到的问题（Troubleshooting）：**
Docker 成功拉取到了镜像，但在启动容器时报错：
`failed to add the host (veth) <=> sandbox pair interfaces: operation not supported`
**根本原因：**
精简编译的定制内核去掉了网桥（Bridge/veth）的支持，导致 Docker 的默认桥接网络模式无法工作。

### 3.5 采用 Host 网络模式成功注册 QEMU
**执行命令：**
```bash
sudo docker run --rm --privileged --network host multiarch/qemu-user-static --reset -p yes
```
**结果：** 成功绕过网桥限制，输出 `Setting /usr/bin/qemu-riscv64-static as binfmt interpreter for riscv64`，这意味着 Main PC 已经成功获得了运行 RISC-V 架构程序的能力。

### 3.6 启动 RISC-V QEMU 容器
**执行命令：**
```bash
sudo docker rm -f codex-builder 2>/dev/null
sudo docker run -d --name codex-builder --platform linux/riscv64 --network host -v /home/gateman/projects/codex-riscv-port/codex:/workspace debian:13 tail -f /dev/null
```
**结果：** 成功启动。`docker exec codex-builder uname -m` 验证返回 `riscv64`。

### 3.7 安装系统编译依赖 (Target 环境内)
进入容器，安装编译 V8 及底层库必须的原生 C/C++ 工具链。
**执行命令：**
```bash
sudo docker exec -e http_proxy=... -e https_proxy=... codex-builder sh -c "apt-get update && apt-get install -y build-essential pkg-config libglib2.0-dev libssl-dev ninja-build python3 curl"
```
**结果：** `dpkg` 经过漫长的 QEMU 翻译解压，成功安装完毕。

### 3.8 安装 Target 原生 Rust 工具链
在容器内为 RISC-V 安装原生的 Rust 编译器（不再是宿主机的交叉编译链）。
**执行命令：**
```bash
sudo docker exec -e http_proxy=... -e https_proxy=... codex-builder sh -c "curl --proto \"=https\" --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y"
```
**结果：** Rust 1.95.0 (`riscv64gc-unknown-linux-gnu`) 成功安装并就位。

### 3.9 开始 Native 编译满血版 Codex (Target 环境内)
在配置好所有的网络代理、依赖和 C 系统库之后，我们在容器内部发起终极的 Release 编译命令。由于是借助 QEMU 执行的 Native 编译，此过程不再涉及交叉编译工具链，因此可以完美规避系统库缺失的报错。
**执行命令：**
```bash
sudo docker exec -e http_proxy=http://10.0.1.105:7890 -e https_proxy=http://10.0.1.105:7890 codex-builder bash -c "source \$HOME/.cargo/env && cd /workspace/codex-rs && cargo build --release"
```
**当前状态：**
编译进程已在后台成功启动。Cargo 正在拉取依赖索引。由于 QEMU 用户态指令翻译存在巨大的性能折耗，特别是 V8 引擎（`rusty_v8`）的编译，预计耗时将非常漫长（可能需要数小时）。后台进程正受到持续监控。

### 3.10 编译性能优化确认 (16线程满载)
在编译启动后，验证了 QEMU 容器内的资源分配和宿主机 CPU 占用情况：
*   **容器内逻辑核数**：执行 `docker exec codex-builder nproc` 返回 `16`，确认未受到 Docker `--cpus` 限制，完整继承了 5800H 的 16 线程。
*   **宿主机 CPU 负载**：执行 `top` 检查，宿主机 CPU 占用率达 `97.9% us`，Load Average 飙升至 `15.43`。
*   **结论**：`qemu-user-static` 完美支持多进程/多线程翻译，底层的多个 `rustc` 和 `cc1` (C/C++编译器) 进程已经全部分发到了所有 16 个核心上火力全开，最大化利用了 Main PC 的算力。

### 3.11 编译耗时预期与原理分析 (QEMU 性能损耗说明)
针对“为何 16 线程的强劲 x86_64 宿主机编译 Codex 仍需要超过一小时，而编译 Linux 内核仅需数分钟”的疑问，在此做技术归档：
1. **QEMU 用户态翻译损耗 (The Emulation Tax)**: 我们并非在进行传统的交叉编译（Native 编译器产出异构二进制），而是运行了一个完整的 RISC-V 容器。这意味着容器内的 `rustc`、`g++` 甚至 `cargo` 本身都是 **RISC-V 架构的二进制文件**。QEMU 必须在用户态将编译器的每一条指令进行“同声传译”转为 x86_64 指令执行，这通常会带来 **5倍到 10倍** 的性能折损。
2. **Rust 语言特性 (The Rust Tax)**: 与 C 语言（如编译 Linux 内核）相比，Rust 的编译包含复杂的生命周期检查、宏展开以及 LLVM 后端的深度优化，单核编译速度远慢于 C 语言。
3. **V8 引擎的体量**: Codex 依赖的 `rusty_v8` 包含了完整的 Google V8 引擎源码。这是一个庞大的 C++ 模板巨兽，即便在无模拟器的原生环境下满载编译，通常也需要数十分钟。
结合以上三点，长达数小时的编译时间在 QEMU-user-static 架构下属于正常的预期表现。
