---
name: visualdsp-rules
description: >-
  Use when writing, refactoring, or reviewing Analog Devices VisualDSP++
  firmware (ccblkfn, Blackfin ADSP-BF5xx, .dpj/.ldf projects) such as DAQ5b,
  especially code that touches HALOS tasks, messages, timers, or semaphores,
  OSAL calls, interrupt handlers, error/status returns, buffer ownership, or
  DAQ12/DAQ24/BIRD_DOG build variants.
---

# VisualDSP Rules

Make failure, ownership, and concurrency visible in return values, names, and message contracts, using the project's own HALOS/OSAL primitives. Concrete code for every rule is in `patterns.md`.

## Toolchain facts

- VisualDSP++ 5.x `ccblkfn` with `-c++`: every translation unit, including `.c` files, compiles as pre-C++11 C++. Exceptions are off (`-eh`); RTTI may be on, which does not enable `throw`.
- Not available: `std::optional`/`expected`/`span`/`variant`/`unique_ptr`, `nullptr`, lambdas, `enum class`, `auto` type deduction, exceptions. Do not add the STL, Rust, a `Result` type, or a framework.
- The code is C++ in a C-like style: classes such as `HALOS_MESSAGE`, `SEMAPHORE`, `TIMER_CLASS`; `new`/`delete` (thread-safe via HALOS `redefined_new.cpp`); `u8`/`u16`/`u32`/`s32` from `HALOS\types.h`. Write new code the same way.
- Keep Windows backslash, mixed-case include paths. Read `.github\copilot-instructions.md` and `Config\main_defs.h` before changing tasks, memory, or peripherals.

## Return-value conventions (read before writing any `if`)

| API family | Success | Failure detail |
|---|---|---|
| HALOS (`halos_Send_Message`, `Install_Interrupt_Handler`, `TryEnterCriticalSection`, `Queue->Get`) | nonzero (`1`) | none; queue overflow sets `MESSAGE_QUEUE_OVERFLOW` in `Status()` |
| OSAL (`osalWaitEvent`, `osalLockMutex`, ...) | `true` | `osalGetLastError()` returns `osalERROR_*` |
| New module code you write | `STATUS_OK == 0` | the returned enum value |

Never test a HALOS result with `== 0` for success. `DAQ_OK`, `DAQ_ERROR` in `packet.h` are wire-protocol bytes; do not use them as internal status codes.

## Rules

### 1. Explicit failure
- New functions with expected failures return a module status enum and put results in an out-pointer.
- At an OSAL boundary, check the `bool`, read `osalGetLastError()` immediately (before any other OSAL call), and map it to your status. Do not let callers fish for last-error later.
- At a HALOS boundary, convert the nonzero-success `u8` into your status at the call site.
- Do not ignore return values of `halos_Send_Message`, OSAL waits, or allocation. `new` returns `NULL` on exhaustion (no exceptions); check it.
- Halt, assert, or `System_Reset()` only for bugs or unrecoverable hardware state.

### 2. Closed domains
- Model variants as `enum` plus `switch`, or a tagged union of POD members only.
- List every enumerator. `default` logs a fault (`Print_to_Sys_Log`) and returns an error; it is never a real case.
- Message dispatch on `hm.msg_id` uses an `if / else if` chain with `else` logging the unknown id. A missing `else` lets one message fall into a second test.

### 3. Ownership and lifetime
- One owner deletes. Pair `Foo_Create`/`Foo_Destroy` (or `new`/`delete` in the same module).
- `hm.msg` is a raw `void *`. For every message id, document in the header whether `msg` is a scalar, a pointer to sender-owned storage that must outlive delivery (`static` or pool), or a pointer the receiver takes ownership of and frees. Never send a pointer to a sender's stack variable.
- A message id registered `MESSAGE_BUFFERED_ONE_TIME` coalesces: a resend while queued overwrites `msg`, so earlier pointers are lost. Use `MESSAGE_BUFFERED_EVERY_TIME` when each send carries a distinct pointer or value.
- One message id can fan out to several queues. Transfer ownership through `msg` only when exactly one queue is registered for that id.
- Borrowed buffers are `(const u8 *data, u32 len)`; the callee does not free or retain them.
- `delete NULL` and `free(NULL)` are no-ops; do not add guards for them.
- Multi-step setup releases partial resources in reverse order on failure (single `cleanup:` label or early returns that undo). Do not rely on destructors for error-path cleanup.

### 4. Tasks and messages
- Follow the HALOS pattern: `Xxx_Init()` calls `CreateTask`, creates the `HALOS_MESSAGE_QUEUE`, registers message ids, and creates timers. Tasks live for the life of the firmware.
- The task loop drains its queue with `while (queue->Get(hm))`, dispatches, then `TaskSleep()`.
- Share nothing mutable across tasks except through messages. When state must be shared, keep it `static` in one module behind one `SEMAPHORE`, and comment which fields it guards.
- Respect startup order in `main.cpp`. Add new `*_Init()` calls only after their dependencies, and before `Enable_Multitasking()`.

### 5. Blocking and timeouts
- A message handler must not block indefinitely. For waits longer than a few ms, arm a `TIMER_CLASS` (single shot) whose message comes back to the same queue, and return.
- Bounded short waits use `Sleep(ms)` or OSAL calls with a finite `osalTime_ms_t` timeout; handle the timeout result.
- Do not add a task just to make one operation asynchronous.

### 6. Interrupt handlers
- Install with `Install_Interrupt_Handler`. Keep handlers short: acknowledge hardware, move data, signal a task.
- In ISR context (including callbacks invoked from an ISR): no `new`/`delete`, no `Sleep`, no semaphores, no file or network I/O, no `Print_to_Sys_Log`.
- Data shared with an ISR is `volatile`. Task-side multi-field access uses `u32 imask = cli(); ... sti(imask);` and keeps that section minimal.

### 7. Variants and composition
- Product variants are compile-time: `APPLICATION`/`BOOTLOADER_CODE`, `DAQ12` (else 24-channel), `BIRD_DOG`. Check the active `.dpj` configuration before assuming a path is live, and keep both sides of an `#if` compiling.
- Prefer small structs and free functions. No new base-class hierarchies or virtual interfaces for a single implementation.

### 8. Data layout and named constants
- On hot paths (ADC, PPI/DMA, packet FIFOs), use fixed arrays, ring buffers, and integer indexes over pointer graphs. Elsewhere, clarity wins.
- Do not move code into `fast_code`/L1 sections without measuring; L1 is small.
- Replace magic numbers and protocol bytes with `static const` or `enum`, matching the existing `static const u8 NAME = ...;` style.

## Verification

After editing, build every affected configuration, for example `make -f Application.mak AppBD5_Release` in `AppBD5\` and `make -f Application.mak Application_DAQ12` in `Application\`. Shared code under `Common\` or `System\` affects both variants and possibly `Bootloader\`. If a build cannot be run, say so.

## Limits
- These rules approximate safety; do not claim memory-safety or race-freedom.
- Make incremental, reviewable changes. Leave working code alone unless the task touches it.

## Output
After changes, list each significant change with the rule number that drove it, and the configurations you built.
