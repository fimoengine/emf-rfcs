- Feature Name: `better_discriminants`
- Start Date: 2021-05-17
- RFC PR: [fimoengine/emf-rfcs#0011](https://github.com/fimoengine/emf-rfcs/pull/0011)

# Summary

[summary]: #summary

Swaps the order of the discriminant and the union, for the specified option and result types. 
This allows for an easier implementation in languages which natively support tagged unions like Rust.

# Motivation

[motivation]: #motivation

Algebraic data types like sum types are very useful. The `emf-core-base` interface specifies optional and result types.
Unfortunately, the layout of those types is incompatible with languages like Rust, which natively support tagged unions.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

## Rust enum

Consider the following Rust enum:

```rust
#[repr(C, u8)]
enum OptionalI32 {
    None,
    Some(i32)
}
```

Rust allows us to specify the layout of the union, and the data type of the discriminant.
The enum is equivalent to the following C type:

```c
union optional_i32_payload {
  int32_t some;
};

struct optional_i32 {
  uint8_t tag;
  optional_i32_payload payload;
};
```

## New layout

The optional and result types are currently specifies as follows:

```c
struct optional_T {
  union {
    T value;
    int8_t _dummy;
  };
  emf_cbase_bool_t has_value;
};

struct result_R_E {
  union {
    R result;
    E error;
  };
  emf_cbase_bool_t has_error;
};
```

Simple changes to the union and the discriminant makes it compatible with the Rust enums:

```c
struct optional_T {
  int8_t tag;
  union {
    int8_t _none;
    T some;
  };
};

struct result_R_E {
  int8_t tag;
  union {
    R ok;
    E err;
  };
};
```

While not strictly necessary, the `_none` member allows for easier initialisation
in languages with non-trivial constructors.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## Required work

Since all optional and result types are defined via macros, the required work should be minimal.

# Drawbacks

[drawbacks]: #drawbacks

This is a breaking change to the interface api.
