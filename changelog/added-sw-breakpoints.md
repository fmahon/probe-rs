Added software breakpoint support for ARM Cortex-M targets, enabling breakpoints in RAM-resident code where the FPB hardware unit cannot reach (e.g. RP2040 firmware running from RAM). The DAP and GDB servers automatically fall back to software breakpoints when hardware breakpoints are unavailable for the target address.

---

## PR Description

### feat: software breakpoint support for ARM Cortex-M

**Motivation:** I'm working on an RP2040 project with an A/B bootloader that copies firmware from flash to RAM and jumps to it. Since the FPB comparator on Cortex-M0+ only matches flash addresses (bits [28:2]), hardware breakpoints cannot be set in RAM. This made debugging impossible — no breakpoints at all in the running firmware.

This implementation is probably not perfect, but it works for my use case and could serve as a starting point if you're interested in integrating software breakpoint support.

References: Discussion #1067, Issue #627.

### How it works

- **BKPT injection:** Reads the original 16-bit Thumb instruction at the target address, saves it in a `HashMap<u64, u16>` in `CortexMState`, and writes `BKPT #0` (`0xBE00`) in its place.
- **Unified API:** `Core::set_breakpoint()` tries HW first; if the address is out of FPB range, it falls back to SW. `Core::clear_breakpoint()` tries HW then SW. This is transparent to the DAP and GDB servers.
- **Cleanup:** `Session::drop()` calls `clear_all_sw_breakpoints()` to restore original instructions before disconnecting.
- **Architecture support:** Implemented for ARMv6-M (Cortex-M0/M0+). ARMv7-M and ARMv8-M have the same structure but only as stubs — they could use the same approach with Thumb-2 awareness.

### What works

- Setting/clearing SW breakpoints at RAM addresses via DAP (VSCode) and GDB server
- Breakpoint halts correctly on Cortex-M0+ (RP2040) running code from RAM
- Tested with `probe-rs` DAP server + VSCode and with GDB server + `gdb-multiarch`

### Known limitations

- **Step-past:** When the core halts on a SW breakpoint, the current `run()` path does not yet restore the original instruction, single-step, and re-insert the BKPT. This means `continue` after hitting a SW breakpoint skips the instruction at that address. A proper restore-step-reinsert cycle is needed.
- **Thumb-2 only handles 16-bit:** The implementation saves/restores 16-bit instructions. 32-bit Thumb-2 instructions at the breakpoint address would need special handling.
- **No RISC-V:** Only ARM Cortex-M is implemented. RISC-V would need `EBREAK` (`0x00100073`) instead of `BKPT`.

### Changes

- `probe-rs/src/core.rs` — `CoreInterface` trait: `set_sw_breakpoint`, `clear_sw_breakpoint`, `sw_breakpoint_addresses`. `Core` public API: `set_breakpoint` / `clear_breakpoint` (unified HW+SW).
- `probe-rs/src/architecture/arm/core/mod.rs` — `sw_breakpoints: HashMap<u64, u16>` in `CortexMState`
- `probe-rs/src/architecture/arm/core/armv6m.rs` — Full implementation for ARMv6-M
- `probe-rs/src/architecture/arm/core/armv7m.rs` — Stub (returns `NotImplemented`)
- `probe-rs/src/architecture/arm/core/armv8m.rs` — Stub (returns `NotImplemented`)
- `probe-rs/src/session.rs` — `clear_all_sw_breakpoints()` + call in `Session::drop`
- `probe-rs-tools/.../dap_server/server/core_data.rs` — Use `set_breakpoint`/`clear_breakpoint` instead of HW-only
- `probe-rs-tools/.../gdb_server/target/breakpoints.rs` — Enable `SwBreakpoint` support
