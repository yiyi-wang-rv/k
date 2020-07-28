---
copyright: Copyright (c) 2019-2020 K Team. All Rights Reserved.
---

K Foreign Function Interface
============================

The K Foreign Function Interface (FFI) module provides a way to call native
functions directly from a K semantics using the C ABI. It also provides
utilities for allocating and deallocating byte buffers with static addresses
that are suitable for being passed to native code.

It is built off of the underlying libffi library
(https://sourceware.org/libffi/) and is subject to some of the same
limitations as that library. Bear in mind, because this library exposes
a number of unsafe C APIs directly, misuse of the library is likely to lead
to memory corruption in your interpreter and can cause segmentation faults or
corrupted term representations that lead to undefined behavior at runtime.

```k
require "domains.k"

module FFI-SYNTAX
  imports LIST
```

The FFIType sort is used to declare the native C ABI types of operands passed
to the `#ffiCall` function. These types roughly correspond to the types 
declared in `ffi.h` by libffi.

```k
  syntax FFIType ::= "#void" [klabel(#void), symbol]
                  | "#uint8" [klabel(#uint8), symbol]
                  | "#sint8" [klabel(#sint8), symbol]
                  | "#uint16" [klabel(#uint16), symbol]
                  | "#sint16" [klabel(#sint16), symbol]
                  | "#uint32" [klabel(#uint32), symbol]
                  | "#sint32" [klabel(#sint32), symbol]
                  | "#uint64" [klabel(#uint64), symbol]
                  | "#sint64" [klabel(#sint64), symbol]
                  | "#float" [klabel(#float), symbol]
                  | "#double" [klabel(#double), symbol]
                  | "#uchar" [klabel(#uchar), symbol]
                  | "#schar" [klabel(#schar), symbol]
                  | "#ushort" [klabel(#ushort), symbol]
                  | "#sshort" [klabel(#sshort), symbol]
                  | "#uint" [klabel(#uint), symbol]
                  | "#sint" [klabel(#sint), symbol]
                  | "#ulong" [klabel(#ulong), symbol]
                  | "#slong" [klabel(#slong), symbol]
                  | "#longdouble" [klabel(#longdouble), symbol]
                  | "#struct" "(" List ")" [klabel(#struct), symbol]
endmodule

module FFI
  imports FFI-SYNTAX
  imports BYTES
  imports STRING

```

FFI Calls
---------

The `#ffiCall` functions are designed to call a native C ABI function and 
return a native result. They come in three variants:

### Non-variadic

In the first variant, `#ffiCall(Address, Args, ArgTypes, ReturnType)` takes
an integer address of a function (which can be obtained from
`#functionAddress`), a `List` of `Bytes` containing the arguments of the
function, a `List` of `FFIType`s containing the types of the parameters of the
function, and an `FFIType` containing the return type of the function, and 
returns the return value of the function as a `Bytes`.

```k
  syntax Bytes ::= "#ffiCall" "(" Int "," List "," List "," FFIType ")" [function, hook(FFI.call)]
```

### Variadic

In the second variant,
`#ffiCall(Address, Args, FixedTypes, VariadicTypes, ReturnType` takes an
integer address of a function, a `List` of `Bytes` containing the arguments
of the call, a `List` of `FFIType`s containing the types of the fixed
parameters of the function, a `List` of `FFIType`s containing the types of the
variadic parameters of the function, and an `FFIType` containing the return
type of the function, and returns the return value of the function as a
`Bytes`.

```k
  syntax Bytes ::= "#ffiCall" "(" Int "," List "," List "," List "," FFIType ")" [function, hook(FFI.call_variadic)]
```

### Generic

In the third variant,
`#ffiCall(IsVariadic, Address, Args, ArgTypes, NFixed, ReturnType` takes
a boolean indicating whether the function is variadic or not, an integer
address of a function, a `List` of `Bytes` containing the arguments of the
call, a `List` of `FFIType`s containing the parameter typess of the call
followed by the types of the variadic arguments of the call, if any, an `Int`
containing how many of the arguments of the call are fixed or not, and an
`FFIType` containing the return type of the function, and returns the return
value of the function as a `Bytes`.

```k
  syntax Bytes ::= "#ffiCall" "(" Bool "," Int "," List "," List "," Int "," FFIType ")" [function]

  rule #ffiCall(false, Addr::Int, Args::List, Types::List, _, Ret::FFIType) => #ffiCall(Addr, Args, Types, Ret)
  rule #ffiCall(true, Addr::Int, Args::List, Types::List, NFixed::Int, Ret::FFIType) => #ffiCall(Addr, Args, range(Types, 0, size(Types) -Int NFixed), range(Types, NFixed, 0), Ret)
```

Symbol Lookup
-------------

The FFI module provides a mechanism to look up any function symbol and return
that function's address.

```k
  syntax Int ::= "#functionAddress" "(" String ")" [function, hook(FFI.address)]
```

Direct Memory Management
------------------------

Most memory used by the LLVM backend to represent terms is managed
automatically via garbage collection. However, a consequence of this is that
a particular term does not have a fixed address across its entire lifetime
in most cases. Sometimes this is undesirable, especially if you intend for
the address of the memory to be taken by the semantics or if you intend
to pass this memory directly to native code. As a result, the FFI module
exposes the following unsafe APIs for memory management. Note that use of 
these APIs leaves the burden of memory management completely on the user,
and thus misuse of these functions can lead to things like use-after-free 
and other memory corruption bugs.

### Allocation

`#alloc(Key, Size, Align)` will allocate `Size` bytes with an alignment
requirement of `Align` (which must be a power of two), and return it as a 
`Bytes` term. The memory is uniquely identified by its key and that key will
be used later to free the memory. The memory is not implicitly freed by garbage
collection; failure to call `#free` on the memory at a later date can lead to
memory leaks.

```k
  syntax Bytes ::= "#alloc" "(" KItem "," Int "," Int ")" [function, hook(FFI.alloc)]
```

### Addressing

`#addess(B)` will return an `Int` representing the address of the first byte of
B, which must be a `Bytes`. Unless the `Bytes` term was allocated by `#alloc`,
the return value is unspecified and may not be the same across multipl
invocations on the same byte buffer. However, it is guaranteed that memory
allocated by `#alloc` will have the same address throughout its lifetime.

```k
  syntax Int ::= "#address" "(" Bytes ")" [function, hook(FFI.bytes_address)]
```

### Deallocation

`#free(Key)` will free the memory of the `Bytes` object that was allocated
by a previous call to `#alloc`. If `Key` was not used in a previous call to
`#alloc`, or the memory was already freed, no action is taken. It will generate
undefined behavior if the `Bytes` term returned by the previous call to
`#alloc` is still referenced by any other term in the configuration or a
currently evaluating rule. The function returns `.K`.

```k
  syntax K ::= "#free" "(" KItem ")" [function, hook(FFI.free)]
endmodule
```