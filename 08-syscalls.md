# Syscall ABI

The syscall "ABI" defines how arguments are passed from actors to the FVM, and how values are
returned from the FVM to actors.

The actual syscall API can be found in the [SDK
documentation](https://docs.rs/fvm_sdk/latest/fvm_sdk/sys/index.html).

## Syscall Safe Types

A "syscall-safe" is a type that can safely be passed across the syscall ABI boundary. A syscall-safe
type must have the following property:

> A type of size-S (bits) is "syscall-safe" if every size-S bit-string is a universally unique,
> valid, and distinct instance of this type.

- Universally Unique: The value cannot have different interpretations in different contexts (e.g., no pointers).
- Valid: There can be no "undefined" values (potentially leading to undefined behavior).
- Distinct: There can be no instance with more than one valid representation. Otherwise, the choice
  of representation could lead to non-determinism on-chain.

Concretely, we limit "syscall-safe" types to:

1. Primitives: 1, 2, 4, or 8-byte signed (two's complement) or unsigned little-endian integer.
2. Packed (no padding), ordered records composed of primitives and other records. Fields are packed
   in the order in which they appear in the syscall definition.

## Arguments & Return Values

Each syscall takes zero or more i32 and i64 arguments and returns exactly one i32 indicating the
final status of the syscall (the "error number").

- Signed integers are passed as two's-complement.
- An actor may pass syscall-safe values to the FVM by passing a pointer (as a 32bit offset) to the
  value in Wasm memory.
- An actor may receive "returned" values from the FVM by passing a pointer (as a 32bit offset) to a
  location in Wasm memory where the value should be written.

## Conventions

Conventionally:

- Byte buffers & strings are passed as pointer/length pairs.
- CIDs are passed as pointers to serialized CID buffers (with no explicit length).
    - The FVM enforces a maximum digest size of 64 bytes (512bits).
    - A 100 byte buffer is guaranteed to be large enough to hold any CID "returned" by a syscall.
- In parameters, token amounts are passed as two `u64` values representing the "high" and "low" bits
  of a `u128` attoFIL value, in that order.
- If a syscall returns a fixed, non-empty value (other than the error number), the syscall takes an
  out-pointer as the first parameter.
    - On call, the FVM will check that the return pointer is "valid". If it doesn't fall within the
      actor's memory, or there isn't enough memory at the specified offset to fit the return value,
      the syscall will return an `IllegalArgument` error without side effect.
    - On return, the syscall will atomically write the return value to this pointer if, and only
      if, the syscall succeeds.
- Some syscalls return additional values through additional explicit out-pointers.
- Some syscalls return variable-length values through a combination of an out-pointer and a length.

### Example 1

For example, given the function:

```rust
#[derive(Copy, Clone)]
struct ComplexValue {
    foo: u64,
    bar: u16,
}
fn compute_thing(k: Cid, data: &[u8]) -> Result<ComplexValue, ErrorNumber>;
```

Would be defined as the syscall:

```rust
#[derive(Copy, Clone)]
#[repr(packed, C)]
struct ComplexValue {
    foo: u64,
    bar: u16,
}
extern "C" fn compute_thing(result_ptr: *mut ComplexValue, k_offset: *const u8, data_offset: *const u8, data_length: u32) -> ErrorNumber;
```

With a Wasm signature of:

```wat
(func $compute_thing (param $result_ptr i32) (param $k_offset i32) (param $data_offset i32) (param $data_length u32) (result i32))
```

### Example 2

A syscall that does not return a value, such as the following:

```rust
fn do_thing(data: &[u8]) -> Result<(), ErrorNumber>;
```

Would be defined as:

```rust
extern "C" fn do_thing(data_offset: *const u8, data_length: u32) -> ErrorNumber;
```

With _no_ out-pointer for the return value.

It would have a Wasm signature of:

```wat
(func $do_thing (param $data_offset i32) (param $data_length u32) (result i32))
```
