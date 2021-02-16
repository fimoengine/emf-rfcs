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
const emf_cbase_sync_handler_interface_t* sys_get_sync_handler();
void sys_set_sync_handler(const emf_cbase_sync_handler_interface_t* sync_handler);
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
> const emf_cbase_sync_handler_interface_t* sys_get_sync_handler();
> ```
>
> - Ensures: The result of this call will never be `NULL`.
> - Returns: Pointer to the active synchronization handler.

---

> ```c
> void sys_set_sync_handler(const emf_cbase_sync_handler_interface_t* sync_handler);
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

The version api implements the versioning scheme specified in
the [versioning-specification RFC](0004-versioning-specification.md).

#### Version api example

```c
/// `base_interf` is a structure containing the `emf-core-base` interface.
/// `base_handle` is a handle to the `emf-core-base` module.

emf_cbase_version_t v1 = base_interf->version_new_short(base_handle, 1, 2, 3);

const char* v2_string = "1.2.3-beta.5+54845652";
emf_cbase_version_const_string_buffer_t v2_string_buff = {
    .data = v2_string, 
    .length = strlen(v2_string)
};
emf_cbase_version_result_t v2_res = 
        base_interf->version_from_string(base_handle, &v2_string_buff);
if (v2_res.has_error) {
    base_interf->sys_lock(base_handle);
    base_interf->sys_panic(base_handle, "Could not construct version from string.");
    base_interf->sys_unlock(base_handle);
}
emf_cbase_version_t v2 = v2_res.result;

if (base_interf->version_compare_weak(base_handle, &v1, &v2) != 0) {
    base_interf->sys_lock(base_handle);
    base_interf->sys_panic(base_handle, "Should not happen.");
    base_interf->sys_unlock(base_handle);
}
```

#### Version api overview

```c
// construction
emf_cbase_version_t version_new_short(
        int32_t major, 
        int32_t minor,
        int32_t patch);
emf_cbase_version_t version_new_long(
        int32_t major, 
        int32_t minor,
        int32_t patch, 
        emf_cbase_version_release_t release_type, 
        int8_t release_number);
emf_cbase_version_t version_new_full(
        int32_t major, 
        int32_t minor,
        int32_t patch, 
        emf_cbase_version_release_t release_type, 
        int8_t release_number, 
        int64_t build);
emf_cbase_version_result_t version_from_string(const emf_cbase_version_const_string_buffer_t* version_string);

// strings
size_t version_string_length_short(const emf_cbase_version_t* version);
size_t version_string_length_long(const emf_cbase_version_t* version);
size_t version_string_length_full(const emf_cbase_version_t* version);
emf_cbase_version_size_result_t version_as_string_short(
        const emf_cbase_version_t* version,
        emf_cbase_version_string_buffer_t* buffer);
emf_cbase_version_size_result_t version_as_string_long(
        const emf_cbase_version_t* version,
        emf_cbase_version_string_buffer_t* buffer);
emf_cbase_version_size_result_t version_as_string_full(
        const emf_cbase_version_t* version,
        emf_cbase_version_string_buffer_t* buffer);
emf_cbase_bool_t version_string_is_valid(const emf_cbase_version_const_string_buffer_t* version_string);

// comparisons
int32_t version_compare(const emf_cbase_version_t* lhs, const emf_cbase_version_t* rhs);
int32_t version_compare_weak(const emf_cbase_version_t* lhs, const emf_cbase_version_t* rhs);
int32_t version_compare_strong(const emf_cbase_version_t* lhs, const emf_cbase_version_t* rhs);
emf_cbase_bool_t version_is_compatible(const emf_cbase_version_t* lhs, const emf_cbase_version_t* rhs);
```

#### Version api functions

##### Version api construction

> ```c
> emf_cbase_version_t version_new_short(
>       int32_t major, 
>       int32_t minor,
>       int32_t patch);
> ```
>
> - Effects: Constructs a new version.
    Constructs a new version with `major`, `minor` and `patch` and sets the rest to `0`.
> - Returns: Constructed version.

---

> ```c
> emf_cbase_version_t version_new_long(
>       int32_t major, 
>       int32_t minor,
>       int32_t patch, 
>       emf_cbase_version_release_t release_type, 
>       int8_t release_number);
> ```
>
> - Effects: Constructs a new version.
    Constructs a new version with `major`, `minor`, `patch`, `release_type` and
    `release_number` and sets the rest to `0`.
> - Returns: Constructed version.

---

> ```c
> emf_cbase_version_t version_new_full(
>       int32_t major, 
>       int32_t minor,
>       int32_t patch, 
>       emf_cbase_version_release_t release_type, 
>       int8_t release_number, 
>       int64_t build);
> ```
>
> - Effects: Constructs a new version.
    Constructs a new version with `major`, `minor`, `patch`, `release_type`, `release_number` and `build`.
> - Returns: Constructed version.

---

