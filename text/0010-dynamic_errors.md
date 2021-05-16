- Feature Name: `dynamic_errors`
- Start Date: 2021-05-16
- RFC PR: [fimoengine/emf-rfcs#0010](https://github.com/fimoengine/emf-rfcs/pull/0010)

# Summary

[summary]: #summary

Introduce the new error type `emf_cbase_error_t` to the `emf-core-base` interface.
This new type can be used to model arbitrary error values.

# Motivation

[motivation]: #motivation

The interface uses error enums to model error values.
While easy to implement, this poses a few disadvantages:

1. New errors can only be added through RFCs.
2. Errors can not be chained.
3. Errors have no textual description.
4. It is difficult to specify every possible error value.

This RCS proposes the introduction of a single error type `emf_cbase_error_t`.
This type will be able to model arbitrary error values and corrects the shortcoming listed above.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

## The new error api

The new error api consists of three main components:

1. An owned and an unowned error type, called `emf_cbase_error_t` and `emf_cbase_error_ref_t` respectively.
   Those types represent arbitrary error values and contain information relevant to the programmer.
2. A formatted error value of the type `emf_cbase_error_info_t`. This value contains
   the textual representation of the generated error.
3. An `UTF-8` encoded error string of the type `emf_cbase_error_string_t`.

## Conceptual api

The following Rust code show the conceptual API:

```rust
pub struct Error {
    // ... 
}

impl Error {
    /// The lowe-level cause of this error, if it exists.
    pub fn source(&self) -> Option<&Error> {
        // ...
    }

    /// Display error info.
    pub fn display(&self) -> ErrorInfo {
        // ...
    }

    /// Debug error info.
    pub fn debug(&self) -> ErrorInfo {
        // ...
    }
}

impl Drop for Error {
    // ...
}

impl Debug for Error {
    // ...
}

impl Display for Error {
    // ...
}

pub struct ErrorInfo {
    // ...
}

impl ErrorInfo {
    /// Textual representation of the error.
    pub fn as_str(&self) -> &str {
        // ...
    }
}

impl Drop for ErrorInfo {
    // ...
}

impl Clone for ErrorInfo {
    /// ...
}

impl Debug for ErrorInfo {
    // ...
}

impl Display for ErrorInfo {
    // ...
}
```

## C implementation

The error types are implemented as fat-pointers, which contain a pointer to the data and a pointer to the vtable.
The vtable-pointer is a static non `NULL` value, while the data pointer may be dynamically allocated and nullable.

### Error values

The types `emf_cbase_error_t` and `emf_cbase_error_ref_t` are fat-pointers, where the data-pointer is of the type `emf_cbase_error_data_t` and the vtable-pointer has the type `emf_cbase_error_vtable_t`:

```c
typedef struct emf_cbase_error_data_t emf_cbase_error_data_t;
typedef struct emf_cbase_error_vtable_t emf_cbase_error_vtable_t;

/// Owned error value.
typedef struct emf_cbase_error_t {
    emf_cbase_error_data_t* data;
    const emf_cbase_error_vtable_t* vtable;
} emf_cbase_error_t;

/// Unowned error value.
typedef struct emf_cbase_error_ref_t {
    const emf_cbase_error_data_t* data;
    const emf_cbase_error_vtable_t* vtable;
} emf_cbase_error_ref_t;

/// Optional error.
typedef struct emf_cbase_error_optional_t {
    union {
        emf_cbase_error_t value;
        int8_t _dummy;
    };
    emf_cbase_bool_t has_value;
} emf_cbase_error_optional_t;

/// Optional error.
typedef struct emf_cbase_error_ref_optional_t {
    union {
        emf_cbase_error_ref_t value;
        int8_t _dummy;
    };
    emf_cbase_bool_t has_value;
} emf_cbase_error_ref_optional_t;

/// Cleanup function.
typedef void(*emf_cbase_error_vtable_cleanup_fn)(emf_cbase_error_data_t* data);

/// Source fn.
typedef emf_cbase_error_ref_optional_t(*emf_cbase_error_vtable_source_fn)(const emf_cbase_error_data_t* data);

/// Display fn.
typedef emf_cbase_error_info_t(*emf_cbase_error_vtable_display_info_fn)(const emf_cbase_error_data_t* data);

/// Debug fn.
typedef emf_cbase_error_info_t(*emf_cbase_error_vtable_debug_info_fn)(const emf_cbase_error_data_t* data);

/// The error vtable.
typedef struct emf_cbase_error_vtable_t {
    emf_cbase_error_vtable_cleanup_fn cleanup_fn;
    emf_cbase_error_vtable_source_fn source_fn;
    emf_cbase_error_vtable_display_info_fn display_info_fn;
    emf_cbase_error_vtable_debug_info_fn debug_info_fn;
}
```

### Error info

The `emf_cbase_error_info_t` type is implemented similarly to the previous types:

```c
/// An unowned error string.
typedef struct emf_cbase_error_string_t {
    char* data;
    size_t length;
} emf_cbase_error_string_t;

typedef struct emf_cbase_error_info_data_t emf_cbase_error_info_data_t;

/// Cleanup function.
typedef void(*emf_cbase_error_info_vtable_cleanup_fn)(emf_cbase_error_info_data_t* data);

/// Clone function.
typedef emf_cbase_error_info_data_t*(*emf_cbase_error_info_vtable_clone_fn)(const emf_cbase_error_info_data_t* data);

/// As-str function.
typedef emf_cbase_error_string_t(*emf_cbase_error_info_vtable_as_str_fn)(const emf_cbase_error_info_data_t* data);

/// Error info vtable.
typedef struct emf_cbase_error_info_vtable_t {
    emf_cbase_error_info_vtable_cleanup_fn cleanup_fn;
    emf_cbase_error_info_vtable_clone_fn clone_fn;
    emf_cbase_error_info_vtable_as_str_fn as_str_fn;
} emf_cbase_error_info_vtable_t;

/// Error info.
typedef struct emf_cbase_error_info_t {
    emf_cbase_error_info_data_t* data;
    const emf_cbase_error_info_vtable_t* vtable;
} emf_cbase_error_info_t;
```

### Header file

The new types are defined in the `emf_cbase_error_t.h` header.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## Changes to the interface

Any definition that used to contain a custom error is modified, so that it uses the new `emf_cbase_error_t` type.
The error enums, such as `emf_cbase_library_error_t` are removed. Furthermore, the panic function of the sys api and of the `unwind_internal` extension will use the `emf_cbase_error_optional_t` type instead of a simple `char*` error:

```c
void (*emf_cbase_sys_panic_fn_t)(emf_cbase_t* base, emf_cbase_error_optional_t error);
void (*emf_cbase_ext_unw_int_panic_fn_t)(emf_cbase_ext_unw_int_context_t* context, emf_cbase_error_optional_t error);
```

# Drawbacks

[drawbacks]: #drawbacks

This is a breaking change to the interface api.
Another drawback is the increased difficulty when implementing a new error,
compared to a simple enum constant.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

## Flexibility and Stability

The most obvious advantages of this RFC are the improved API-stability of the interface,
and the, much improved, flexibility when implementing custom error values.

# Prior art

[prior-art]: #prior-art

This proposal is inspired by the `Error` trait of the Rust language.
