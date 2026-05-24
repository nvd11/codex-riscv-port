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