> ```c
> emf_cbase_version_result_t version_from_string(const emf_cbase_version_const_string_buffer_t* version_string);
> ```
>
> - Effects: Constructs a version from a string.
> - Mandates: `version_string != NULL && version_string->data != NULL`.
> - Failure: Fails if `version_string_is_valid(version_string) == emf_cbase_bool_false`.
> - Returns: Constructed version.

##### Version api strings

> ```c
> size_t version_string_length_short(const emf_cbase_version_t* version);
> ```
>
> - Effects: Computes the length of the short version string.
> - Mandates: `version != NULL`.
> - Returns: Length of the string.

---

> ```c
> size_t version_string_length_long(const emf_cbase_version_t* version);
> ```
>
> - Effects: Computes the length of the long version string.
> - Mandates: `version != NULL`.
> - Returns: Length of the string.

---

> ```c
> size_t version_string_length_full(const emf_cbase_version_t* version);
> ```
>
> - Effects: Computes the length of the full version string.
> - Mandates: `version != NULL`.
> - Returns: Length of the string.

---

> ```c
> emf_cbase_version_size_result_t version_as_string_short(
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer);
> ```
>
> - Effects: Represents the version as a short string.
> - Mandates: `version != NULL && buffer != NULL && buffer->data != NULL`.
> - Failure: This function fails if `buffer->length < version_string_length_short(version)`.
> - Returns: Number of written characters on success, error otherwise.

---

> ```c
> emf_cbase_version_size_result_t version_as_string_long(
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer);
> ```
>
> - Effects: Represents the version as a long string.
> - Mandates: `version != NULL && buffer != NULL && buffer->data != NULL`.
> - Failure: This function fails if `buffer->length < version_string_length_long(version)`.
> - Returns: Number of written characters on success, error otherwise.

---

> ```c
> emf_cbase_version_size_result_t version_as_string_full(
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer);
> ```
>
> - Effects: Represents the version as a full string.
> - Mandates: `version != NULL && buffer != NULL && buffer->data != NULL`.
> - Failure: This function fails if `buffer->length < version_string_length_full(version)`.
> - Returns: Number of written characters on success, error otherwise.

---

> ```c
> emf_cbase_bool_t version_string_is_valid(const emf_cbase_version_const_string_buffer_t* version_string);
> ```
>
> - Effects: Checks whether the version string is valid.
> - Mandates: `version_string != NULL && version_string->data != NULL`.
> - Returns: `emf_cbase_bool_true` if the string is valid, `emf_cbase_bool_false` otherwise.

## Reference

[reference]: #reference

### Conventions

For the sake of brevity, we will introduce the following macros:

#### BUFFER_T

> ```c
> #define BUFFER_T(NAME, T, LENGTH) \  
>   typedef struct NAME { \
>       T data[LENGTH]; \
>       size_t length; \
>   } NAME;
> ```
>
> - Description: A buffer with a fixed length.

---

#### OPTIONAL_T

> ```c
> #define OPTIONAL_T(NAME, T) \  
>   typedef struct NAME { \
>       union { \
>           T value; \
>           int8_t _dummy; \
>       }; \
>       emf_cbase_bool_t has_value; \
>   } NAME;
> ```
>
> - Description: An optional value.

---

#### RESULT_T

> ```c
> #define RESULT_T(NAME, RES_T, ERR_T) \  
>   typedef struct NAME { \
>       union { \
>           RES_T result; \
>           ERR_T error; \
>       }; \
>       emf_cbase_bool_t has_error; \
>   } NAME;
> ```
>
> - Description: A structure containing a `RES_T` or an `ERR_T`.

---

#### SPAN_T

> ```c
> #define SPAN_T(NAME, T) \  
>   typedef struct NAME { \
>       T* data; \
>       size_t length; \
>   } NAME;
> ```
>
> - Description: A borrowed view of a memory region.

### Constants

[constants]: #constants

> ```c
> #define EMF_CBASE_INTERFACE_EXTENSION_NAME_MAX_LENGTH 32
> ```
>
> - Description: Max length of an interface extension name.

---

> ```c
> #define EMF_CBASE_INTERFACE_INFO_NAME_MAX_LENGTH 32
> ```
>
> - Description: Max length of an interface name string.

---

> ```c
> #define EMF_CBASE_LIBRARY_LOADER_DEFAULT_HANDLE \
>    (emf_cbase_library_loader_handle_t) { emf_cbase_library_predefined_handles_native }
> ```
>
> - Description: Handle to the default library loader.

---

> ```c
> #define EMF_CBASE_LIBRARY_LOADER_TYPE_MAX_LENGTH 64
> ```
>
> - Description: Max length of a library type name.

---

#### EMF_CBASE_MODULE_INFO_NAME_MAX_LENGTH

> ```c
> #define EMF_CBASE_MODULE_INFO_NAME_MAX_LENGTH 32
> ```
>
> - Description: Max length of a module name.

---

#### EMF_CBASE_MODULE_INFO_VERSION_MAX_LENGTH

