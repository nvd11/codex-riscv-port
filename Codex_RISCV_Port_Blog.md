# 绝地求生：如何在 2026 年把 OpenAI Codex 强行交叉编译到 RISC-V 架构

OpenAI 官方开源的 Codex CLI 是当前极为强大的本地代码 Agent，但官方却唯独没有提供 RISC-V 架构的预编译版本。
为了在我们的 Starfive 星光板上跑起这个大杀器，昨晚我们曾试图在 QEMU 模拟器中偷懒编译，结果被 V8 引擎庞大的源码量和指令翻译开销拖到内存爆炸、进程卡死。
痛定思痛后，我们决定采用最硬核的方式——**基于 x86_64 强大算力的真实交叉编译 (Cross-Compilation)**。

在这场战役中，我们历经重重险阻，连续趟平了 10 个匪夷所思的“天坑”，最终拿下了 138MB 的纯正 RISC-V ELF 可执行文件！以下是本次战役的复盘记录。

---

## 一、 参战主机架构矩阵

本次交叉编译任务是一场由三台物理机协同打配合的联合作战：

1. **💪 算力主C (编译机)：`10.0.1.173` (AMD Ryzen 7 5800H)**
   老板的主力开发机，性能怪兽。负责承担所有重度编译（特别是十几万个 C++ 文件的 V8 引擎）。所有的坑和命令都在这台机器上执行。
2. **🛡️ 辅助辅助 (跳板/代理机)：`10.0.1.105` / `100.115.214.26` (Radxa Moon)**
   内网的 Bastion Host。由于编译过程中需要拉取被 GFW 屏蔽的 Google/Chrome Infra 源码，它在内网的 `7890` 端口提供了至关重要的魔法代理。
3. **🎯 最终归宿 (目标板)：`10.0.1.227` (Starfive 星光板)**
   原汁原味的纯 RISC-V 硬件，用于接收最终编译产物并验证运行。

---

## 二、 正常交叉编译的完整基石：环境准备与前置命令

如果你也想在主力机上对 Rust 进行 RISC-V 的交叉编译，除了要绕过后面的各种坑，第一步必须建立完整的交叉编译环境。以下是基建所需的步骤：

### 1. 配置 Ubuntu 的多架构支持 (Multi-arch)
为了在 amd64 主机上安装 riscv64 的依赖包，必须首先区分主仓库和 Ports 仓库：

```bash
# 1. 拆分主源只支持 amd64/i386，防止 apt 混淆报错
sudo sed -i '/^Architectures:/d' /etc/apt/sources.list.d/ubuntu.sources
sudo sed -i '/^Components:/a Architectures: amd64 i386' /etc/apt/sources.list.d/ubuntu.sources

# 2. 为 riscv64 创建专门的 Ubuntu Ports 源
cat << 'SOURCES_EOF' | sudo tee /etc/apt/sources.list.d/ubuntu-ports.sources
Types: deb
URIs: http://ports.ubuntu.com/ubuntu-ports/
Suites: resolute resolute-updates resolute-backports resolute-security
Components: main restricted universe multiverse
Architectures: riscv64
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
SOURCES_EOF

# 3. 添加架构支持并更新索引
sudo dpkg --add-architecture riscv64
sudo apt-get update
```

### 2. 安装 GCC 交叉编译工具链与底层库
你需要一个能够将代码翻译成 RISC-V 指令的链接器（Linker）和一系列目标平台的基础 C/C++ 共享库：

```bash
# 安装 GNU 官方的 riscv64 交叉编译器
sudo apt-get install -y gcc-riscv64-linux-gnu g++-riscv64-linux-gnu

# 安装我们编译 Codex 和 V8 引擎过程中需要的各种系统级底层依赖（包含 32 位兼容库）
sudo apt-get install -y \
    libc6-dev-i386 \
    libcap-dev:riscv64 \
    libglib2.0-dev:riscv64 \
    libssl-dev:riscv64 \
    liblzma-dev:riscv64
```

### 3. Rust 的 Target 添加
让 Cargo 知道我们要构建的目标架构：
```bash
rustup target add riscv64gc-unknown-linux-gnu
```

当这套交叉编译基建搭好后，我们才正式有了挑战 Codex 的资本。

---

## 三、 披荆斩棘：我们在交叉编译中踩平的十大天坑

哪怕基建做好了，V8 庞大的构建系统和复杂的 C++ 互操作依然让我们迎面撞上了十大天坑：

### 坑 1：Prebuilt V8 Missing（找不到预编译的 V8）
**Issue**: `codex-rs` 依赖的 `rusty_v8` 默认会去网上拉取预编译库，但官方压根没发过 RISC-V 版的库，直接 Panic。
**Solution**: 强制要求从源码编译 V8：`export V8_FROM_SOURCE=1`

### 坑 2：v8.fslock Permission Denied（锁文件权限被拒）
**Issue**: 之前某次测试用了 `sudo`，导致生成的 `target/release/build/v8.fslock` 归 root 所有，普通用户再去编译直接报 `Permission denied`。
**Solution**: 删掉流氓锁文件：`sudo rm target/release/build/v8.fslock`

