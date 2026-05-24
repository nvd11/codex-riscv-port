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

### 3.12 架构设计探讨：为何 Agent 需要内嵌 JS 引擎执行文件操作？
在编译等待期间，对 Codex 的架构设计进行了深度的逻辑验证。
**质疑点**：既然 Agent 的核心和底层操作系统交互都是由 Rust 编写，为何在进行本地文件操作或计算时，要“脱裤子放屁”内嵌 V8 引擎让大模型写 JS 代码来执行，而不是直接给大模型暴露底层 Rust 写好的固定文件操作工具？
**架构论证结论**：
1. **表达能力的降维打击 (Turing Completeness)**：如果只提供固定的 Rust 工具（如 `read_file`, `grep_file`），当遇到“遍历目录下所有 JSON，将里面某个嵌套字段值提取出来加和，再把结果写回另一个 Markdown”这种复杂逻辑时，大模型必须进行成百上千次的 API 交互（拉取->思考->再拉取），Token 消耗和延迟将达到天文数字。而如果允许大模型写一段 JS（图灵完备语言），它只需**一次**思考，生成一段几百行的脚本，丢给 V8 瞬间跑出最终结果。
2. **大模型的心智舒适区**：当前所有的主流 LLM（GPT-4, Claude 3）在预训练时见得最多的“脚本语言”就是 JavaScript/TypeScript 和 Python。让它们写一段 JS 去操作数据，比让它们去死记硬背一套生涩的自定义 JSON API 工具协议的准确率要高得多。
3. **Rust 的编译时静态束缚**：Rust 是静态编译语言，无法在运行时接收大模型生成的 Rust 代码并动态执行（除非挂载一个庞大的 Rust 编译器环境，且启动耗时极长）。而 JS 引擎天然支持即时解释执行 (JIT)。
4. **沙盒隔离 (Sandboxing)**：大模型生成的代码是不可信的。直接让其通过 Shell 运行系统命令有毁灭性的安全风险（甚至可能执行 `rm -rf /`）。将其圈养在 V8 引擎内，Rust 宿主可以对其可调用的系统接口进行纳米级的阻断与拦截（仅开放特定的 `fs.readFile` 权限）。

### 3.13 架构深度辨析：大模型为何使用动态脚本 (JS/Python) 而非静态编译语言 (Rust/C++) 执行本地探测逻辑？
针对架构选择的进一步探讨：“大模型遇到动态探索需求时，为何不直接写一段 Rust 代码并在本地计算，难道仅仅因为 Rust 需要编译？”

**核心原因剖析：**
1. **即时执行与迭代延迟 (The Compilation Latency)**：大模型（Agent）探索本地项目时往往需要反复试错（写脚本 -> 看到错误输出 -> 修正脚本 -> 再次运行）。如果是 JS 或 Python，大模型扔出脚本后，数十毫秒内即可得到执行结果并开始下一次思考循环。如果是 Rust：大模型写完代码 -> 宿主机触发 `cargo build`（涉及拉取 crate 索引、漫长的宏展开、编译链接，通常数秒至数十秒） -> 执行。高昂的编译延迟会使得整个 Agent 工作流的耗时变得令人发指。
2. **环境依赖陷阱 (Dependency Management)**：如果大模型生成的 Rust 代码为了解析 JSON 而加上了 `serde_json` 依赖，那么不仅需要生成代码，还需要精确地管理和生成 `Cargo.toml` 配置文件，并等待这些依赖进行网络下载和漫长的全量编译。而 Codex 的内嵌 JS 环境中，官方已经提前将常用的文件操作和基础类型解析（或者预置了一些通用工具库）以系统全局接口的形式注入进了 V8 Sandbox 的上下文中（如注入全局 `text()` 或特制的 JSON 工具）。
3. **隔离机制缺失与越权风险 (Lack of Native Sandboxing)**：Rust 编译出来的是操作系统级的 Native 二进制文件。一旦它被系统执行，它将具备与其执行用户同等的所有底层系统权限。大模型生成的二进制文件有极大的“越权与破坏”风险。相反，JavaScript 运行在宿主（Agent 核心）控制的 V8 或 Deno Sandbox 内部。其发起的所有 syscall（文件读写、网络请求）都被限制在这个虚拟运行时中，宿主可以直接熔断危险操作。
4. **模型表达亲和度 (Model Proficiency)**：大量开源数据使得当前所有顶级大语言模型在生成 Python 和 JavaScript/TypeScript 这类“胶水脚本语言”去处理 JSON、正则、文本清洗等杂活时，一次性通过率和正确率远高于要求其写出带有生命周期和复杂错误处理（Result<T,E> / Borrow Checker）的 Rust 代码。