> ```c
> #define EMF_CBASE_MODULE_INFO_VERSION_MAX_LENGTH 32
> ```
>
> - Description: Max length of a module version string.

---

#### EMF_CBASE_MODULE_LOADER_DEFAULT_HANDLE

> ```c
> #define EMF_CBASE_MODULE_LOADER_DEFAULT_HANDLE \
>    (emf_cbase_module_loader_handle_t) { emf_cbase_module_predefined_handles_native }
> ```
>
> - Description: Handle to the default module loader.

---

#### EMF_CBASE_MODULE_LOADER_TYPE_MAX_LENGTH

> ```c
> #define EMF_CBASE_MODULE_LOADER_TYPE_MAX_LENGTH 64
> ```
>
> - Description: Max length of a module type name.

---

#### EMF_CBASE_NATIVE_LIBRARY_TYPE_NAME

> ```c
> #define EMF_CBASE_NATIVE_LIBRARY_TYPE_NAME "emf::core_base::native"
> ```
>
> - Description: Name of the native library type.

---

#### EMF_CBASE_NATIVE_MODULE_INTERFACE_SYMBOL_NAME

> ```c
> #define EMF_CBASE_NATIVE_MODULE_INTERFACE_SYMBOL_NAME emf_cbase_native_module_interface
> ```
>
> - Description: Name of the native interface symbol.

---

#### EMF_CBASE_NATIVE_MODULE_TYPE_NAME

> ```c
> #define EMF_CBASE_NATIVE_MODULE_TYPE_NAME "emf::core_base::native"
> ```
>
> - Description: Name of the native module type.

---

#### EMF_CBASE_OS_PATH_CHAR

> ```c
> #if defined(Win32) || defined(_WIN32)
> #define EMF_CBASE_OS_PATH_CHAR wchar_t
> #else
> #define EMF_CBASE_OS_PATH_CHAR char
> #endif
> ```
>
> - Description: Character used by the os to represent a path.

---

### Enums

[enums]: #enums

#### emf_cbase_bool_t

> ```c
> typedef enum emf_cbase_bool_t : int8_t {
>     emf_cbase_bool_false = 0,
>     emf_cbase_bool_true = 1
> } emf_cbase_bool_t;
> ```
>
> - Description: An enum describing a boolean value.
>
> | Name                     | Value | Description  |
> | ------------------------ | ----- | ------------ |
> | **emf_cbase_bool_false** | `0`   | False value. |
> | **emf_cbase_bool_true**  | `1`   | True value.  |

---

#### emf_cbase_library_error_t

> ```c
> typedef enum emf_cbase_library_error_t : int32_t {
>     emf_cbase_library_error_library_path_not_found = 0,
>     emf_cbase_library_error_library_handle_invalid = 1,
>     emf_cbase_library_error_loader_handle_invalid = 2,
>     emf_cbase_library_error_loader_library_handle_invalid = 3,
>     emf_cbase_library_error_library_type_invalid = 4,
>     emf_cbase_library_error_library_type_not_found = 5,
>     emf_cbase_library_error_duplicate_library_type = 6,
>     emf_cbase_library_error_symbol_not_found = 7,
>     emf_cbase_library_error_buffer_overflow = 8,
> } emf_cbase_library_error_t;
> ```
>
> - Description: An enum describing all defined error values.
    The values `0-99` are reserved for future use.
>
> | Name                                                      | Value | Description                             |
> | --------------------------------------------------------- | ----- | --------------------------------------- |
> | **emf_cbase_library_error_library_path_not_found**        | `0`   | A path could not be found.              |
> | **emf_cbase_library_error_library_handle_invalid**        | `1`   | The library handle is invalid.          |
> | **emf_cbase_library_error_loader_handle_invalid**         | `2`   | The loader handle is invalid.           |
> | **emf_cbase_library_error_loader_library_handle_invalid** | `3`   | The internal library handle is invalid. |
> | **emf_cbase_library_error_library_type_invalid**          | `4`   | The library type is invalid.            |
> | **emf_cbase_library_error_library_type_not_found**        | `5`   | The library type could not be found.    |
> | **emf_cbase_library_error_duplicate_library_type**        | `6`   | The library type already exists.        |
> | **emf_cbase_library_error_symbol_not_found**              | `7`   | A symbol could not be found.            |
> | **emf_cbase_library_error_buffer_overflow**               | `8`   | The buffer is too small.                |

---

#### emf_cbase_library_predefined_handles_t

> ```c
> typedef enum emf_cbase_library_predefined_handles_t : int32_t {
>     emf_cbase_library_predefined_handles_native = 0,
> } emf_cbase_library_predefined_handles_t;
> ```
>
> - Description: An enum describing all predefined library loader handles.
    The values `0-99` are reserved for future use.
>
> | Name                                            | Value | Description         |
> | ----------------------------------------------- | ----- | ------------------- |
> | **emf_cbase_library_predefined_handles_native** | `0`   | The default handle. |

---

#### emf_cbase_module_error_t