### 坑 3：Chrome Infra Network Block（构建工具链被墙）
**Issue**: `ninja_gn_binaries.py` 去 `appspot.com` 下载 GN 和 Ninja 时被 GFW 拦截，报 `Connection reset by peer` 和 `Network is unreachable`。
**Solution**: 指向内网的魔法代理：
```bash
export http_proxy=http://10.0.1.105:7890
export https_proxy=http://10.0.1.105:7890
```

### 坑 4：gn gen File Ownership Error（又见权限拦路虎）
**Issue**: GN 在生成配置时，试图写入 `gn_out/mobileprovisions.filelist` 失败，原来 `gn_out` 整个目录又是 root 的。
**Solution**: 夺回目录控制权：`sudo chown -R $USER:$USER target/release/gn_out`

### 坑 5：codex-bwrap pkg-config Missing RISC-V Libraries（底层沙盒缺库）
**Issue**: 编译 `codex-bwrap` 时，`pkg-config` 报错找不到 RISC-V 的 `libcap` 和 `glib-2.0`。
**Solution**: 这就是前文中为何我们要配置 `ubuntu-ports.sources` 并安装 `:riscv64` 依赖包的原因。配置好多架构源即可解决。

### 坑 6：libc++ Cross-Compilation Wordsize Error（编译器自身的 32 位套娃）
**Issue**: Clang 内部在编译自己用的代码生成器时，报 `fatal error: 'bits/wordsize.h' file not found`。因为工具链需要用到主机的 32位 i386 头文件来兼容。
**Solution**: 前文中的 `sudo apt-get install -y libc6-dev-i386` 解决了此问题。

### 坑 7 & 8：V8 Architecture Hardcoding & GN Arguments Missing（V8 架构写死与漏写）
**Issue**: bindgen 解析 `v8config.h` 时报错：`Target architecture x64 is only supported on x64 and arm64 host`。尽管 Cargo 指定了 RISC-V，但由于 `rusty_v8` 源码里漏了 `riscv64` 的判断分支，导致 V8 构建系统自己偷偷退回到了宿主架构（x64）。
**Solution**: 下猛药，通过后门参数强行把架构传递给 GN：
```bash
export EXTRA_GN_ARGS='target_cpu="riscv64" v8_target_cpu="riscv64"'
```

### 坑 9 & 10：Missing openssl & liblzma for RISC-V Cross-Compilation（链接库找不到）
**Issue**: 终于熬到最后的 Linking 阶段，`riscv64-linux-gnu-gcc` 抱怨 `cannot find -lssl` 和 `-llzma`。它嫌弃 x86 主机自带的库“incompatible”跳过了。
**Solution**: 除了前文安装好的 `:riscv64` 库之外，还必须显式告诉 `pkg-config` 允许交叉编译，并指向正确的包路径：
```bash
export PKG_CONFIG_ALLOW_CROSS=1
export PKG_CONFIG_PATH=/usr/lib/riscv64-linux-gnu/pkgconfig
```

---

## 四、 胜利会师：见证奇迹的最终构建指令

在搞定了基建环境、代理、架构冲突和所有的目标库后，我们清除了僵尸进程死锁，注入了完美的魔法变量，发起了最后的终极决战：

```bash
cd codex/codex-rs
source ~/.cargo/env

export http_proxy=http://10.0.1.105:7890
export https_proxy=http://10.0.1.105:7890
export V8_FROM_SOURCE=1
export EXTRA_GN_ARGS='target_cpu="riscv64" v8_target_cpu="riscv64"'
export CARGO_TARGET_RISCV64GC_UNKNOWN_LINUX_GNU_LINKER=riscv64-linux-gnu-gcc
export PKG_CONFIG_ALLOW_CROSS=1
export PKG_CONFIG_PATH=/usr/lib/riscv64-linux-gnu/pkgconfig

cargo build --release --target riscv64gc-unknown-linux-gnu
```

**`Finished release profile [optimized] target(s) in 31m 15s`** 
经过半小时漫长的编译与链接合体，终端终于弹出了这句令人泪目的成功提示！

## 五、 成果验收：推送到 RISC-V 星光板

打包刚出炉的 5 个二进制组件：
```bash
tar -czvf codex-riscv64.tar.gz -C target/riscv64gc-unknown-linux-gnu/release codex codex-app-server codex-exec codex-tui codex-mcp-server
```

通过跳板机将压缩包直接 SCP 到远端的星光板：
```bash
sshpass -p '******' scp -o StrictHostKeyChecking=no codex-riscv64.tar.gz gateman@10.0.1.227:/home/gateman/codex-riscv64.tar.gz
```

在星光板 (`10.0.1.227`) 上解压并执行：
```bash
mkdir -p /home/gateman/codex-riscv
tar -xzvf /home/gateman/codex-riscv64.tar.gz -C /home/gateman/codex-riscv
/home/gateman/codex-riscv/codex --version
```
**原生输出结果**：`codex-cli 0.0.0`
（注：源码 `Cargo.toml` 默认版本号占位符即为 `0.0.0`，对于从源码强行编译的版本这是正常现象）。

至此，OpenAI Codex 跨平台移植 RISC-V 的战役，大获全胜！🚀