### 3.14 架构深度辨析：为何 Agent 弃用 Bash Shell 而内嵌 JS V8 环境处理本地探索？
**质疑点**：如果 Rust 有编译延迟且原生不安全，那为何不直接让大模型生成原生的 Bash / Shell 脚本？Shell 本就是系统原生的“胶水语言”，无需编译，且擅长处理文件。
**核心原因剖析：**
1. **跨平台兼容性的梦魇 (Cross-Platform Hell)**：Agent 工具被设计为跨平台使用。如果使用 Bash，那么在 Windows 系统下将直接瘫痪（除非强依赖 WSL 或 Git Bash）；即使在 Unix 体系内，macOS 默认的 BSD 工具链（如 `sed`, `awk`, `grep`）与 Linux 下的 GNU 工具链在语法和参数上存在大量让人抓狂的差异。大模型很难确保生成的 Bash 脚本能在所有用户的终端里无缝运行。而内嵌的 V8 JavaScript 环境是**绝对平台无关的**，同一段 JS 代码在 Windows, Mac 和 Linux 上的执行结果完全一致。
2. **结构化数据处理能力的灾难 (Structured Data Manipulation)**：Bash 擅长按行处理纯文本，但在面对现代工程中最核心的 **JSON, YAML, TOML 甚至 AST（抽象语法树）**时，Bash 的原生处理能力近乎为零（必须依赖外部工具如 `jq`，而用户机器上未必安装了 `jq`）。相比之下，JavaScript 原生自带 `JSON.parse`，对复杂嵌套数据的提取和聚合具有降维打击般的优势。
3. **安全逃逸控制与沙盒管控 (Sandboxing & Jailbreak Prevention)**：如果允许大模型使用 Bash，相当于直接给其开了一个通向底层操作系统的后门。Bash 可以轻易调用系统中任意危险的二进制文件（如 `curl` 下载恶意木马，`rm` 删除数据）。尽管可以对 Bash 命令进行字符串匹配过滤，但在 Bash 极度灵活的语法（混淆、拼接、反撇号、管道、重定向）面前，安全拦截器形同虚设，极易被“越狱”。而运行在 V8 引擎中的 JS 代码，其宿主（Rust 本体）可以实施 **"Default Deny"** 策略：除了显式注入的接口（如受限的 `readFile`），JS 代码即使想调用外部二进制也绝对做不到。
4. **状态隔离与副作用清理 (State Mutation & Side Effects)**：Bash 脚本在运行过程中可能会修改环境变量、残留临时文件、或者在后台遗留悬挂进程（Zombie Process）。而 V8 Engine 每次执行大模型代码时都可以分配一个全新的、干净的 `Isolate` 沙盒实例，执行完毕后内存即刻销毁，没有任何副作用残留，保证了极度的纯净和幂等性。

### 3.15 架构深度辨析：为何 Agent 内嵌 V8 (JavaScript) 而非 Python 作为动态执行引擎？
**质疑点**：Python 同样是解释型脚本语言，跨平台，处理 JSON 和文本能力极强，且大模型本身最擅长编写 Python 代码。为何 Codex 不选择嵌入 Python？
**核心原因剖析：**
1. **终极沙盒与安全性 (The Ultimate Sandbox)**：这是最致命的区别。V8 引擎（JavaScript）从诞生之初（为 Chrome 浏览器服务）就是为了**“安全执行全网互不信任的恶意代码”**而设计的。通过 V8 的 `Isolate` 机制，Rust 宿主可以在几毫秒内拉起一个极度受限的内存黑盒，精确控制内存上限、CPU 运行超时，并且默认切断一切系统级 API。而 CPython **根本没有原生的安全沙盒**。Python 中的隔离极其脆弱，大模型生成的 Python 代码很容易通过 `ctypes`、`os.system` 等底层模块击穿所谓的限制。要安全执行 Python，通常需要拉起 OS 级别的 Docker 容器或 Firecracker，这带来了极大的资源消耗和启动延迟。
2. **“单文件可执行”的部署极致 (Single Binary Deployment)**：Agent 工具（如 Codex）追求极致的安装体验：下载一个可执行文件直接运行。Rust 与 `rusty_v8` 结合，可以将整个 V8 引擎静态链接进最终的二进制文件内，无需依赖外界环境。而 CPython 的嵌入极其痛苦：它不仅需要 C 运行库，还需要庞大且碎片的 Python 标准库（数十兆的 `.py` 文件）。如果依赖用户机器上的宿主 Python，则会立刻陷入“Python 版本地狱（3.8 vs 3.12）”和环境变量冲突中。
3. **毫秒级状态销毁与并发 (Stateless & Concurrency)**：由于 `v8::Isolate` 的轻量级特性，Codex 甚至可以为大模型的每一步思考瞬间开辟多个并行的 JS 沙盒，互不干扰，用完即毁。而 CPython 存在全局解释器锁（GIL），且在单一进程内反复重置解释器状态极易发生内存泄漏或状态污染。
4. **JIT 性能碾压 (Performance)**：V8 引擎自带极其强悍的即时编译器 (JIT)。在执行大量的字符串拼接、正则匹配、AST 遍历等 Agent 常见任务时，V8 的执行速度通常是传统 CPython 字节码解释器的数倍甚至数十倍。

### 3.16 终局之战探讨：能否在 x86 架构上直接交叉编译 V8 (V8_FROM_SOURCE=1)？
**质疑点**：既然在 QEMU (RISC-V 模拟) 中原生编译 V8 极其缓慢，而预编译库又 404，那我们能否退回 x86 宿主机，通过 `V8_FROM_SOURCE=1` 环境变量，让强大的 5800H 直接把 V8 交叉编译为 RISC-V 目标代码？
**架构论证与可行性：**
理论上完全可行，且这是 Google 官方（Chromium/Android 团队）的常规做法。V8 使用的构建系统（`gn` 和 `ninja`）对交叉编译支持极好。
但工程实践上，我们将面临一个经典的“交叉编译死锁悖论”：
1. **速度极快**：如果回到 x86 宿主机交叉编译，V8 将发挥 5800H 100% 的原生性能，十几分钟即可编译完毕。
2. **依赖风暴重现**：退回 x86，意味着我们将再次面对 2.3 节中击溃我们的那个报错 —— `OpenSSL` 系统库找不到。
**终极破局方案（Debian Multiarch 交叉环境）：**
如果坚决要走 x86 交叉编译路线，我们需要在宿主机（或一个纯 x86 的 Docker 容器）中开启 `dpkg --add-architecture riscv64`，直接安装 `libssl-dev:riscv64` 等目标架构的开发包，并配置复杂的 `PKG_CONFIG` 环境变量，打造一个完美的“异构 Sysroot 虚拟系统根”，最后加上 `V8_FROM_SOURCE=1` 进行总攻。