> ```c
> typedef enum emf_cbase_module_error_t : int32_t {
>     emf_cbase_module_error_path_invalid = 0,
>     emf_cbase_module_error_module_state_invalid = 1,
>     emf_cbase_module_error_module_handle_invalid = 2,
>     emf_cbase_module_error_loader_handle_invalid = 3,
>     emf_cbase_module_error_loader_module_handle_invalid = 4,
>     emf_cbase_module_error_module_type_invalid = 5,
>     emf_cbase_module_error_module_type_not_found = 6,
>     emf_cbase_module_error_duplicate_module_type = 7,
>     emf_cbase_module_error_interface_not_found = 8,
>     emf_cbase_module_error_duplicate_interface = 9,
>     emf_cbase_module_error_module_dependency_not_found = 10,
>     emf_cbase_module_error_buffer_overflow = 11,
> } emf_cbase_module_error_t;
> ```
>
> - Description: An enum describing all defined error values.
    The values `0-99` are reserved for future use.
>
> | Name                                                    | Value | Description                                            |
> | ------------------------------------------------------- | ----- | ------------------------------------------------------ |
> | **emf_cbase_module_error_path_invalid**                 | `0`   | The path is invalid.                                   |
> | **emf_cbase_module_error_module_state_invalid**         | `1`   | The state of the module is invalid.                    |
> | **emf_cbase_module_error_module_handle_invalid**        | `2`   | The module handle is invalid.                          |
> | **emf_cbase_module_error_loader_handle_invalid**        | `3`   | The loader is invalid.                                 |
> | **emf_cbase_module_error_loader_module_handle_invalid** | `4`   | The internal module handle is invalid.                 |
> | **emf_cbase_module_error_module_type_invalid**          | `5`   | The module type is invalid.                            |
> | **emf_cbase_module_error_module_type_not_found**        | `6`   | The module type does not exist.                        |
> | **emf_cbase_module_error_duplicate_module_type**        | `7`   | The operation would result in a duplicate module type. |
> | **emf_cbase_module_error_interface_not_found**          | `8`   | The interface could not be found.                      |
> | **emf_cbase_module_error_duplicate_interface**          | `9`   | The operation would result in a duplicate interface.   |
> | **emf_cbase_module_error_module_dependency_not_found**  | `10`  | A dependency could not be found.                       |
> | **emf_cbase_module_error_buffer_overflow**              | `11`  | The operation would result in a buffer overflow.       |

---

#### emf_cbase_module_predefined_handles_t

> ```c
> typedef enum emf_cbase_module_predefined_handles_t : int32_t {
>     emf_cbase_module_predefined_handles_native = 0,
> } emf_cbase_module_predefined_handles_t;
> ```
>
> - Description: An enum describing all predefined module loader handles.
    The values `0-99` are reserved for future use.
>
> | Name                                           | Value | Description            |
> | ---------------------------------------------- | ----- | ---------------------- |
> | **emf_cbase_module_predefined_handles_native** | `0`   | The default handle.    |

---

#### emf_cbase_module_status_t

> ```c
> typedef enum emf_cbase_module_status_t : int32_t {
>     emf_cbase_module_status_unloaded = 0,
>     emf_cbase_module_status_terminated = 1,
>     emf_cbase_module_status_ready = 2,
> } emf_cbase_module_status_t;
> ```
>
> - Description: An enum describing all possible module states.
>
> | Name                                   | Value | Description                      |
> | -------------------------------------- | ----- | -------------------------------- |
> | **emf_cbase_module_status_unloaded**   | `0`   | The module has not been loaded.  |
> | **emf_cbase_module_status_terminated** | `1`   | The module has been loaded.      |
> | **emf_cbase_module_status_ready**      | `2`   | The module has been initialized. |

---

#### emf_cbase_version_error_t

> ```c
> typedef enum emf_cbase_version_error_t : int32_t {
>     emf_cbase_version_error_invalid_string = 0,
>     emf_cbase_version_error_buffer_overflow = 1,
> } emf_cbase_version_error_t;
> ```
>
> - Description: An enum describing the possible error values of the version api.
    The values `0-99` are reserved for future use.
>
> | Name                                        | Value | Description              |
> | ------------------------------------------- | ----- | ------------------------ |
> | **emf_cbase_version_error_invalid_string**  | `0`   | The string is invalid.   |
> | **emf_cbase_version_error_buffer_overflow** | `1`   | The buffer is too small. |

---

#### emf_cbase_version_release_t

> ```c
> typedef enum emf_cbase_version_release_t : int8_t {
>     emf_cbase_version_release_stable = 0,
>     emf_cbase_version_release_unstable = 1,
>     emf_cbase_version_release_beta = 2,
> } emf_cbase_version_release_t;
> ```
>
> - Description: An enum describing the release type of a version.
>
> | Name                                    | Value | Description            |
> | --------------------------------------- | ----- | ---------------------- |
> | **emf_cbase_version_release_stable**    | `0`   | A stable release.      |
> | **emf_cbase_version_release_pre_alpha** | `1`   | An unstable release.   |
> | **emf_cbase_version_release_beta**      | `2`   | A beta release.        |

