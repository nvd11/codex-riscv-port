# Codex RISC-V Porting Pitfalls & Solutions

1. **Prebuilt V8 Missing**: codex-rs panics because no RISC-V v8 binaries exist. 
   **Fix**: export V8_FROM_SOURCE=1

2. **v8.fslock Permission**: root ran it previously. 
   **Fix**: sudo rm target/release/build/v8.fslock

3. **Chrome Infra Network Block**: ninja/gn download fails due to appspot.com block.
   **Fix**: Configure proper proxies (e.g. export http_proxy=http://10.0.1.105:7890).

4. **gn gen File Ownership Error**: gn gen fails with Permission denied (13) on mobileprovisions.filelist in target/release/gn_out.
   **Fix**: A previous test run as root created the gn_out directory. Running sudo chown -R gateman:gateman target/release/gn_out resolves it.

5. **codex-bwrap pkg-config Missing RISC-V Libraries**: codex-bwrap fails to compile because pkg-config cannot find libcap and glib-2.0 for the RISC-V target. The system only has them for amd64.
   **Fix**: 
   - Separate ubuntu.sources architectures by setting Architectures: amd64 i386 for the main repositories.
   - Add a new ubuntu-ports.sources specifically for Architectures: riscv64 pointing to http://ports.ubuntu.com/ubuntu-ports/.
   - Run sudo dpkg --add-architecture riscv64.
   - Install the required target development headers: sudo apt-get install -y libcap-dev:riscv64 libglib2.0-dev:riscv64.

6. **libc++ Cross-Compilation Wordsize Error**: Clang internal builder (clang++) fails with fatal error: 'bits/wordsize.h' file not found while compiling alloc_error_handler_impl.cc for x86_64-unknown-linux-gnu (host toolchain).
   **Fix**: This is a known issue on multiarch systems where the 32-bit (i386) libc dev headers are missing. Running sudo apt-get install -y libc6-dev-i386 resolves the missing headers for the host tools build.

7. **V8 Architecture Hardcoding (x64)**: While generating Rust bindings with bindgen, V8 header v8config.h throws error: Target architecture x64 is only supported on x64 and arm64 host. The gn gen configured V8 for x64 despite cargo targeting RISC-V.
   **Fix**: V8's build system needs explicit target CPU arguments when cross-compiling. Setting V8_TARGET_CPU=riscv64 ensures GN configures V8 correctly.

8. **GN Arguments Missing RISC-V Cross-Compilation**: rusty_v8's build.rs natively adds target_cpu and v8_target_cpu for ARM and AArch64 when cross compiling, but fails to do so for RISC-V, allowing GN to fall back to the host architecture (x64) and triggering the bindgen error.
   **Fix**: Pass the correct CPU parameters to GN using EXTRA_GN_ARGS=target_cpu="riscv64" v8_target_cpu="riscv64".

9. **Missing liblzma & openssl for RISC-V Cross-Compilation**: Linker (riscv64-linux-gnu-gcc) fails with cannot find -llzma / -lssl: No such file or directory. The linker skips the incompatible x86_64-linux-gnu versions.
   **Fix**: Install the target architecture packages: sudo apt-get install -y liblzma-dev:riscv64 libssl-dev:riscv64, and export PKG_CONFIG_ALLOW_CROSS=1 PKG_CONFIG_PATH=/usr/lib/riscv64-linux-gnu/pkgconfig.
