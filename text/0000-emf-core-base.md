- Feature Name: `emf_core_base_interface`
- Start Date: 2021-02-16
- RFC PR: [fimoengine/emf-rfcs#0000](https://github.com/fimoengine/emf-rfcs/pull/0000)

# Summary

[summary]: #summary

This RFC proposes a new interface to be added to the EMF project. This proposed interface will serve as a foundation for
all future interfaces.

# Motivation

[motivation]: #motivation

As software grows in complexity, it tends to become more tightly integrated and their subsystems more tightly coupled.
Such a tight coupling reduces flexibility and re-usability, and modifying it becomes a hassle. Because of that, many
developers are forced to constantly reinvent the wheel, which takes time and manpower. This becomes apparent, when we
look at the plethora of existing game engines.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

This RFC proposes an interface, which poses as a backbone for a loosely coupled game engine. This interface named **
emf-core-base** specifies the lifecycle and the basic capabilities of a conforming engine.

It consists of several apis:

- [Sys api](#sys-api)
- [Library api](#library-api)
- [Module api](#module-api)
- [Version api](#version-api)

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

The **emf-core-base** interface specifies a number of apis and conventions to be implemented by a conforming engine. The
interface constitutes a module in itself. To specify the interface, we will make use of the C language, as it serves as
a lingua franca for many other languages.

The relevant sections of the specification can be found in the following table:

## Table of contents

[table-of-contents]: #table-of-contents

- [Specification](#reference-level-explanation)
- [Table of contents](#table-of-contents)
- [Apis](#api)
  - [Sys api](#sys-api)
  - [Library api](#library-api)
  - [Module api](#module-api)
  - [Version api](#version-api)
- [Reference](#reference)
  - [Constants](#constants)
  - [Enums](#enums)
  - [Structs](#structs)
  - [Types](#types)
- [Headers](#headers)
- [Glossary](#glossary)

## Api

[api]: #api

Like already mentioned, the interface is composed of a collection of apis, which control one aspect of an engine. What
follows is the specification of each api. Unless otherwise stated, each function requires unique access to the
interface.

### Sys api

[sys-api]: #sys-api

The sys api controls the synchronization and termination of the engine.

#### Sys api overview

```c
// termination
_Noreturn void sys_shutdown();
_Noreturn void sys_panic(const char* error);

// feature request
emf_cbase_bool_t sys_has_function(emf_cbase_fn_ptr_id_t fn_id);
emf_cbase_sys_fn_optional_t sys_get_function(emf_cbase_fn_ptr_id_t fn_id);

// synchronisation
void sys_lock();
emf_cbase_bool_t sys_try_lock();
void sys_unlock();

// manual synchronisation
const emf_sync_handler_interface_t* sys_get_sync_handler();
void sys_set_sync_handler(const emf_sync_handler_interface_t* sync_handler);
```

#### Sys api functions

##### Sys api termination

> ```c
> _Noreturn void sys_shutdown();
> ```
>
> - Effects: Sends a termination signal.

---

> ```c
> _Noreturn void sys_panic(const char* error);
> ```
>
> - Effects: Execution of the program is stopped abruptly. The error may be logged.

##### Sys api feature request

> ```c
> emf_cbase_bool_t sys_has_function(emf_cbase_fn_ptr_id_t fn_id);
> ```
>
> - Effects: Checks if a function is implemented.
> - Returns: `emf_cbase_bool_true` if the function exists, `emf_cbase_bool_false` otherwise.

---

> ```c
> emf_cbase_sys_fn_optional_t sys_get_function(emf_cbase_fn_ptr_id_t fn_id);
> ```
>
> - Returns: Function pointer to the requested function.

##### Sys api synchronization

> ```c
> void sys_lock();
> ```
>
> - Effects: Locks the interface.
    The calling thread is stalled until the lock can be acquired. Only one thread can hold the lock at a time.
> - Deadlock: Calling this function with a thread that already holds a lock may result in a deadlock.

---

> ```c
> emf_cbase_bool_t sys_try_lock();
> ```
>
> - Effects: Tries to lock the interface.
    The function fails if another thread already holds the lock.
> - Returns: `emf_cbase_bool_true` on success and `emf_cbase_bool_false` otherwise.

---

> ```c
> void sys_unlock();
> ```
>
> - Effects: Unlocks the interface.

##### Sys api manual synchronization

> ```c
> const emf_sync_handler_interface_t* sys_get_sync_handler();
> ```
>
> - Ensures: The result of this call will never be `NULL`.
> - Returns: Pointer to the active synchronization handler.

---

> ```c
> void sys_set_sync_handler(const emf_sync_handler_interface_t* sync_handler);
> ```
>
> - Effects: Sets a new synchronization handler.
>   The default synchronization handler is used, if `sync_handler` is `NULL`.
> - Uses: This function can be used by modules, that want to provide a more complex
>       synchronization mechanism than the one presented by the default handler.
> - Swapping: The swapping occurs in three steps:
>   1. The new synchronization handler is locked.
>   2. The new synchronization handler is set as the active synchronization handler.
>   3. The old synchronization handler is unlocked.
> - Remarks: Changing the synchronization handler may break some modules,
>       if they depend on a specific synchronization handler.

### Library api

[library-api]: #library-api

The library api is a collection of procedures that provide a platform agnostic interface to loading shared libraries.
The actual loading of a library is handled by a library loader. Each library loader is associated to a library type.

#### Library Loaders

The job of a library loader is to manage the loading and unloading of libraries. A library loader can be added by
constructing a `emf_cbase_library_loader_interface_t` and then calling the `library_register_loader()`
function.

#### Library types

The library api allows the loading of custom library formats. Each format is identified by a `emf_cbase_library_type_t`
and is associated to exactly one library loader.

#### Predefined library loaders

Some library loaders are always present and can not be removed at runtime.

##### Native library loader

The native library loader is able to load platform-specific libraries (e.g. dlopen()/LoadLibrary()). It is associated to
the `EMF_CBASE_NATIVE_LIBRARY_TYPE_NAME` library type and is reachable with
the `EMF_CBASE_LIBRARY_LOADER_DEFAULT_HANDLE` handle. Libraries with the same absolute path are loaded only once and
subsequent load calls increase their reference count.

On POSIX the library loader defaults to the `RTLD_LAZY` and `RTLD_LOCAL` flags. More control of how to load a library
can be achieved by fetching a pointer to the interface with `library_get_loader_interface()`, casting the result
of `get_internal_interface_fn` to a `emf_cbase_native_library_loader_interface_t` and calling the `load_ext_fn`
function.

If the library has dependencies on other libraries, then it must specify them by specifying an `rpath` on POSIX or
embedding an `Activation Context manifest` into the library on Windows.

#### Library api example

```c
/// `base_interf` is a structure containing the `emf-core-base` interface.
/// `base_handle` is a handle to the `emf-core-base` module.

const emf_cbase_os_path_char_t* library_path = "./example_lib.so";

base_interf->sys_lock(base_handle);

emf_cbase_library_handle_result_t library_handle_res = 
        base_interf->library_load(base_handle, EMF_CBASE_NATIVE_LIBRARY_TYPE_NAME, library_path);
if (library_handle_res.has_error) {
    base_interf->sys_panic(base_handle, "Unable to load the `./example_lib.so` library.");
}

emf_cbase_library_handle_t library_handle = library_handle_res.result;

emf_cbase_library_fn_symbol_result_t fn_sym_res = 
        base_interf->library_get_function_symbol(base_handle, library_handle, "example_fn");
if (fn_sym_res.has_error) {
    base_interf->sys_panic(base_handle, "Unable to load the `example_fn` function from the library.");
}

emf_cbase_library_fn_symbol_t fn_sym = fn_sym_res.result;
void (*fn)(int, int) = (void(*)(int, int))fn_sym.symbol;
(*fn)(5, 7);

emf_cbase_library_result_t result = base_interf->library_unload(base_handle, library_handle);
if (result.has_error) {
    base_interf->sys_panic(base_handle, "Unable to unload the `./example_lib.so` library.");
}

base_interf->sys_unlock(base_handle);
```

#### Library api overview

```c
// loader management
emf_cbase_library_loader_handle_result_t library_register_loader(
        const emf_cbase_library_loader_interface_t* loader_interface, 
        const emf_cbase_library_type_t* library_type);
emf_cbase_library_result_t library_unregister_loader(
        emf_cbase_library_loader_handle_t loader_handle);
emf_cbase_library_loader_interface_result_t library_get_loader_interface(
        emf_cbase_library_loader_handle_t loader_handle);
emf_cbase_library_loader_handle_result_t library_get_loader_handle_from_type(
        const emf_cbase_library_type_t* library_type);
emf_cbase_library_loader_handle_result_t library_get_loader_handle_from_library(
        emf_cbase_library_handle_t library_handle);

// queries
size_t library_get_num_loaders();
emf_cbase_bool_t library_library_exists(emf_cbase_library_handle_t library_handle);
emf_cbase_bool_t library_type_exists(const emf_cbase_library_type_t* library_type);
emf_cbase_library_size_result_t library_get_library_types(emf_cbase_library_type_span_t* buffer);

// internal library management
emf_cbase_library_handle_t library_create_library_handle();
emf_cbase_library_result_t library_remove_library_handle(
        emf_cbase_library_handle_t library_handle);
emf_cbase_library_result_t library_link_library(
        emf_cbase_library_handle_t library_handle, 
        emf_cbase_library_loader_handle_t loader_handle,
        emf_cbase_internal_library_handle_t internal_handle);
emf_cbase_internal_library_handle_result_t library_get_internal_library_handle(
        emf_cbase_library_handle_t library_handle);

// library management
emf_cbase_library_handle_result_t library_load(
        emf_cbase_library_loader_handle_t loader_handle, 
        const emf_cbase_os_path_char_t* library_path);
emf_cbase_library_result_t library_unload(emf_cbase_library_handle_t library_handle);
emf_cbase_library_data_symbol_result_t library_get_data_symbol(
        emf_cbase_library_handle_t library_handle, 
        const char* symbol_name);
emf_cbase_library_fn_symbol_result_t library_get_function_symbol(
        emf_cbase_library_handle_t library_handle, 
        const char* symbol_name);
```

#### Library api functions

##### Library api loader management

> ```c
> emf_cbase_library_loader_handle_result_t library_register_loader(
>       const emf_cbase_library_loader_interface_t* loader_interface, 
>       const emf_cbase_library_type_t* library_type);
> ```
>
> - Effects: Registers a new loader. The loader can load libraries of the type `library_type`.
> - Mandates: `loader_interface != NULL && library_type != NULL`.
> - Failure: The function fails if the library type already exists.
> - Returns: Handle on success, error otherwise.

---

> ```c
> emf_cbase_library_result_t library_unregister_loader(emf_cbase_library_loader_handle_t loader_handle);
> ```
>
> - Effects: Unregisters an existing loader.
> - Failure: The function fails if `loader_handle` is invalid.
> - Returns: Error on failure.

---

> ```c
> emf_cbase_library_loader_interface_result_t library_get_loader_interface(
>       emf_cbase_library_loader_handle_t loader_handle);
> ```
>
> - Effects: Fetches the interface of a library loader.
> - Failure: The function fails if `loader_handle` is invalid.
> - Returns: Interface on success, error otherwise.

---

> ```c
> emf_cbase_library_loader_handle_result_t library_get_loader_handle_from_type(
>       const emf_cbase_library_type_t* library_type);
> ```
>
> - Effects: Fetches the loader handle associated with the library type.
> - Mandates: `library_type != NULL`.
> - Failure: The function fails if `library_type` is not registered.
> - Returns: Handle on success, error otherwise.

---

> ```c
> emf_cbase_library_loader_handle_result_t library_get_loader_handle_from_library(
>       emf_cbase_library_handle_t library_handle);
> ```
>
> - Effects: Fetches the loader handle linked with the library handle.
> - Failure: The function fails if `library_handle` is invalid.
> - Returns: Handle on success, error otherwise.

##### Library api queries

> ```c
> size_t library_get_num_loaders();
> ```
>
> - Effects: Fetches the number of registered loaders.
> - Returns: Number of registered loaders.

---

> ```c
> emf_cbase_bool_t library_library_exists(emf_cbase_library_handle_t library_handle);
> ```
>
> - Effects: Checks if a the library handle is valid.
> - Returns: `emf_cbase_bool_true` if the handle is valid, `emf_cbase_bool_false` otherwise.

---

> ```c
> emf_cbase_bool_t library_type_exists(const emf_cbase_library_type_t* library_type);
> ```
>
> - Effects: Checks if a library type exists.
> - Mandates: `library_type != NULL`.
> - Returns: `emf_cbase_bool_true` if the type is exists, `emf_cbase_bool_false` otherwise.

---

> ```c
> emf_cbase_library_size_result_t library_get_library_types(emf_cbase_library_type_span_t* buffer);
> ```
>
> - Effects: Copies the strings of the registered library types into a buffer.
> - Mandates: `buffer != NULL && buffer->data != NULL`.
> - Failure: The function fails if `buffer->length < library_get_num_loaders()`.
> - Returns: Number of written types on success, error otherwise.

##### Library api internal library management

> ```c
> emf_cbase_library_handle_t library_create_library_handle();
> ```
>
> - Effects: Creates a new unlinked library handle.
> - Remarks: The handle must be linked before use.
> - Returns: Library handle.

---

> ```c
> emf_cbase_library_result_t library_remove_library_handle(emf_cbase_library_handle_t library_handle);
> ```
>
> - Effects: Removes an existing library handle.
> - Failure: The function fails if `library_handle` is invalid.
> - Remarks: Removing the handle does not unload the library.
> - Returns: Error on failure.

---

> ```c
> emf_cbase_library_result_t library_link_library(
>       emf_cbase_library_handle_t library_handle, 
>       emf_cbase_library_loader_handle_t loader_handle,
>       emf_cbase_internal_library_handle_t internal_handle);
> ```
>
> - Effects: Links a library handle to an internal library handle.
    Overrides the internal link of the library handle by setting it to the new library loader and internal handle.
> - Failure: The function fails if `library_handle` or `loader_handle` are invalid.
> - Remarks: Incorrect usage can lead to dangling handles or use-after-free errors.
> - Returns: Error on failure.

---

> ```c
> emf_cbase_internal_library_handle_result_t library_get_internal_library_handle(
>       emf_cbase_library_handle_t library_handle);
> ```
>
> - Effects: Fetches the internal handle linked with the library handle.
> - Failure: The function fails if `library_handle` is invalid.
> - Returns: Handle on success, error otherwise.

##### Library api library management

> ```c
> emf_cbase_library_handle_result_t library_load(
>       emf_cbase_library_loader_handle_t loader_handle, 
>       const emf_cbase_os_path_char_t* library_path);
> ```
>
> - Effects: Loads a library. The resulting handle is unique.
> - Mandates: `library_path != NULL`.
> - Failure: The function fails if `loader_handle` or `library_path` is invalid 
    or the type of the library can not be loaded with the loader.
> - Returns: Handle on success, error otherwise.

---

> ```c
> emf_cbase_library_result_t library_unload(emf_cbase_library_handle_t library_handle);
> ```
>
> - Effects: Unloads a library.
> - Failure: The function fails if `library_handle` is invalid.
> - Returns: Error on failure.

---

> ```c
> emf_cbase_library_data_symbol_result_t library_get_data_symbol(
>       emf_cbase_library_handle_t library_handle, 
>       const char* symbol_name);
> ```
>
> - Effects: Fetches a data symbol from a library.
> - Mandates: `symbol_name != NULL`.
> - Failure: The function fails if `library_handle` is invalid or library does not contain `symbol_name`.
> - Remarks: Some platforms may differentiate between a `function-pointer` and a `data-pointer`.
    See `library_get_function_symbol()` when fetching a function.
> - Returns: Symbol on success, error otherwise.

---

> ```c
> emf_cbase_library_fn_symbol_result_t library_get_function_symbol(
>       emf_cbase_library_handle_t library_handle, 
>       const char* symbol_name);
> ```
>
> - Effects: Fetches a function symbol from a library.
> - Mandates: `symbol_name != NULL`.
> - Failure: The function fails if `library_handle` is invalid or library does not contain `symbol_name`.
> - Remarks: Some platforms may differentiate between a `function-pointer` and a `data-pointer`.
    See `library_get_data_symbol()` when fetching some data.
> - Returns: Symbol on success, error otherwise.

### Module api

[module-api]: #module-api

TBA

### Version api

[version-api]: #version-api

TBA

## Reference

[reference]: #reference

### Constants

[constants]: #constants

### Enums

[enums]: #enums

### Structs

[structs]: #structs

### Types

[types]: #types

## Headers

[headers]: #headers

## Glossary

[glossary]: #glossary

# Drawbacks

[drawbacks]: #drawbacks

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

# Prior art

[prior-art]: #prior-art

# Unresolved questions

[unresolved-questions]: #unresolved-questions

# Future possibilities

[future-possibilities]: #future-possibilities
