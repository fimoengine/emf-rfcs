- Feature Name: `fat_ptrs`
- Start Date: 2021-05-22
- RFC PR: [fimoengine/emf-rfcs#0000](https://github.com/fimoengine/emf-rfcs/pull/0000)

# Summary

[summary]: #summary

Introduces a uniform representation for data + vtable pointers to objects, as fat pointers.

# Motivation

[motivation]: #motivation

Objects are used for encapsulating data, and the functions that act on that data. Many programming languages allow the
creation of objects, for example through classes in C++ or traits in Rust. There are multiple approaches for the
implementation such of objects. The first one combines the data, and the functions inside the object itself, resulting
in thin pointers, which are simple pointers to the object. Another approach is separating the data and the function
vtable, and having an object point to both. Both approaches are used by the `emf-core-base` interface, with types like
`emf_cbase_sync_handler_interface_t` which is defined as a thin pointer, or `emf_cbase_error_t` which is a fat pointer.
Having a uniform model for objects would simplify the interface, and allow for easier modifications in the future.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Consider the following object definition:

```c
struct my_object {
  void* object_data;
  void (*func_1)(void*);
  void (*func_2)(void*);
  void (*func_3)(void*);
};
```

Assuming that we may want to extend the object in the future, e.g., by adding more functions, it is unsafe to store the
object by value, instead we must store a thin pointer to an instance:

```c
// Construct the object.
struct my_object* object = ...;

// Use it afterwards.
object->func_1(object->object_data);
```

The same object can be represented as a fat pointer:

```c
struct my_object_vtable {
  void (*func_1)(void*);
  void (*func_2)(void*);
  void (*func_3)(void*);
};

struct my_object {
  void* data;
  const struct my_object_vtable* vtable;
};
```

Because the vtable is stored separately, it is safe to store the object by value, without knowing the exact layout (and
contents) of the vtable structure:

```c
// Construct the object.
struct my_object object = ...;

// Use it afterwards.
object.vtable->func_1(object.data);
```

Both approaches incur some pointer indirections, the thin pointer approach must dereference the object twice to gain
access to the data and functions, but could be optimized away by the compiler. The fat pointer approach only
dereferences the vtable once, but incurs the const of having to store two pointers instead of one.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## Changes to the interface

Any occurrence of thin pointer objects will have to be replaced, like shown in the example above. The types that need to
be changed are:

- `emf_cbase_interface_t`
- `emf_cbase_sync_handler_interface_t`
- `emf_cbase_library_loader_interface_t`
- `emf_cbase_module_loader_interface_t`

Functions that accepted (or returned) thin pointers will be modified to use the newly created fat pointers. Furthermore,
the types `emf_cbase_native_library_loader_interface_t` and `emf_cbase_native_module_loader_interface_t` will be renamed
to `emf_cbase_native_library_loader_vtable_t` and `emf_cbase_native_module_loader_vtable_t`, to better express the use
of those types. The function pointer `get_internal_loader_fn`, inside `emf_cbase_library_loader_interface_t`
and `emf_cbase_module_loader_interface_t`, will be renamed to `get_extended_vtable_fn`.

# Drawbacks

[drawbacks]: #drawbacks

One drawback of the proposed approach is, that a pointer to an object takes up twice the space a thin pointer would
take. Depending on the number of objects, this may have a significant impact on the execution of the code.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

## Thin pointers

An alternative is to use thin pointers to objects. While usable, this has the disadvantage when dealing with
polymorphism. Because the data of the object is combined with the vtable, a polymorphic object will inevitably have to
resort to storing a thin pointer to the base class, resulting to multiple indirections. Fat pointers only necessitate
swapping the pointer to the vtable with another vtable.

# Prior art

[prior-art]: #prior-art

This RFC is heavily inspired by how Rust implements trait objects. Traits define shared behaviour of objects. Trait
objects are fat pointers to an object which implements a given trait. Similarly to the approach in this RFC, a reference
of the trait object `&dyn SomeTrait` is as follows:

```rust
struct TraitObject {
    pub data: *mut (),
    pub vtable: *mut (),
}
```

The approach was modified to allow for constant data pointers.