### Structs

[structs]: #structs

#### emf_cbase_data_symbol_t

> ```c
> typedef struct emf_cbase_data_symbol_t {
>     void* symbol;
> } emf_cbase_data_symbol_t;
> ```
>
> - Description: A data symbol contained in a library.
> - Remarks: `symbol` may not be `NULL`.

---

#### emf_cbase_fn_symbol_t

> ```c
> typedef struct emf_cbase_fn_symbol_t {
>     emf_cbase_fn_t symbol;
> } emf_cbase_fn_symbol_t;
> ```
>
> - Description: A data symbol contained in a library.
> - Remarks: `symbol` may not be `NULL`.

---

#### emf_cbase_interface_descriptor_const_span_result_t

> ```c
> RESULT(emf_cbase_interface_descriptor_const_span_result_t, 
>       emf_cbase_interface_descriptor_const_span_t, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_interface_descriptor_const_span_t` or an `emf_cbase_module_error_t`.

---

#### emf_cbase_interface_descriptor_const_span_t

> ```c
> SPAN_T(emf_cbase_interface_descriptor_const_span_t, const emf_cbase_interface_descriptor_t)
> ```
>
> - Description: A span of constant interface descriptors.

---

#### emf_cbase_interface_descriptor_span_t

> ```c
> SPAN_T(emf_cbase_interface_descriptor_span_t, emf_cbase_interface_descriptor_t)
> ```
>
> - Description: A span of interface descriptors.

---

#### emf_cbase_interface_descriptor_t

> ```c
> typedef struct emf_cbase_interface_descriptor_t {
>     emf_cbase_interface_name_t name;
>     emf_cbase_version_t version;
>     emf_cbase_interface_extension_span_t extensions;
> } emf_cbase_interface_descriptor_t;
> ```
>
> - Description: Descriptor of an interface.

---

#### emf_cbase_interface_extension_span_t

> ```c
> SPAN_T(emf_cbase_interface_extension_span_t, const emf_cbase_interface_extension_t)
> ```
>
> - Description: A span of interface extensions.

---

#### emf_cbase_interface_extension_t

> ```c
> BUFFER_T(emf_cbase_interface_extension_t, char, EMF_CBASE_INTERFACE_EXTENSION_NAME_MAX_LENGTH)
> ```
>
> - Description: Extension of an interface.

---

#### emf_cbase_interface_name_t

> ```c
> BUFFER_T(emf_cbase_interface_name_t, char, EMF_CBASE_INTERFACE_INFO_NAME_MAX_LENGTH)
> ```
>
> - Description: Name of an interface.

---

#### emf_cbase_library_data_symbol_result_t

> ```c
> RESULT_T(emf_cbase_library_data_symbol_result_t, emf_cbase_data_symbol_t, emf_cbase_library_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_data_symbol_t` or an `emf_cbase_library_error_t`.

---

#### emf_cbase_library_fn_symbol_result_t

> ```c
> RESULT_T(emf_cbase_library_fn_symbol_result_t, emf_cbase_fn_symbol_t, emf_cbase_library_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_fn_symbol_t` or an `emf_cbase_library_error_t`.

---

#### emf_cbase_library_handle_result_t

> ```c
> RESULT_T(emf_cbase_library_handle_result_t, emf_cbase_library_handle_t, emf_cbase_library_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_library_handle_t` or an `emf_cbase_library_error_t`.

---

#### emf_cbase_library_handle_t

> ```c
> typedef struct emf_cbase_library_handle_t {
>     int32_t id;
> } emf_cbase_library_handle_t;
> ```
>
> - Description: Handle to a library.

---

#### emf_cbase_library_loader_handle_result_t

> ```c
> RESULT_T(emf_cbase_library_loader_handle_result_t, emf_cbase_library_loader_handle_t, emf_cbase_library_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_library_loader_handle_t` or an `emf_cbase_library_error_t`.

---

#### emf_cbase_library_loader_handle_t

> ```c
> typedef struct emf_cbase_library_loader_handle_t {
>     int32_t id;
> } emf_cbase_library_loader_handle_t;
> ```
>
> - Description: Handle to a library loader.

---

#### emf_cbase_library_loader_interface_result_t

> ```c
> emf_cbase_library_loader_interface_result_t,
>     const emf_cbase_library_loader_interface_t*, emf_cbase_library_error_t
> ```
>
> - Description: A struct containing either an `const emf_cbase_library_loader_interface_t*` or an `emf_cbase_library_error_t`.
> - Remarks: In case it contains a `const emf_cbase_library_loader_interface_t*`, it may never be `NULL`.

---

#### emf_cbase_library_loader_interface_t

