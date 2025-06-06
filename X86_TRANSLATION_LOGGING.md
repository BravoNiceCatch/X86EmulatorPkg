# X86 Translation Logging in X86EmulatorPkg

This document describes the detailed logging functionality added to X86EmulatorPkg for debugging and analyzing x86 to AArch64 instruction translation.

## Overview

The X86 translation logging feature provides comprehensive visibility into the instruction translation process, including:

- Individual x86 instruction decoding and analysis
- Translation Block (TB) creation and management
- TCG (Tiny Code Generator) code generation
- Generated AArch64 machine code output

## New Log Level

A new log level `CPU_LOG_X86_TRANS` has been added to control x86 translation logging:

```c
#define CPU_LOG_X86_TRANS  (1 << 10)
```

This can be used with the existing QEMU logging infrastructure.

## Usage

### 1. Enable Logging

```c
// Set log file
cpu_set_log_filename("/tmp/x86_translation.log");

// Enable x86 translation logging only
cpu_set_log(CPU_LOG_X86_TRANS);

// Or combine with other log types
cpu_set_log(CPU_LOG_X86_TRANS | CPU_LOG_TB_OP | CPU_LOG_EXEC);
```

### 2. String-based Configuration

```c
// Enable using string configuration
int mask = cpu_str_to_log_mask("x86_trans,op,exec");
cpu_set_log(mask);
```

### 3. Disable Logging (for performance)

```c
// Disable all logging for optimal performance
cpu_set_log(0);
```

## Log Output Format

### 1. X86 Instruction Decoding

```
[X86_TRANS] PC=0x401000 bytes=[48 89 e5] prefixes=0x0 insn=decoded
[X86_TRANS] PC=0x401003 bytes=[48 83 ec 10] prefixes=0x0 insn=decoded
[X86_TRANS] PC=0x401007 bytes=[89 7d fc] prefixes=0x0 insn=decoded
```

**Fields:**
- `PC`: x86 program counter address
- `bytes`: Raw instruction bytes (up to 15 bytes shown)
- `prefixes`: Instruction prefixes (REX, segment override, etc.)
- `insn`: Instruction mnemonic or status

### 2. Translation Block Creation

```
[X86_TRANS] TB: Creating new translation block for PC=0x401000 cs_base=0x0 flags=0x0 cflags=0x0
[X86_TRANS] TB: Allocated TB at 0x7f8b2c001000, code buffer at 0x7f8b2c002000, phys_pc=0x401000
[X86_TRANS] TB: Created TB at 0x7f8b2c001000, size=16 bytes, code_size=64 bytes, phys_pc=0x401000 phys_page2=0xffffffffffffffff
```

**Fields:**
- `PC`: Starting x86 address of the translation block
- `cs_base`: Code segment base address
- `flags`: Translation flags
- `cflags`: Compilation flags
- `TB address`: Memory address of the TranslationBlock structure
- `code buffer`: Memory address where generated AArch64 code is stored
- `size`: Size of original x86 code in bytes
- `code_size`: Size of generated AArch64 code in bytes

### 3. TCG Code Generation

```
[X86_TRANS] TCG: Starting code generation at buffer=0x7f8b2c002000
[X86_TRANS] TCG: Generated AArch64 code: addr=0x7f8b2c002000 size=64 bytes
[X86_TRANS] TCG: Machine code: d2800000 f9000fe0 d2800020 f9000be0 
[X86_TRANS] TCG:                d2800040 f90007e0 d65f03c0 00000000 
```

**Fields:**
- `buffer`: Starting address of code generation buffer
- `addr`: Address of generated code
- `size`: Size of generated AArch64 code
- `Machine code`: Hexadecimal dump of generated AArch64 instructions

## Implementation Details

### 1. Modified Functions

#### `disas_insn()` in `qemu/target-i386/translate.c`
- Added `log_x86_instruction()` helper function
- Captures instruction bytes before decoding
- Logs instruction details after successful decoding
- Logs illegal instruction attempts

