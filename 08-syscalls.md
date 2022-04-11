# Syscall ABI

The syscall "ABI" defines how arguments are passed from actors to the FVM, and how values are
returned from the FVM to actors.

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

- Pointers to syscall-safe types are passed as 32bit offsets into the Wasm module's memory.
- Syscall-safe types are returned from the FVM to the Wasm module by writing to the Wasm module's
  memory at a location specified by an "out pointer" parameter.
- Signed integers are passed as two's-complement.

## Conventions

Conventionally:

- Byte buffers & strings are passed as pointer/length pairs.
- CIDs are passed as pointer only.
- Token amounts are passed as two `u64` values representing the "high" and "low" bits of a `u128`
  attoFIL value.
- If a syscall returns a non-zero-sized value (other than the error number), the syscall takes an
  out-pointer as the first parameter.

By example, given the function:

```rust
#[derive(Copy, Clone)]
struct ComplexValue {
    foo: u64,
    bar: u16,
}
fn compute_thing(k: Cid, data: &[u8]) -> Result<ComplexValue, ErrorNumber>;
```

We'd define it as a syscall a follows:

```rust
#[derive(Copy, Clone)]
#[repr(packed, C)]
struct ComplexValue {
    foo: u64,
    bar: u16,
}
extern "C" fn compute_thing(result_ptr: *mut ComplexValue, k_offset: *const u8, data_offset: *const u8, data_length: u32) -> ErrorNumber;
```

And would have a Wasm signature of:

```wat
(func $compute_thing (param $result_ptr i32) (param $k_offset i32) (param $data_offset i32) (param $data_length u32) (result i32))
```

However, a syscall that does not return a value:

```rust
fn do_thing(data: &[u8]) -> Result<(), ErrorNumber>;
```

Would be defined as:

```rust
extern "C" fn do_thing(data_offset: *const u8, data_length: u32) -> ErrorNumber;
```

With _no_ out-pointer for the return value.
