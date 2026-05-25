# Codex RISC-V Porting Pitfalls & Solutions
1. Prebuilt V8 Missing: codex-rs panics because no RISC-V v8 binaries exist. Fix: export V8_FROM_SOURCE=1
2. v8.fslock Permission: root ran it previously. Fix: sudo rm target/release/build/v8.fslock
3. Chrome Infra Network Block: ninja/gn download fails due to appspot.com block.
