# Known Bugs and Issues

This document tracks bugs found in the esp-hal codebase that may cause Load Access Faults or other issues on ESP32-C3 (RISC-V).

## Load Access Fault Bugs

### BUG-001: WaitQueue::from_ptr lacks null check

**File:** `esp-rtos/src/esp_radio/mod.rs:169`

**Status:** FIXED

**Severity:** High

**Description:**
The `WaitQueue::from_ptr` function dereferences the `WaitQueuePtr` without checking if it's null. If called with a null pointer, this will cause a Load Access Fault on ESP32-C3.

**Fix Applied:** Added `debug_assert!` to catch null pointer in debug builds.

---

### BUG-002: WaitQueue::delete lacks null check

**File:** `esp-rtos/src/esp_radio/mod.rs:180`

**Status:** FIXED

**Severity:** High

**Description:**
The `WaitQueueImplementation::delete` function dereferences the queue pointer without checking if it's null. If called with a null pointer, this will cause a Load Access Fault on ESP32-C3.

**Fix Applied:** Added early return if queue pointer is null.

---

### BUG-003: __pender function lacks null check for context

**File:** `esp-rtos/src/embassy/mod.rs:154`

**Status:** FIXED

**Severity:** Medium

**Description:**
The `__pender` function handles various interrupt contexts. The catch-all case (`_`) attempts to cast and dereference the context pointer. While the current implementation uses `unwrap!` which would panic rather than cause a Load Access Fault, an explicit null check would be clearer and more defensive.

**Fix Applied:** Added early return if context is null.

---

### BUG-004: _getreent assumes current_task is always valid

**File:** `esp-rtos/src/syscall.rs:14`

**Status:** FIXED (documentation)

**Severity:** Low (panics instead of causing fault)

**Description:**
The `_getreent` function calls `current_task()` and immediately dereferences it without any null checks. While `current_task()` itself uses `unwrap!` which would panic rather than cause a Load Access Fault, this assumption relies on the function never being being called before scheduler initialization.

**Fix Applied:** Updated comment to correctly document the pre-condition that the scheduler must be initialized before calling allocation functions.

---

## Related Fixes

The following issues were fixed in commit `ebcb9597`:

- `current_task_thread_semaphore()`: Added null check, returns dangling semaphore if task is null
- `task_priority()`: Added null check, returns Priority::ZERO if task is null
- `set_task_priority()`: Added null check, returns early if task is null

Debug assertions were added in commit `14ac9c7f` to catch these issues during development.

---

## RISC-V Load Access Fault Background

On ESP32-C3 (RISC-V architecture), accessing a null or invalid memory address causes a Load Access Fault exception. This is different from the typical "null pointer dereference" behavior on other architectures where dereferencing null typically just reads/writes to address 0x0.

The root cause of many of these bugs is that `current_task()` reads from the thread pointer (tp) register, which may be null or uninitialized before the RTOS scheduler is fully started. Functions that depend on `current_task()` or operate on task pointers must check for null before dereferencing.
