# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SLJIT is a platform-independent low-level JIT (Just-In-Time) compiler designed for translating bytecode into machine code. It supports multiple target architectures (x86, ARM, RISC-V, s390x, PowerPC, LoongArch, MIPS) and provides a low-level intermediate representation (LIR) for efficient code generation.

## Build System and Commands

### Primary Build Commands
- `make` or `make all` - Build main test executable and regex test
- `make clean` - Remove all compiled objects and executables
- `make examples` - Build tutorial example programs

### Test Commands
- `./bin/sljit_test` - Run the main SLJIT test suite
- `./bin/regex_test` - Run regex JIT tests

### Build Targets
- `bin/sljit_test` - Main test executable with comprehensive SLJIT tests
- `bin/regex_test` - Regex engine test with JIT compilation
- Tutorial examples in `bin/` directory (first_program, branch, loop, etc.)

### Cross-compilation
Use `CROSS_COMPILER` environment variable to specify cross-compiler:
```bash
make CROSS_COMPILER=arm-linux-gnueabihf-gcc
```

### Build Configuration
Key Makefile variables:
- `CC` - Compiler (default: gcc)
- `CFLAGS` - Compilation flags
- `WERROR` - Enable -Werror (default: enabled)
- `OPT_FLAGS` - Optimization flags (default: -O2)

## Architecture and Code Structure

### Core Components
- **sljit_src/sljitLir.h** - Main API header with comprehensive documentation
- **sljit_src/sljitLir.c** - Core LIR implementation
- **sljit_src/sljitNative*.c** - Architecture-specific code generators
- **sljit_src/allocator_src/** - Platform-specific memory allocators

### Key Concepts
- **LIR (Low-level Intermediate Representation)** - Platform-independent instruction set
- **Compiler Context** - Managed by `sljit_emit_enter` or `sljit_set_context`
- **Register Management** - Manual register and stack management required
- **Type System** - Strict typing with `int32_t` vs `intptr_t` distinction

### Register Types
- Integer registers: Store `int32_t` or `intptr_t` values
- Floating point registers: Store single or double precision values
- Type matching is mandatory between instruction expectations and register contents

### Memory Management
- All memory operations use `intptr_t` addressing
- Supports complex addressing modes (SLJIT_MEM1, SLJIT_MEM2)
- Platform-specific allocators handle executable memory

## Development Guidelines

### Function Generation
- Always start with `sljit_emit_enter` or `sljit_set_context` after compiler creation
- Use `sljit_emit_return` for function exits
- Context affects register mapping and local space allocation

### Type Safety
- Match register data types with instruction requirements
- Use SLJIT-defined types: `sljit_sw/uw`, `sljit_s32/u32`, etc.
- Type conversion instructions available for safe casting

### Testing
- Test files in `test_src/` provide comprehensive examples
- Tutorial sources in `docs/tutorial/sources/` show practical usage
- Use `sljit_test` for regression testing

### Documentation
Primary documentation sources:
1. `sljit_src/sljitLir.h` - Complete API documentation
2. `docs/general/architecture.md` - Architecture overview
3. `docs/tutorial/` - Step-by-step tutorials
4. Website: https://zherczeg.github.io/sljit/

## Common Patterns

### Basic Function Structure
```c
struct sljit_compiler *compiler = sljit_create_compiler(NULL);
sljit_emit_enter(compiler, options, args, scratches, saveds, fscratches, fsaveds, local_size);
// ... emit instructions ...
sljit_emit_return(compiler, SLJIT_MOV, src, srcw);
```

### Configuration Headers
- Use `SLJIT_HAVE_CONFIG_PRE=1` for custom pre-configuration
- Include `sljitConfigPre.h` and `sljitConfigPost.h` in test builds
- Platform detection handled automatically via `sljitConfigCPU.h`