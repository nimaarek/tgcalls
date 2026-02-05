# libFuzzer harness layout for `tgcalls`

This document proposes a concrete fuzzing layout for this repository, focused on Telegram call signaling and call setup surfaces exposed through the C++ binding.

## Why these targets

The Python extension exposes call-signaling entrypoints through `NativeInstance`:

- `startCall(...)`
- `receiveSignalingData(...)`
- `setSignalingDataEmittedCallback(...)`

Those are exported in the binding and route data to Telegram's native call stack, making them the best first fuzz surfaces.

## Directory layout

```text
tgcalls/
  fuzz/
    CMakeLists.txt
    common/
      FuzzInit.h
      FuzzInputReader.h
      FuzzInputReader.cpp
      SeedCorpusNotes.md
    targets/
      signaling_packet_fuzzer.cc
      signaling_sequence_fuzzer.cc
      startcall_descriptor_fuzzer.cc
  fuzz/corpus/
    signaling_packet/
    signaling_sequence/
    startcall_descriptor/
```

You can also keep corpus under repo root (`fuzz/corpus/...`) if preferred by CI.

## Target 1: `signaling_packet_fuzzer` (highest priority)

**Goal:** mutate one signaling blob and feed it into a minimally initialized call instance.

### Harness shape

1. Build one `NativeInstance` once per process (or per iteration if stability requires).
2. Provide deterministic dummy `RtcServer` list and fixed 256-byte auth key.
3. Call `startCall(...)` with outgoing/incoming mode from fuzz data.
4. Parse the fuzz buffer into one packet (bounded size).
5. Call `receiveSignalingData(packet)`.

### Suggested entry function

```cpp
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
  // parse -> init call -> receiveSignalingData
  return 0;
}
```

### Useful guardrails

- Cap packet length (for example `<= 64 KiB`) to avoid pathological allocation.
- Wrap only expected runtime exceptions from harness plumbing (not import-time operations).
- Skip empty packet early.

## Target 2: `signaling_sequence_fuzzer` (stateful)

**Goal:** fuzz ordering/state transitions by sending multiple packets.

### Input format

Use a tiny binary framing format:

- `u8 packet_count` (0..16)
- repeated:
  - `u16 len`
  - `len` bytes payload

### Execution

1. Initialize `NativeInstance` once.
2. `startCall(...)` once.
3. Replay `packet_count` blobs via `receiveSignalingData(...)`.
4. Optionally toggle network type / mute / restart audio device between packets (if exposed safely).

This catches parser + state-machine bugs that single-packet fuzzing misses.

## Target 3: `startcall_descriptor_fuzzer` (constructor/config path)

**Goal:** fuzz inputs used by `startCall(...)` itself.

### Mutated fields

- Number of RTC servers (bounded, e.g. 0..8)
- Server host strings (UTF-8/invalid bytes normalized to safe strings)
- Port values
- `isTurn` / `isStun` combinations
- Username/password lengths
- `isOutgoing` flag
- Auth key bytes (always exactly 256 bytes by deriving from input)

### Expected value

This exercises call descriptor assembly and server-list handling before signaling data is processed.

## Corpus seeding plan

Seed each target with both **structure-aware** and **chaos** samples.

### `signaling_packet` corpus

Start with:

- Empty (`""`)
- 1-byte, 2-byte, 3-byte packets
- All-zero 64B / 512B
- All-`0xFF` 64B / 512B
- Valid-ish packet captures (if available from local test harness)

### `signaling_sequence` corpus

- 0 packets
- 1 packet with tiny payload
- 2 packets with same payload
- 4 packets with increasing lengths
- A captured real sequence from test traffic (redacted)

### `startcall_descriptor` corpus

- 0 servers
- 1 STUN only
- 1 TURN only
- Mixed STUN+TURN entries
- Max bounded entries with long credentials

### Corpus growth workflow

1. Run each fuzzer with `-merge=1` into canonical corpus.
2. Keep crashers in `fuzz/artifacts/<target>/`.
3. Minimize with `-minimize_crash=1`.

## Sanitizer flags

For Linux/macOS Clang fuzz builds:

- **Required for first pass:**
  - `-fsanitize=fuzzer,address,undefined`
- **Recommended follow-up runs:**
  - `-fsanitize=fuzzer,no-link` for library objects + link in fuzz target
  - `-fsanitize=integer` (if toolchain supports and noise is manageable)

Recommended runtime options:

```bash
ASAN_OPTIONS=detect_leaks=1:strict_string_checks=1:check_initialization_order=1
UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=1
```

For long fuzzing jobs, prefer:

- `-max_len=65536`
- `-timeout=10`
- `-rss_limit_mb=4096`

## CMake integration sketch

Add an option in root CMake:

- `option(TGCALLS_ENABLE_FUZZING "Build fuzz targets" OFF)`
- `if(TGCALLS_ENABLE_FUZZING) add_subdirectory(tgcalls/fuzz) endif()`

In `tgcalls/fuzz/CMakeLists.txt`:

1. Detect Clang and libFuzzer support.
2. Build a reusable harness support library from `common/*`.
3. Add 3 executables for targets.
4. Link target binaries to the same core dependencies used by the extension where possible.
5. Apply sanitizer/fuzzer flags only for fuzz targets.

## CI suggestions

- Add one quick smoke run per target (`-runs=200`) on PRs.
- Run longer jobs nightly with corpus upload/download.
- Upload sanitizer reports + minimized crash artifacts.

## Windows note

`libFuzzer` is best supported with Clang/LLVM toolchains. For Windows fuzzing:

- Prefer **clang-cl + AddressSanitizer** for local fuzz-like mutation harnesses.
- For pure MSVC, keep deterministic unit-style repro runners for crashing inputs.

The normal project Windows output is still the Python extension module (`tgcalls.pyd`, a DLL), not a standalone EXE.