> ```c
> typedef struct emf_cbase_library_loader_interface_t {
>     emf_cbase_library_loader_t* library_loader;
>     emf_cbase_library_loader_interface_load_fn_t load_fn;
>     emf_cbase_library_loader_interface_unload_fn_t unload_fn;
>     emf_cbase_library_loader_interface_get_data_symbol_fn_t get_data_symbol_fn;
>     emf_cbase_library_loader_interface_get_function_symbol_fn_t get_function_fn;
>     emf_cbase_library_loader_interface_get_internal_interface_fn_t get_internal_interface_fn;
> } emf_cbase_library_loader_interface_t;
> ```
>
> - Description: Interface of a library loader.

---

#### emf_cbase_internal_library_handle_result_t

> ```c
> RESULT_T(emf_cbase_internal_library_handle_result_t, emf_cbase_internal_library_handle_t, emf_cbase_library_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_internal_library_handle_t` or an `emf_cbase_library_error_t`.

---

#### emf_cbase_internal_library_handle_t

> ```c
> typedef struct emf_cbase_library_loader_library_handle_t {
>     intptr_t id;
> } emf_cbase_library_loader_library_handle_t;
> ```
>
> - Description: Internal handle to a library.

---

#### emf_cbase_library_loader_t

> ```c
> typedef struct emf_cbase_library_loader_t emf_cbase_library_loader_t;
> ```
>
> - Description: Opaque structure representing a library loader.

---

#### emf_cbase_library_result_t

> ```c
> RESULT_T(emf_cbase_library_result_t, int8_t, emf_cbase_library_error_t)
> ```
>
> - Description: A struct containing either an `int8_t` or an `emf_cbase_library_error_t`.
> - Remarks: The `int8_t` value should not be used, as it is never initialized.

---

#### emf_cbase_library_size_result_t

> ```c
> RESULT_T(emf_cbase_library_size_result_t, size_t, emf_cbase_library_error_t)
> ```
>
> - Description: A struct containing either a `size_t` or an `emf_cbase_library_error_t`.

---

#### emf_cbase_library_type_span_t

> ```c
> SPAN_T(emf_cbase_library_type_span_t, emf_cbase_library_type_t)
> ```
>
> - Description: A span of library types.

---

#### emf_cbase_library_type_t

> ```c
> BUFFER_T(emf_cbase_library_type_t, char, EMF_CBASE_LIBRARY_LOADER_TYPE_MAX_LENGTH)
> ```
>
> - Description: Type of a library.

---

#### emf_cbase_module_handle_result_t

> ```c
> RESULT_T(emf_cbase_module_handle_result_t, emf_cbase_module_handle_t, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_module_handle_t` or an `emf_cbase_module_error_t`.

---

#### emf_cbase_module_handle_t

> ```c
> typedef struct emf_cbase_module_handle_t {
>     int32_t id;
> } emf_cbase_module_handle_t;
> ```
>
> - Description: Handle to a module.

---

#### emf_cbase_module_info_ptr_result_t

