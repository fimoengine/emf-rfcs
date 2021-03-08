- Feature Name: `unwind_internal`
- Start Date: 2021-03-06
- RFC PR: [fimoengine/emf-rfcs#0007](https://github.com/fimoengine/emf-rfcs/pull/0007)

# Summary

[summary]: #summary

This RFC proposes an extension to be added to the `emf-core-base` interface.
This extension allows for more control of the unwinding functionality specified by the base interface.

# Motivation

[motivation]: #motivation

The `emf-core-base` interface already provides some sort of unwinding functionality
with its `shutdown` and `panic` functions. The implementation of those functions is not specified.
This causes some problems when the interface is not in full control of the process, which
may be the case if it was loaded as a module. In such cases, it is conceivable that a user
may want to catch a shutdown or panic and continue outside of the interface.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

This extension specifies a new Interface, which controls the unwinding of the `emf-core-base` interface.
This interface can be accessed with the `get_unwind_internal_interface()` function.
It contains functions for accessing and modifying the active unwind context and functions.

The following example shows how one might use the extension:

```c
/// Fetch the interface
emf_cbase_t* base_module = ...;
emf_cbase_ext_unw_int_interface_t* interface = ...;

/// Save state
emf_cbase_ext_unw_int_context_t* prev_cont = interface->get_context_fn(base_module);
emf_cbase_ext_unw_int_shutdown_fn_t prev_shutdown = interface->get_shutdown_fn_fn(base_module);
emf_cbase_ext_unw_int_panic_fn_t prev_panic = interface->get_panic_fn_fn(base_module);

/// Create a context
emf_cbase_ext_unw_int_context_t* context = ...;
emf_cbase_ext_unw_int_shutdown_fn_t shutdown_fn = ...;
emf_cbase_ext_unw_int_panic_fn_t panic_fn = ...;

/// Set as active
interface->set_context_fn(base_module, context);
interface->set_shutdown_fn_fn(base_module, shutdown_fn);
interface->set_panic_fn_fn(base_module, panic_fn);

/// Use
...

/// Restore
interface->set_context_fn(base_module, prev_cont);
interface->set_shutdown_fn_fn(base_module, prev_shutdown);
interface->set_panic_fn_fn(base_module, prev_panic);
```

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

This extension is mandatory in case `emf-core-base` is implemented
as a standalone module. This extension adds a new header called `emf_unwind_internal.h`
and a new function to the `emf-core-base` interface. The unwind operation is controlled
by a context which encompasses a context pointer a panic handler and a shutdown handler.
Each thread maintains its own context.

## Api overview

```c
void set_context(emf_cbase_t* base_module, emf_cbase_ext_unw_int_context_t* context);
emf_cbase_ext_unw_int_context_t* get_context(emf_cbase_t* base_module);

void set_shutdown_fn(emf_cbase_t* base_module, emf_cbase_ext_unw_int_shutdown_fn_t shutdown_fn);
emf_cbase_ext_unw_int_shutdown_fn_t get_shutdown_fn(emf_cbase_t* base_module);

void set_panic_fn(emf_cbase_t* base_module, emf_cbase_ext_unw_int_panic_fn_t panic_fn);
emf_cbase_ext_unw_int_panic_fn_t get_panic_fn(emf_cbase_t* base_module);

const emf_cbase_ext_unw_int_interface_t* get_unw_int_interface(emf_cbase_t* base_module)
```

## Functions

### set_context

> ```c
> void set_context(emf_cbase_t* base_module, emf_cbase_ext_unw_int_context_t* context);
> ```
>
> - Effects: Sets a new context.
> - Remarks: Passing `NULL` will select the default context.

---

### get_context

> ```c
> emf_cbase_ext_unw_int_context_t* get_context(emf_cbase_t* base_module);
> ```
>
> - Effects: Fetches the active context.
> - Returns: Active context.

---

### set_shutdown_fn

> ```c
> void set_shutdown_fn(emf_cbase_t* base_module, emf_cbase_ext_unw_int_shutdown_fn_t shutdown_fn);
> ```
>
> - Effects: Sets a new shutdown function.
> - Remarks: Passing `NULL` will select the default function.

---

### get_shutdown_fn

> ```c
> emf_cbase_ext_unw_int_shutdown_fn_t get_shutdown_fn(emf_cbase_t* base_module);
> ```
>
> - Effects: Fetches the active shutdown function.
> - Returns: Active shutdown function.

---

### set_panic_fn

> ```c
> void set_panic_fn(emf_cbase_t* base_module, emf_cbase_ext_unw_int_panic_fn_t panic_fn);
> ```
>
> - Effects: Sets a new panic function.
> - Remarks: Passing `NULL` will select the default function.

---

### get_panic_fn

> ```c
> emf_cbase_ext_unw_int_panic_fn_t get_panic_fn(emf_cbase_t* base_module);
> ```
>
> - Effects: Fetches the active panic function.
> - Returns: Active panic function.

---

### get_unw_int_interface

> ```c
> const emf_cbase_ext_unw_int_interface_t* get_unw_int_interface(emf_cbase_t* base_module)
> ```
>
> - Effects: Fetches the interface of the extension.
> - Returns: Interface.

## Fn id

The `get_unw_int_interface()` function can be accessed by using the id
`emf_cbase_fn_ptr_id_ext_get_unwind_internal_interface`, which is defined as:

```c
typedef enum emf_cbase_fn_ptr_id_t : int32_t {
    ...
    emf_cbase_fn_ptr_id_ext_get_unwind_internal_interface = 51,
    ...
} emf_cbase_fn_ptr_id_t;
```

## Types

### emf_cbase_ext_unw_int_shutdown_fn_t

> ```c
> FN_T(emf_cbase_ext_unw_int_shutdown_fn_t, void, emf_cbase_ext_unw_int_context_t* context)
> ```

---

### emf_cbase_ext_unw_int_panic_fn_t

> ```c
> FN_T(emf_cbase_ext_unw_int_panic_fn_t, void, emf_cbase_ext_unw_int_context_t* context, const char* error)
> ```

---

### emf_cbase_ext_unw_int_set_context_fn_t

> ```c
> BASE_FN_T(emf_cbase_ext_unw_int_set_context_fn_t, void, emf_cbase_ext_unw_int_context_t* context)
> ```

---

### emf_cbase_ext_unw_int_get_context_fn_t

> ```c
> BASE_FN_T(emf_cbase_ext_unw_int_get_context_fn_t, emf_cbase_ext_unw_int_context_t*, void)
> ```

---

### emf_cbase_ext_unw_int_set_shutdown_fn_fn_t

> ```c
> BASE_FN_T(emf_cbase_ext_unw_int_set_shutdown_fn_fn_t, void, emf_cbase_ext_unw_int_shutdown_fn_t shutdown_fn)
> ```

---

### emf_cbase_ext_unw_int_get_shutdown_fn_fn_t

> ```c
> BASE_FN_T(emf_cbase_ext_unw_int_get_shutdown_fn_fn_t, emf_cbase_ext_unw_int_shutdown_fn_t, void)
> ```

---

### emf_cbase_ext_unw_int_set_panic_fn_fn_t

> ```c
> BASE_FN_T(emf_cbase_ext_unw_int_set_panic_fn_fn_t, void, emf_cbase_ext_unw_int_panic_fn_t panic_fn)
> ```

---

### emf_cbase_ext_unw_int_get_panic_fn_fn_t

> ```c
> BASE_FN_T(emf_cbase_ext_unw_int_get_panic_fn_fn_t, emf_cbase_ext_unw_int_panic_fn_t, void)
> ```

---

### emf_cbase_ext_unw_int_get_unw_int_interface

> ```c
> BASE_FN_T(emf_cbase_ext_unw_int_get_panic_fn_fn_t, const emf_cbase_ext_unw_int_interface_t*, void)
> ```

---


### emf_cbase_ext_unw_int_interface_t

> ```c
> typedef struct emf_cbase_ext_unw_int_interface_t {
>     emf_cbase_ext_unw_int_set_context_fn_t set_context_fn;
>     emf_cbase_ext_unw_int_get_context_fn_t get_context_fn;
>     emf_cbase_ext_unw_int_set_shutdown_fn_fn_t set_shutdown_fn_fn;
>     emf_cbase_ext_unw_int_get_shutdown_fn_fn_t get_shutdown_fn_fn;
>     emf_cbase_ext_unw_int_set_panic_fn_fn_t set_panic_fn_fn;
>     emf_cbase_ext_unw_int_get_panic_fn_fn_t get_panic_fn_fn;
> } emf_cbase_ext_unw_int_interface_t;
> ```

# Drawbacks

[drawbacks]: #drawbacks

The proposed unwinding mechanism is unpractical for languages which call destructors when a
stack frame is cleaned up. Unwinding across ffi boundaries remains unspecified.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Like standard practice in many languages, calling a panic or shutdown should only occur in
exceptional circumstances. This design allows to handle those rare occurrences by propagating
those calls like in the following example.

```c
// Start in C and setup the first context.
// Uses setjmp and longjmp.
typedef struct context_t {
    jump_buf env;
    const char* error;
} context_t;

_Noreturn void shutdown_handler(emf_cbase_ext_unw_int_context_t* context) {
    context_t* cont = (context_t*)context;
    // Use 0 for shutdown.
    longjmp(cont->env, 0);
}

_Noreturn void panic_handler(emf_cbase_ext_unw_int_context_t* context, const char* error) {
    context_t* cont = (context_t*)context;
    cont->error = error;
    // Use 1 for panic.
    longjmp(cont->env, 1);
}

int main() {
    // Initialize the interface.
    emf_cbase_t* base_module = ...;
    emf_cbase_ext_unw_int_interface_t* unw_interface = ...;
    ...

    // Setup unwinding.
    volatile int panic = 0;
    volatile context_t context;
    memset(&context, 0, sizeof(context_t));

    switch(setjmp(context.env)) {
    case 0:
        // Set the context. We want to use our own unwinding 
        // implementation so there is no need to save the default context.
        unw_interface->set_context_fn(base_module, &context);
        unw_interface->set_shutdown_fn_fn(base_module, shutdown_handler);
        unw_interface->set_panic_fn_fn(base_module, panic_handler);

        // Use the interface as normal.
        ...
        break;
    case 1:
        // Shutdown, no need for special treatment, as we are already
        // cleaning up.
        panic = 0;
        break;
    case 2:
        // Panic
        panic = 1;
        break;
    }

    // Reset context.
    unw_interface->set_context_fn(base_module, NULL);
    unw_interface->set_shutdown_fn_fn(base_module, NULL);
    unw_interface->set_panic_fn_fn(base_module, NULL);

    // Handle possible error.
    if (panic == 1) {
        if (context.error) {
            printf(context.error);
        }
        abort();
    }

    // Cleanup.
    ...
    return 0;
}
```

Another context must be setup, when crossing into another language:

```cpp
// Example of setting up another context in C++.
struct UnwindException {
    bool panic;
    const char* error;
};

void called_by_interface() {
    // Fetch the interface.
    emf_cbase_t* base_module = ...;
    emf_cbase_interface_t* base_interface = ...;
    emf_cbase_ext_unw_int_interface_t* unw_interface = ...;

    // Save the previous context.
    auto prev_context = unw_interface->get_context_fn(base_module);
    auto prev_shutdown_fn = unw_interface->get_shutdown_fn_fn(base_module);
    auto prev_panic_fn = unw_interface->get_panic_fn_fn(base_module);

    // Set new context.
    bool context_dummy;
    auto shutdown_fn = [] () { throw UnwindException { false, nullptr }; };
    auto panic_fn = [] (const char* error) { throw UnwindException { true, error }; };

    unw_interface->set_context_fn(base_module, &context_dummy);
    unw_interface->set_shutdown_fn_fn(base_module, shutdown_fn);
    unw_interface->set_panic_fn_fn(base_module, panic_fn);

    try {
        // Continue as normal.
        ...
    } catch (const UnwindException& e) {
        // Restore context and propagate.
        unw_interface->set_context_fn(base_module, prev_context);
        unw_interface->set_shutdown_fn_fn(base_module, prev_shutdown_fn);
        unw_interface->set_panic_fn_fn(base_module, prev_panic_fn);

        if (e.panic) {
            // Propagate panic.
            base_interface->sys_panic_fn(base_module, e.error);
        } else {
            // Propagate shutdown.
            base_interface->sys_shutdown_fn(base_module);
        }
    }

    // Restore context.
    unw_interface->set_context_fn(base_module, prev_context);
    unw_interface->set_shutdown_fn_fn(base_module, prev_shutdown_fn);
    unw_interface->set_panic_fn_fn(base_module, prev_panic_fn);
}
```

# Prior art

[prior-art]: #prior-art

# Unresolved questions

[unresolved-questions]: #unresolved-questions

# Future possibilities

[future-possibilities]: #future-possibilities

In the future we may decide to incorporate this extension into the main interface.
