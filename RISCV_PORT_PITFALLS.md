# Codex RISC-V Porting Pitfalls & Solutions
1. Prebuilt V8 Missing: codex-rs panics because no RISC-V v8 binaries exist. Fix: export V8_FROM_SOURCE=1
2. v8.fslock Permission: root ran it previously. Fix: sudo rm target/release/build/v8.fslock
3. Chrome Infra Network Block: ninja/gn download fails due to appspot.com block.

## 5. gn gen File Ownership Error
**Issue**:  fails with  on  in .
**Solution**: A previous test run as  created the  directory. Running  resolves it.

## 6. codex-bwrap pkg-config Missing RISC-V Libraries
**Issue**:  fails to compile because  cannot find  and  for the RISC-V target. The system only has them for .
**Solution**: 
1. Separate  architectures by setting  for the main repositories.
2. Add a new  specifically for  pointing to .
3. Run .
4. Install the required target development headers: Reading package lists...
Building dependency tree...
Reading state information....

## 5. gn gen File Ownership Error
**Issue**: gn gen fails with Permission denied (13) on mobileprovisions.filelist in target/release/gn_out.
**Solution**: A previous test run as root created the gn_out directory. Running sudo chown -R gateman:gateman target/release/gn_out resolves it.

## 6. codex-bwrap pkg-config Missing RISC-V Libraries
**Issue**: codex-bwrap fails to compile because pkg-config cannot find libcap and glib-2.0 for the RISC-V target. The system only has them for amd64.
**Solution**: 
1. Separate ubuntu.sources architectures by setting Architectures: amd64 i386 for the main repositories.
2. Add a new ubuntu-ports.sources specifically for Architectures: riscv64 pointing to http://ports.ubuntu.com/ubuntu-ports/.
3. Run sudo dpkg --add-architecture riscv64.
4. Install the required target development headers: sudo apt-get install -y libcap-dev:riscv64 libglib2.0-dev:riscv64.

## 7. libc++ Cross-Compilation Wordsize Error
**Issue**: Clang internal builder () fails with  while compiling  for  (host toolchain).
**Solution**: This is a known issue on multiarch systems where the 32-bit (i386) libc dev headers are missing. Running Reading package lists...
Building dependency tree...
Reading state information...
Package gcc-multilib is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source resolves the missing headers for the host tools build.

## 7. libc++ Cross-Compilation Wordsize Error
**Issue**: Clang internal builder (clang++) fails with fatal error: 'bits/wordsize.h' file not found while compiling alloc_error_handler_impl.cc for x86_64-unknown-linux-gnu (host toolchain).
**Solution**: This is a known issue on multiarch systems where the 32-bit (i386) libc dev headers are missing. Running sudo apt-get install -y gcc-multilib g++-multilib libc6-dev-i386 resolves the missing headers for the host tools build.

## 8. V8 Architecture Hardcoding (x64)
**Issue**: While generating Rust bindings with bindgen, V8 header  throws . The  configured V8 for x64 despite  targeting RISC-V.
**Solution**: V8's build system needs explicit target CPU arguments when cross-compiling. Setting  ensures GN configures V8 correctly.

## 8. V8 Architecture Hardcoding (x64)
**Issue**: While generating Rust bindings with bindgen, V8 header v8config.h throws error: Target architecture x64 is only supported on x64 and arm64 host. The gn gen configured V8 for x64 despite cargo targeting RISC-V.
**Solution**: V8's build system needs explicit target CPU arguments when cross-compiling. Setting V8_TARGET_CPU=riscv64 ensures GN configures V8 correctly.

## 9. GN Arguments Missing RISC-V Cross-Compilation
**Issue**: 's  natively adds  and  for ARM and AArch64 when cross compiling, but fails to do so for RISC-V, allowing GN to fall back to the host architecture (x64) and triggering the bindgen error.
**Solution**: Pass the correct CPU parameters to GN using .

## 9. GN Arguments Missing RISC-V Cross-Compilation
**Issue**: rusty_v8's build.rs natively adds target_cpu and v8_target_cpu for ARM and AArch64 when cross compiling, but fails to do so for RISC-V, allowing GN to fall back to the host architecture (x64) and triggering the bindgen error.
**Solution**: Pass the correct CPU parameters to GN using EXTRA_GN_ARGS="target_cpu=\"riscv64\" v8_target_cpu=\"riscv64\"".

## 10. Missing liblzma for RISC-V Cross-Compilation
**Issue**: Linker () fails with  when compiling  and . The linker skips the incompatible .
**Solution**: Install the  package so the linker has the correct target architecture libraries available.

## 10. Missing liblzma for RISC-V Cross-Compilation
**Issue**: Linker (riscv64-linux-gnu-gcc) fails with cannot find -llzma: No such file or directory when compiling codex-thread-manager-sample and codex-app-server-test-client. The linker skips the incompatible x86_64-linux-gnu/liblzma.so.
**Solution**: Install the liblzma-dev:riscv64 package so the linker has the correct target architecture libraries available.
