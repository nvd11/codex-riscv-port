# Codex CLI for RISC-V: 跨平台编译移植计划

## 🎯 背景与目标
OpenAI 官方开源的 [Codex CLI](https://github.com/openai/codex) 核心由 Rust 编写，且强依赖了包含完整 V8 引擎的 `rusty_v8` 库。目前官方 NPM 包及预编译的二进制文件均不支持 `riscv64` 架构，导致在 Starfive 等 RISC-V 开发板上无法直接运行。

本计划基于 GitHub Issue [#16272](https://github.com/openai/codex/issues/16272) 中 `spital` 的实验性方案，制定了一套详细的“宿主降维编译 -> 目标机原生重构”的两步走战略，旨在最终于 Starfive 上编译出满血版（带 V8）的 Codex 智能体。

---

## 🖥️ 硬件角色定义
*   **编译宿主机 (Host)**: Main PC (AMD Ryzen 7 5800H, IP: `10.0.1.173`, x86_64) - 负责重度交叉编译。
*   **目标运行机 (Target)**: Starfive (IP: `10.0.1.227`, riscv64) - 负责最终的原生编译和实际运行。
*   **跳板机 (Bastion)**: Moon (Radxa, IP: `100.115.214.26`, arm64) - 负责 SSH 隧道穿透。

---

## 🗺️ 阶段一：在 Main PC 上搭建交叉编译兵工厂
**目标：在强算力机器上准备好编译 RISC-V 架构 Rust 代码的工具链。**

1. **进入 Main PC**:
   ```bash
   ssh -J gateman@100.115.214.26 gateman@10.0.1.173
   ```
2. **安装基础 C/C++ 交叉编译工具**:
   ```bash
   sudo apt-get update
   sudo apt-get install -y gcc-riscv64-linux-gnu g++-riscv64-linux-gnu
   ```
3. **配置 Rust 目标架构**:
   ```bash
   rustup target add riscv64gc-unknown-linux-gnu
   ```
4. **配置 Cargo 的交叉链接器**:
   修改 `~/.cargo/config.toml`，显式指定连接器：
   ```toml
   [target.riscv64gc-unknown-linux-gnu]
   linker = "riscv64-linux-gnu-gcc"
   ```

---

## ⚔️ 阶段二：阉割版 Bootstrapping 交叉编译 (Host -> Target)
**目标：因为 `rusty_v8` 的交叉编译极其困难，我们首先在 Main PC 上编译一个“剥离 V8 特性”的阉割版 Codex。**

1. **获取源码**:
   ```bash
   git clone https://github.com/openai/codex.git
   cd codex
   ```
2. **禁用 `rusty_v8` 特性**:
   * 打开项目根目录或核心 crate 的 `Cargo.toml`。
   * 找到 `features` 定义，移除 `default` 列表中涉及 js/v8 的特性（如 `js_repl` 等）。
   * 确保依赖树中跳过编译 `rusty_v8`。
3. **执行交叉编译**:
   ```bash
   cargo build --target riscv64gc-unknown-linux-gnu --release --no-default-features
   ```
4. **空投到星光板**:
   ```bash
   scp target/riscv64gc-unknown-linux-gnu/release/codex gateman@10.0.1.227:/home/gateman/codex_bootstrap
   ```

---

## 🔥 阶段三：星光板上的涅槃重铸 (Native Build)
**目标：在 Starfive 上，利用刚才生成的阉割版 Codex 协助，辅以 Debian 本地环境，进行满血原生的编译。**

1. **登录 Starfive 并测试阉割版**:
   ```bash
   ssh -J gateman@100.115.214.26 gateman@10.0.1.227
   chmod +x /home/gateman/codex_bootstrap
   ./codex_bootstrap --version
   ```
2. **准备星光板的原生编译环境**:
   由于 `rusty_v8` 会在本地构建 V8，需要大量内存和完整 C++ 环境。
   ```bash
   sudo apt-get install -y build-essential pkg-config libglib2.0-dev libssl-dev ninja-build python3
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```
   *(注：星光板内存较小，强烈建议配置至少 4GB-8GB 的 Swap)*
3. **拉取源码进行满血编译**:
   ```bash
   git clone https://github.com/openai/codex.git
   cd codex
   # 在此阶段，可以让阉割版 codex_bootstrap 作为一个辅助 AI 工具，帮忙排查编译中的缺失依赖。
   cargo build --release
   ```

---

## 🏆 阶段四：验证与部署
1. **测试运行**:
   ```bash
   ./target/release/codex --enable collaboration_modes --enable code_mode
   ```
2. **确认架构和 V8 能力**:
   触发 JS REPL 工具测试 `rusty_v8` 是否被成功加载和执行：
   要求 Codex 执行这段测试代码：
   ```javascript
   text(JSON.stringify({ hasConsole: Object.hasOwn(globalThis, "console") }));
   exit();
   ```
3. **系统级安装**:
   ```bash
   sudo cp ./target/release/codex /usr/local/bin/
   ```

---
*Created by OpenClaw Moon Secretary 💋*