#### `tcg_gen_code()` in `qemu/tcg/tcg.c`
- Logs start of code generation
- Logs generated code size and location
- Optionally dumps generated machine code in hex format

#### `tb_gen_code()` in `qemu/exec.c`
- Logs translation block creation start
- Logs TB allocation details
- Logs completed TB information

### 2. Performance Considerations

- All logging is wrapped in `unlikely(qemu_loglevel_mask(CPU_LOG_X86_TRANS))` checks
- When logging is disabled, performance impact is minimal (single branch prediction)
- Machine code dumping is limited to 256 bytes to prevent excessive output
- Instruction byte capture is only performed when logging is enabled

### 3. Debug vs Release Builds

The logging functionality works in both debug and release builds:
- Debug builds: Full logging capability with detailed output
- Release builds: Logging available but optimized for minimal performance impact

## Use Cases

### 1. Debugging Translation Issues

Enable logging to identify problematic x86 instructions:

```bash
# Enable logging and run problematic code
echo "x86_trans" > /tmp/log_mask
# Run your x86 code
# Check /tmp/x86_translation.log for issues
```

### 2. Performance Analysis

Analyze translation block creation patterns:

```bash
# Enable TB and translation logging
echo "x86_trans,exec" > /tmp/log_mask
# Run performance-critical code
# Analyze TB creation frequency and sizes
```

### 3. Instruction Coverage Analysis

Verify which x86 instructions are being translated:

```bash
# Enable detailed instruction logging
echo "x86_trans,in_asm" > /tmp/log_mask
# Run test suite
# Analyze instruction coverage
```

## Example Log Session

```
[X86_TRANS] TB: Creating new translation block for PC=0x401000 cs_base=0x0 flags=0x0 cflags=0x0
[X86_TRANS] TB: Allocated TB at 0x7f8b2c001000, code buffer at 0x7f8b2c002000, phys_pc=0x401000
[X86_TRANS] PC=0x401000 bytes=[48 89 e5] prefixes=0x0 insn=decoded
[X86_TRANS] PC=0x401003 bytes=[48 83 ec 10] prefixes=0x0 insn=decoded
[X86_TRANS] PC=0x401007 bytes=[c7 45 fc 00 00 00 00] prefixes=0x0 insn=decoded
[X86_TRANS] PC=0x40100e bytes=[8b 45 fc] prefixes=0x0 insn=decoded
[X86_TRANS] PC=0x401011 bytes=[c9] prefixes=0x0 insn=decoded
[X86_TRANS] PC=0x401012 bytes=[c3] prefixes=0x0 insn=decoded
[X86_TRANS] TCG: Starting code generation at buffer=0x7f8b2c002000
[X86_TRANS] TCG: Generated AArch64 code: addr=0x7f8b2c002000 size=128 bytes
[X86_TRANS] TCG: Machine code: d2800000 f9000fe0 d2800020 f9000be0 d2800040 f90007e0 d65f03c0
[X86_TRANS] TB: Created TB at 0x7f8b2c001000, size=19 bytes, code_size=128 bytes, phys_pc=0x401000 phys_page2=0xffffffffffffffff
```

## Troubleshooting

### 1. No Log Output

- Verify logging is enabled: `cpu_set_log(CPU_LOG_X86_TRANS)`
- Check log file permissions and path
- Ensure x86 code is actually being executed

### 2. Excessive Log Output

- Use selective logging: only enable `CPU_LOG_X86_TRANS` without other flags
- Limit execution to specific code sections
- Consider filtering log output with grep/awk

### 3. Performance Impact

- Disable logging in production: `cpu_set_log(0)`
- Use logging only during development and debugging
- Monitor log file size to prevent disk space issues

## Integration with Existing Tools

The x86 translation logging integrates seamlessly with existing QEMU debugging tools:

- Can be combined with GDB debugging
- Works with existing log analysis scripts
- Compatible with QEMU monitor commands
- Supports standard QEMU log rotation mechanisms