> ```c
> RESULT_T(emf_cbase_module_info_ptr_result_t, const emf_cbase_module_info_t*, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `const emf_cbase_module_info_t*` or an `emf_cbase_module_error_t`.
> - Remarks: If it contains a `const emf_cbase_module_info_t*` it may not be `NULL`.

---

#### emf_cbase_module_info_span_t

> ```c
> SPAN_T(emf_cbase_module_info_span_t, emf_cbase_module_info_t)
> ```
>
> - Description: Span of module infos.

---

#### emf_cbase_module_info_t

> ```c
> typedef struct emf_cbase_module_info_t {
>     emf_cbase_module_name_t name;
>     emf_cbase_module_version_t version;
> } emf_cbase_module_info_t;
> ```
>
> - Description: Module info.

---

#### emf_cbase_module_interface_result_t

> ```c
> RESULT_T(emf_cbase_module_interface_result_t, emf_cbase_module_interface_t, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_module_interface_t` or an `emf_cbase_module_error_t`.

---

#### emf_cbase_module_interface_t

> ```c
> typedef struct emf_cbase_module_interface_t {
>     void* interface;
> } emf_cbase_module_interface_t;
> ```
>
> - Description: An interface.
> - Remarks: `interface` may not be `NULL`.

---

#### emf_cbase_module_loader_handle_result_t

> ```c
> RESULT_T(emf_cbase_module_loader_handle_result_t, emf_cbase_module_loader_handle_t, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_module_loader_handle_t` or an `emf_cbase_module_error_t`.

---

#### emf_cbase_module_loader_handle_t

> ```c
> typedef struct emf_cbase_module_loader_handle_t {
>     int32_t id;
> } emf_cbase_module_loader_handle_t;
> ```
>
> - Description: Handle to a module loader.

---

#### emf_cbase_module_loader_interface_result_t

> ```c
> RESULT_T(emf_cbase_module_loader_interface_result_t,
>       const emf_cbase_module_loader_interface_t*, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `const emf_cbase_module_loader_interface_t*` or an `emf_cbase_module_error_t`.
> - Remarks: If it contains a `const emf_cbase_module_loader_interface_t*`, it may never be `NULL`.

---

#### emf_cbase_module_loader_interface_t

> ```c
> typedef struct emf_cbase_module_loader_interface_t {
>     emf_cbase_module_loader_t* module_loader;
>     emf_cbase_module_loader_interface_add_module_fn_t add_module_fn;
>     emf_cbase_module_loader_interface_remove_module_fn_t remove_module_fn;
>     emf_cbase_module_loader_interface_fetch_status_fn_t fetch_status_fn;
>     emf_cbase_module_loader_interface_load_fn_t load_fn;
>     emf_cbase_module_loader_interface_unload_fn_t unload_fn;
>     emf_cbase_module_loader_interface_initialize_fn_t initialize_fn;
>     emf_cbase_module_loader_interface_terminate_fn_t terminate_fn;
>     emf_cbase_module_loader_interface_get_module_info_fn_t get_module_info_fn;
>     emf_cbase_module_loader_interface_get_exportable_interfaces_fn_t get_exportable_interfaces_fn;
>     emf_cbase_module_loader_interface_get_runtime_dependencies_fn_t get_runtime_dependencies_fn;
>     emf_cbase_module_loader_interface_get_interface_fn_t get_interface_fn;
>     emf_cbase_module_loader_interface_get_load_dependencies_fn_t get_load_dependencies_fn;
>     emf_cbase_module_loader_interface_get_module_path_fn_t get_module_path_fn;
>     emf_cbase_module_loader_interface_get_internal_interface_fn_t get_internal_interface_fn;
> } emf_cbase_module_loader_interface_t;
> ```
>
> - Description: Interface of a module loader.

---

#### emf_cbase_module_result_t

> ```c
> RESULT_T(emf_cbase_module_result_t, int8_t, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `int8_t` or an `emf_cbase_module_error_t`.
> - Remarks: The `int8_t` value should not be used, as it is never initialized.

---

#### emf_cbase_internal_module_handle_result_t

> ```c
> RESULT_T(emf_cbase_internal_module_handle_result_t, emf_cbase_internal_module_handle_t, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_internal_module_handle_t` or an `emf_cbase_module_error_t`.

---

#### emf_cbase_internal_module_handle_t

> ```c
> typedef struct emf_cbase_module_loader_module_handle_t {
>     intptr_t id;
> } emf_cbase_module_loader_module_handle_t;
> ```
>
> - Description: Internal module handle used by module loaders.

---

#### emf_cbase_module_loader_t

> ```c
> typedef struct emf_cbase_module_loader_t emf_cbase_module_loader_t;
> ```
>
> - Description: Opaque structure representing a module loader.

---

#### emf_cbase_module_name_t

> ```c
> BUFFER_T(emf_cbase_module_name_t, char, EMF_CBASE_MODULE_INFO_NAME_MAX_LENGTH)
> ```
>
> - Description: Name of a module.

---

#### emf_cbase_module_size_result_t

> ```c
> RESULT_T(emf_cbase_module_size_result_t, size_t, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either a `size_t` or an `emf_cbase_module_error_t`.

---

#### emf_cbase_module_status_result_t

> ```c
> RESULT_T(emf_cbase_module_status_result_t, emf_cbase_module_status_t, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_module_status_t` or an `emf_cbase_module_error_t`.

---

#### emf_cbase_module_type_span_t

> ```c
> SPAN_T(emf_cbase_module_type_span_t, emf_cbase_module_type_t)
> ```
>
> - Description: A span of module types.

---

#### emf_cbase_module_type_t

> ```c
> BUFFER_T(emf_cbase_module_type_t, char, EMF_CBASE_MODULE_LOADER_TYPE_MAX_LENGTH)
> ```
>
> - Description: Type of a module
> - Remarks: A module name is modelled as an `UTF-8` encoded string, without a `\0` terminator.

---

#### emf_cbase_module_version_t

> ```c
> BUFFER_T(emf_cbase_module_version_t, char, EMF_CBASE_MODULE_INFO_VERSION_MAX_LENGTH)
> ```
>
> - Description: Version string of a module.
> - Remarks: A version string is modelled as an `UTF-8` encoded string, without a `\0` terminator.

---

#### emf_cbase_native_library_loader_interface_t

> ```c
> typedef struct emf_cbase_native_library_loader_interface_t {
>     const emf_cbase_library_loader_interface_t* library_loader_interface;
>     emf_cbase_native_library_loader_interface_load_ext_fn_t load_ext_fn;
> } emf_cbase_native_library_loader_interface_t;
> ```
>
> - Description: Interface of the native library loader.
> - Remarks: `library_loader_interface` may not be `NULL`.

---

#### emf_cbase_native_module_interface_t

> ```c
> typedef struct emf_cbase_native_module_interface_t {
>     emf_cbase_native_module_interface_load_fn_t load_fn;
>     emf_cbase_native_module_interface_unload_fn_t unload_fn;
>     emf_cbase_native_module_interface_initialize_fn_t initialize_fn;
>     emf_cbase_native_module_interface_terminate_fn_t terminate_fn;
>     emf_cbase_native_module_interface_get_module_info_fn_t get_module_info_fn;
>     emf_cbase_native_module_interface_get_exportable_interfaces_fn_t get_exportable_interfaces_fn;
>     emf_cbase_native_module_interface_get_runtime_dependencies_fn_t get_runtime_dependencies_fn;
>     emf_cbase_native_module_interface_get_interface_fn_t get_interface_fn;
>     emf_cbase_native_module_interface_get_load_dependencies_fn_t get_load_dependencies_fn;
> } emf_cbase_native_module_interface_t;
> ```
>
> - Description: Interface of a native module.

---

#### emf_cbase_native_module_loader_interface_t

> ```c
> typedef struct emf_cbase_native_module_loader_interface_t {
>     const emf_cbase_module_loader_interface_t* module_loader_interface;
>     emf_cbase_native_module_loader_interface_get_native_module_fn_t get_native_module_fn;
> } emf_cbase_native_module_loader_interface_t;
> ```
>
> - Description: Interface of the native module loader.
> - Remarks: `module_loader_interface` may never be `NULL`.

---

#### emf_cbase_native_module_ptr_result_t

> ```c
> RESULT_T(emf_cbase_native_module_ptr_result_t, emf_cbase_native_module_t*, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_native_module_t*` or an `emf_cbase_module_error_t`.

---

#### emf_cbase_native_module_t

> ```c
> typedef struct emf_cbase_native_module_t emf_cbase_native_module_t;
> ```
>
> - Description: Opaque structure representing a native module.

---

#### emf_cbase_os_path_char_result_t

> ```c
> RESULT_T(emf_cbase_os_path_char_result_t, const emf_cbase_os_path_char_t*, emf_cbase_module_error_t)
> ```
>
> - Description: A struct containing either a `const emf_cbase_os_path_char_t*` or an `emf_cbase_module_error_t`.
> - Remarks. If it contains a `const emf_cbase_os_path_char_t*`, it may never be `NULL`.

---

#### emf_cbase_sync_handler_t

> ```c
> typedef struct emf_cbase_sync_handler_t emf_cbase_sync_handler_t;
> ```
>
> - Description: Opaque structure representing a synchronisation handler.

---

#### emf_cbase_sys_fn_optional_t

> ```c
> OPTIONAL_T(emf_cbase_sys_fn_optional_t, emf_cbase_fn_t)
> ```
>
> - Description: An optional `emf_cbase_fn_t` value.
> - Remarks: If it contains the value, it may never be `NULL`.

---

#### emf_cbase_t

> ```c
> typedef struct emf_cbase_t emf_cbase_t;
> ```
>
> - Description: Opaque structure representing the `emf-core-base` interface.

---

#### emf_cbase_version_const_string_buffer_t

> ```c
> SPAN_T(emf_cbase_version_const_string_buffer_t, const char)
> ```
>
> - Description: A buffer containing a constant string representation of a version.
> - Remarks: The string contains only `ASCII` characters and is not terminated with a `\0`.

---

#### emf_cbase_version_result_t

> ```c
> RESULT_T(emf_cbase_version_result_t, emf_cbase_version_t, emf_cbase_version_error_t)
> ```
>
> - Description: A struct containing either an `emf_cbase_version_t` or an `emf_cbase_version_error_t`.

---

#### emf_cbase_version_size_result_t

> ```c
> RESULT_T(emf_cbase_version_size_result_t, size_t, emf_cbase_version_error_t)
> ```
>
> - Description: A struct containing either a `size_t` or an `emf_cbase_version_error_t`.

---

#### emf_cbase_version_string_buffer_t

> ```c
> SPAN_T(emf_cbase_version_string_buffer_t, char)
> ```
>
> - Description: A buffer to store the string representation of a version.
> - Remarks: The string contains only `ASCII` characters and is not terminated with a `\0`.

---

#### emf_cbase_version_t

> ```c
> typedef struct emf_cbase_version_t {
>     int32_t major;
>     int32_t minor;
>     int32_t patch;
>     int64_t build_number;
>     int8_t release_number;
>     emf_cbase_version_release_t release_type;
> } emf_cbase_version_t;
> ```
>
> - Description: A version.
> - Remarks: If `release_type == emf_cbase_version_release_stable` then `release_number == 0`.

---

#### emf_cbase_sync_handler_interface_t

> ```c
> typedef struct emf_sync_handler_interface_t {
>     emf_cbase_sync_handler_t* sync_handler;
>     emf_cbase_sync_handler_lock_fn_t lock_fn;
>     emf_cbase_sync_handler_try_lock_fn_t try_lock_fn;
>     emf_cbase_sync_handler_unlock_fn_t unlock_fn;
> } emf_sync_handler_interface_t;
> ```
>
> - Description: Interface of a synchronisation handler.

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
