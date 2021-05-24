- Feature Name: `symbol_name`
- Start Date: 2021-05-24
- RFC PR: [fimoengine/emf-rfcs#0000](https://github.com/fimoengine/emf-rfcs/pull/0000)

# Summary

[summary]: #summary

Introduces the `emf_cbase_symbol_name_t` type, which represents symbol names.

# Motivation

[motivation]: #motivation

[RFC 14](https://github.com/fimoengine/emf-rfcs/pull/0014) motivated the usefulness of sized string types. A hole that
was overlooked by the RFC was symbol strings, which are used by the library api.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The type is defined as follows:

```c
typedef struct emf_cbase_symbol_name_t {
  const char* data;
  size_t length;
} emf_cbase_symbol_name_t;
```

Example usage:

```c
emf_cbase_symbol_name_t empty = { .data = NULL, .length = 0 };
emf_cbase_symbol_name_t symbol_name = { .data = "symbol", .length = strlen("symbol") };
```

Functions which required a symbol name (formerly of the type `const char*`) will now require
an `emf_cbase_symbol_name_t` instance.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The new type, as defined above, is added to the `emf_cbase_library.h` header.

The following function pointer types are modified to use the newly added type:

- emf_cbase_library_get_data_symbol_fn_t
- emf_cbase_library_get_function_symbol_fn_t
- emf_cbase_library_loader_interface_get_data_symbol_fn_t
- emf_cbase_library_loader_interface_get_function_symbol_fn_t

# Drawbacks

[drawbacks]: #drawbacks

The length of the string must still be computed, this time in advance.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

See "Motivation" for the rationale.  
The status quo is an alternative.

# Prior art

[prior-art]: #prior-art

This RFC is a follow up to https://github.com/fimoengine/emf-rfcs/pull/0014.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

None so far.

# Future possibilities

[future-possibilities]: #future-possibilities

Allow platform-dependent types.
