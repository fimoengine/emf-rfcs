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

This RFC proposes an interface, which poses as a backbone for a loosely coupled game engine. This interface named 
**emf-core-base** specifies the lifecycle and the basic capabilities of a conforming engine.

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
_Noreturn void sys_shutdown()
_Noreturn void sys_panic(const char* error)

// feature request
emf_cbase_bool_t sys_has_function(emf_cbase_fn_ptr_id_t fn_id)
emf_cbase_sys_fn_optional_t sys_get_function(emf_cbase_fn_ptr_id_t fn_id)

// synchronisation
void sys_lock()
emf_cbase_bool_t sys_try_lock()
void sys_unlock()

// manual synchronisation
const emf_cbase_sync_handler_interface_t* sys_get_sync_handler()
void sys_set_sync_handler(const emf_cbase_sync_handler_interface_t* sync_handler)
```

#### Sys api functions

##### Sys api termination

###### sys_shutdown

> ```c
> _Noreturn void sys_shutdown()
> ```
>
> - Effects: Sends a termination signal.

---

###### sys_panic

> ```c
> _Noreturn void sys_panic(const char* error)
> ```
>
> - Effects: Execution of the program is stopped abruptly. The error may be logged.

##### Sys api feature request

###### sys_has_function

> ```c
> emf_cbase_bool_t sys_has_function(emf_cbase_fn_ptr_id_t fn_id)
> ```
>
> - Effects: Checks if a function is implemented.
> - Returns: `emf_cbase_bool_true` if the function exists, `emf_cbase_bool_false` otherwise.

---

###### sys_get_function

> ```c
> emf_cbase_sys_fn_optional_t sys_get_function(emf_cbase_fn_ptr_id_t fn_id)
> ```
>
> - Returns: Function pointer to the requested function.

##### Sys api synchronization

###### sys_lock

> ```c
> void sys_lock()
> ```
>
> - Effects: Locks the interface.
    The calling thread is stalled until the lock can be acquired. Only one thread can hold the lock at a time.
> - Deadlock: Calling this function with a thread that already holds a lock may result in a deadlock.

---

###### sys_try_lock

> ```c
> emf_cbase_bool_t sys_try_lock()
> ```
>
> - Effects: Tries to lock the interface.
    The function fails if another thread already holds the lock.
> - Returns: `emf_cbase_bool_true` on success and `emf_cbase_bool_false` otherwise.

---

###### sys_unlock

> ```c
> void sys_unlock()
> ```
>
> - Effects: Unlocks the interface.

##### Sys api manual synchronization

###### sys_get_sync_handler

> ```c
> const emf_cbase_sync_handler_interface_t* sys_get_sync_handler()
> ```
>
> - Ensures: The result of this call will never be `NULL`.
> - Returns: Pointer to the active synchronization handler.

---

###### sys_set_sync_handler

> ```c
> void sys_set_sync_handler(const emf_cbase_sync_handler_interface_t* sync_handler)
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

base_interf->sys_lock_fn(base_handle);

emf_cbase_library_handle_result_t library_handle_res = 
        base_interf->library_load_fn(base_handle, EMF_CBASE_NATIVE_LIBRARY_TYPE_NAME, library_path);
if (library_handle_res.has_error) {
    base_interf->sys_panic_fn(base_handle, "Unable to load the `./example_lib.so` library.");
}

emf_cbase_library_handle_t library_handle = library_handle_res.result;

emf_cbase_library_fn_symbol_result_t fn_sym_res = 
        base_interf->library_get_function_symbol_fn(base_handle, library_handle, "example_fn");
if (fn_sym_res.has_error) {
    base_interf->sys_panic_fn(base_handle, "Unable to load the `example_fn` function from the library.");
}

emf_cbase_library_fn_symbol_t fn_sym = fn_sym_res.result;
void (*fn)(int, int) = (void(*)(int, int))fn_sym.symbol;
(*fn)(5, 7);

emf_cbase_library_result_t result = base_interf->library_unload_fn(base_handle, library_handle);
if (result.has_error) {
    base_interf->sys_panic_fn(base_handle, "Unable to unload the `./example_lib.so` library.");
}

base_interf->sys_unlock_fn(base_handle);
```

#### Library api overview

```c
// loader management
emf_cbase_library_loader_handle_result_t library_register_loader(
        const emf_cbase_library_loader_interface_t* loader_interface, 
        const emf_cbase_library_type_t* library_type)
emf_cbase_library_result_t library_unregister_loader(
        emf_cbase_library_loader_handle_t loader_handle)
emf_cbase_library_loader_interface_result_t library_get_loader_interface(
        emf_cbase_library_loader_handle_t loader_handle)
emf_cbase_library_loader_handle_result_t library_get_loader_handle_from_type(
        const emf_cbase_library_type_t* library_type)
emf_cbase_library_loader_handle_result_t library_get_loader_handle_from_library(
        emf_cbase_library_handle_t library_handle)

// queries
size_t library_get_num_loaders()
emf_cbase_bool_t library_library_exists(emf_cbase_library_handle_t library_handle)
emf_cbase_bool_t library_type_exists(const emf_cbase_library_type_t* library_type)
emf_cbase_library_size_result_t library_get_library_types(emf_cbase_library_type_span_t* buffer)

// internal library management
emf_cbase_library_handle_t library_create_library_handle()
emf_cbase_library_result_t library_remove_library_handle(
        emf_cbase_library_handle_t library_handle)
emf_cbase_library_result_t library_link_library(
        emf_cbase_library_handle_t library_handle, 
        emf_cbase_library_loader_handle_t loader_handle,
        emf_cbase_internal_library_handle_t internal_handle)
emf_cbase_internal_library_handle_result_t library_get_internal_library_handle(
        emf_cbase_library_handle_t library_handle)

// library management
emf_cbase_library_handle_result_t library_load(
        emf_cbase_library_loader_handle_t loader_handle, 
        const emf_cbase_os_path_char_t* library_path)
emf_cbase_library_result_t library_unload(emf_cbase_library_handle_t library_handle)
emf_cbase_library_data_symbol_result_t library_get_data_symbol(
        emf_cbase_library_handle_t library_handle, 
        const char* symbol_name)
emf_cbase_library_fn_symbol_result_t library_get_function_symbol(
        emf_cbase_library_handle_t library_handle, 
        const char* symbol_name)
```

#### Library api functions

##### Library api loader management

###### library_register_loader

> ```c
> emf_cbase_library_loader_handle_result_t library_register_loader(
>       const emf_cbase_library_loader_interface_t* loader_interface, 
>       const emf_cbase_library_type_t* library_type)
> ```
>
> - Effects: Registers a new loader. The loader can load libraries of the type `library_type`.
> - Mandates: `loader_interface != NULL && library_type != NULL`.
> - Failure: The function fails if the library type already exists.
> - Returns: Handle on success, error otherwise.

---

###### library_unregister_loader

> ```c
> emf_cbase_library_result_t library_unregister_loader(emf_cbase_library_loader_handle_t loader_handle)
> ```
>
> - Effects: Unregisters an existing loader.
> - Failure: The function fails if `loader_handle` is invalid.
> - Returns: Error on failure.

---

###### library_get_loader_interface

> ```c
> emf_cbase_library_loader_interface_result_t library_get_loader_interface(
>       emf_cbase_library_loader_handle_t loader_handle)
> ```
>
> - Effects: Fetches the interface of a library loader.
> - Failure: The function fails if `loader_handle` is invalid.
> - Returns: Interface on success, error otherwise.

---

###### library_get_loader_handle_from_type

> ```c
> emf_cbase_library_loader_handle_result_t library_get_loader_handle_from_type(
>       const emf_cbase_library_type_t* library_type)
> ```
>
> - Effects: Fetches the loader handle associated with the library type.
> - Mandates: `library_type != NULL`.
> - Failure: The function fails if `library_type` is not registered.
> - Returns: Handle on success, error otherwise.

---

###### library_get_loader_handle_from_library

> ```c
> emf_cbase_library_loader_handle_result_t library_get_loader_handle_from_library(
>       emf_cbase_library_handle_t library_handle)
> ```
>
> - Effects: Fetches the loader handle linked with the library handle.
> - Failure: The function fails if `library_handle` is invalid.
> - Returns: Handle on success, error otherwise.

##### Library api queries

###### library_get_num_loaders

> ```c
> size_t library_get_num_loaders()
> ```
>
> - Effects: Fetches the number of registered loaders.
> - Returns: Number of registered loaders.

---

###### library_library_exists

> ```c
> emf_cbase_bool_t library_library_exists(emf_cbase_library_handle_t library_handle)
> ```
>
> - Effects: Checks if a the library handle is valid.
> - Returns: `emf_cbase_bool_true` if the handle is valid, `emf_cbase_bool_false` otherwise.

---

###### library_type_exists

> ```c
> emf_cbase_bool_t library_type_exists(const emf_cbase_library_type_t* library_type)
> ```
>
> - Effects: Checks if a library type exists.
> - Mandates: `library_type != NULL`.
> - Returns: `emf_cbase_bool_true` if the type is exists, `emf_cbase_bool_false` otherwise.

---

###### library_get_library_types

> ```c
> emf_cbase_library_size_result_t library_get_library_types(emf_cbase_library_type_span_t* buffer)
> ```
>
> - Effects: Copies the strings of the registered library types into a buffer.
> - Mandates: `buffer != NULL && buffer->data != NULL`.
> - Failure: The function fails if `buffer->length < library_get_num_loaders()`.
> - Returns: Number of written types on success, error otherwise.

##### Library api internal library management

###### library_create_library_handle

> ```c
> emf_cbase_library_handle_t library_create_library_handle()
> ```
>
> - Effects: Creates a new unlinked library handle.
> - Remarks: The handle must be linked before use.
> - Returns: Library handle.

---

###### library_remove_library_handle

> ```c
> emf_cbase_library_result_t library_remove_library_handle(emf_cbase_library_handle_t library_handle)
> ```
>
> - Effects: Removes an existing library handle.
> - Failure: The function fails if `library_handle` is invalid.
> - Remarks: Removing the handle does not unload the library.
> - Returns: Error on failure.

---

###### library_link_library

> ```c
> emf_cbase_library_result_t library_link_library(
>       emf_cbase_library_handle_t library_handle, 
>       emf_cbase_library_loader_handle_t loader_handle,
>       emf_cbase_internal_library_handle_t internal_handle)
> ```
>
> - Effects: Links a library handle to an internal library handle.
    Overrides the internal link of the library handle by setting it to the new library loader and internal handle.
> - Failure: The function fails if `library_handle` or `loader_handle` are invalid.
> - Remarks: Incorrect usage can lead to dangling handles or use-after-free errors.
> - Returns: Error on failure.

---

###### library_get_internal_library_handle

> ```c
> emf_cbase_internal_library_handle_result_t library_get_internal_library_handle(
>       emf_cbase_library_handle_t library_handle)
> ```
>
> - Effects: Fetches the internal handle linked with the library handle.
> - Failure: The function fails if `library_handle` is invalid.
> - Returns: Handle on success, error otherwise.

##### Library api library management

###### library_load

> ```c
> emf_cbase_library_handle_result_t library_load(
>       emf_cbase_library_loader_handle_t loader_handle, 
>       const emf_cbase_os_path_char_t* library_path)
> ```
>
> - Effects: Loads a library. The resulting handle is unique.
> - Mandates: `library_path != NULL`.
> - Failure: The function fails if `loader_handle` or `library_path` is invalid
    or the type of the library can not be loaded with the loader.
> - Returns: Handle on success, error otherwise.

---

###### library_unload

> ```c
> emf_cbase_library_result_t library_unload(emf_cbase_library_handle_t library_handle)
> ```
>
> - Effects: Unloads a library.
> - Failure: The function fails if `library_handle` is invalid.
> - Returns: Error on failure.

---

###### library_get_data_symbol

> ```c
> emf_cbase_library_data_symbol_result_t library_get_data_symbol(
>       emf_cbase_library_handle_t library_handle, 
>       const char* symbol_name)
> ```
>
> - Effects: Fetches a data symbol from a library.
> - Mandates: `symbol_name != NULL`.
> - Failure: The function fails if `library_handle` is invalid or library does not contain `symbol_name`.
> - Remarks: Some platforms may differentiate between a `function-pointer` and a `data-pointer`.
    See `library_get_function_symbol()` when fetching a function.
> - Returns: Symbol on success, error otherwise.

---

###### library_get_function_symbol

> ```c
> emf_cbase_library_fn_symbol_result_t library_get_function_symbol(
>       emf_cbase_library_handle_t library_handle, 
>       const char* symbol_name)
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

The module api is built on top of the library api and specifies the loading and unloading of modules. A module is a
collection libraries and resources, that together form an independent unit. Similarly to the library api, the loading
and unloading of modules is implemented by module loaders, which are associated to one module type.

#### Module loaders

Module loaders implement the actual loading and unloading of modules. Custom module loaders can be added by constructing
an `emf_cbase_module_loader_interface_t` and calling the `module_register_loader()` function.

#### Module types

The type of a module identifies a module loader capable of loading a specific module. A module type is represented by
an `emf_cbase_module_type_t`.

#### Module filesystem structure

```text
module/
├── module.json
└── ...
```

A module is a simple directory containing all the required resources. The internal structure of the module is defined by
the respective module loader. The only requirement is the existence of the module.json file at the root of the module.
This file is the module manifest and specifies that the directory is indeed a module.

##### Module manifest

The module manifest identifies a module and specifies how a module can be loaded. To allow for backwards (and forwards)
compatibility, the version of the manifest schema is saved in the manifest with the key `"schema"`.

###### Module manifest schema `"0"`

The version `"0"` introduces several required and optional fields:

- `"name"`: A `string` describing the name of the module. Has a maximum length of 32 ASCII characters. Is required.
- `"version"`: A `<version>` describing the required `emf-core-base` interface version. Is required.
- `"module_type"`: A `string` describing the `module type` of the module. Has a maximum length of 64 ASCII characters.
  Is required.
- `"module_version"`: A `string` describing the version of the module. Can be any string, but the usage of SemVer is
  encouraged. Has a maximum length of 32 ASCII characters. Is required.
- `"load_deps"`: An `array` of `<interface_descriptors>` describing the load dependencies.
- `"runtime_deps"`: An `array` of `<interface_descriptors>` describing the runtime dependencies.
- `"exports"`: An `array` of `<interface_descriptors>` describing the exportable interfaces.

The definition of the custom types can be found below:

- `<interface_descriptor>`: The descriptor of an interface.
  - `name`: A `string` describing the name of the interface. Has a maximum length of 32 ASCII characters. Is required.
  - `version`: A `<version>` describing the version of the interface. Is required.
  - `extensions`: An `array` of strings describing the extensions of the interface. Each extension has a maximum length
    of 32 ASCII characters.

- `<version>`: The `string` representation of a version. See the
  [versioning-specification RFC](0004-versioning-specification.md).
  
Example:

```json
{
    "schema": 0,
    "version": "1.0.0",
    "name": "jobs",
    "module_type": "emf::core_base::native",
    "module_version": "0.1.5-rc.7-a",
    "load_deps": [{
        "name": "memory",
        "version": "0.1.0"
    }],
    "runtime_deps": [{
        "name": "logging",
        "version": "1.0.0"
    }],
    "exports": [{
        "name": "jobs_interface",
        "version": "1.0.0-beta",
        "extensions": [
            "schedulers",
            "fibers"
        ]
    }]
}
```

#### Predefined module loaders

Some module loaders are always present and can not be removed at runtime.

##### Native module loader

The native module loader is built on top of the native library loader and is able to load modules consisting of native
libraries. It is reachable with the `EMF_CBASE_MODULE_LOADER_DEFAULT_HANDLE` handle. The same restrictions of the native
library loader apply to the native module loader. The native module loader requires the presence of a native module
manifest file named `native_module.json` at the root of the module.

##### Native module manifest

The native module manifest specifies which library implements the module. To allow for backwards (and forwards)
compatibility, the version of the manifest schema is saved in the manifest with the key `"schema"`.

###### Native module manifest schema `"0"`

The schema `"0"` specifies one field:

- `library_path`: A `string` describing the relative path to the library. Is required.

Example:

```json
{
    "schema": "0",
    "library_path": "./lib/jobs.so"
}
```

##### Native module interface

Once the module library has been loaded by the native library loader, the native module loader searches for a symbol
with the name `emf_cbase_native_module_interface` (see `EMF_CBASE_NATIVE_MODULE_INTERFACE_SYMBOL_NAME`) and the
type `emf_cbase_native_module_interface_t`.

#### Module api example

```c
/// `base_interf` is a structure containing the `emf-core-base` interface.
/// `base_handle` is a handle to the `emf-core-base` module.

base_interf->sys_lock_fn(base_handle);

const emf_cbase_os_path_char_t* module_path = "./jobs_module";
emf_cbase_module_handle_result_t module_handle_res = 
    base_interf->module_add_module_fn(base_handle, EMF_CBASE_MODULE_LOADER_DEFAULT_HANDLE, module_path);
if (module_handle_res.has_error) {
    base_interf->sys_panic_fn(base_handle, "Unable to load the `./jobs_module` module.");
}
emf_cbase_module_handle_t module_handle = module_handle_res.result;

emf_cbase_module_result_t result;
result = base_interf->module_load_fn(base_handle, module_handle);
if (result.has_error) {
    base_interf->sys_panic_fn(base_handle, "Unable to load the `./jobs_module` module.");
}

result = base_interf->module_initialize_fn(base_handle, module_handle);
if (result.has_error) {
    base_interf->sys_panic_fn(base_handle, "Unable to initialize the `./jobs_module` module.");
}

emf_cbase_interface_descriptor_t interface_descriptor = {
    .name = { .data = "jobs_interface", .length = 14 },
    .version = base_interf->version_new_short_fn(base_handle, 1, 0, 0),
    .extensions = { .data = NULL, length = 0 }
}
result = base_interf->module_export_interface_fn(base_handle, module_handle, &interface_descriptor);
if (result.has_error) {
    base_interf->sys_panic_fn(base_handle, "Unable to export `jobs_interface` from the `./jobs_module` module.");
}

base_interf->sys_unlock_fn(base_handle);
```

#### Module api overview

```c
// loader management
emf_cbase_module_loader_handle_result_t module_register_loader(
        const emf_cbase_module_loader_interface_t* loader_interface,
        const emf_cbase_module_type_t* module_type)
emf_cbase_module_result_t module_unregister_loader(
        emf_cbase_module_loader_handle_t loader_handle)
emf_cbase_module_loader_interface_result_t module_get_loader_interface(
        emf_cbase_module_loader_handle_t loader_handle)
emf_cbase_module_loader_handle_result_t module_get_loader_handle_from_type(
        const emf_cbase_module_type_t* module_type)
emf_cbase_module_loader_handle_result_t module_get_loader_handle_from_module(
        emf_cbase_module_handle_t module_handle)

// queries
size_t module_get_num_modules()
size_t module_get_num_loaders()
size_t module_get_num_exported_interfaces()
emf_cbase_bool_t module_module_exists(emf_cbase_module_handle_t module_handle)
emf_cbase_bool_t module_type_exists(const emf_cbase_module_type_t* module_type)
emf_cbase_bool_t module_exported_interface_exists(
        const emf_cbase_interface_descriptor_t* interface)
emf_cbase_module_size_result_t module_get_modules(
        emf_cbase_module_info_span_t* buffer)
emf_cbase_module_size_result_t module_get_module_types(
        emf_cbase_module_type_span_t* buffer)
emf_cbase_module_size_result_t module_get_exported_interfaces(
        emf_cbase_interface_descriptor_span_t* buffer)
emf_cbase_module_handle_result_t module_get_exported_interface_handle(
        const emf_cbase_interface_descriptor_t* interface)

// internal module management
emf_cbase_module_handle_t module_create_module_handle()
emf_cbase_module_result_t module_remove_module_handle(
        emf_cbase_module_handle_t module_handle)
emf_cbase_module_result_t module_link_module(
        emf_cbase_module_handle_t module_handle,
        emf_cbase_module_loader_handle_t loader_handle,
        emf_cbase_internal_module_handle_t loader_module_handle)
emf_cbase_internal_module_handle_result_t module_get_internal_module_handle(
        emf_cbase_module_handle_t module_handle)

// module management
emf_cbase_module_handle_result_t module_add_module(
        emf_cbase_module_loader_handle_t loader_handle,
        const emf_cbase_os_path_char_t* module_path)
emf_cbase_module_result_t module_remove_module(emf_cbase_module_handle_t module_handle)
emf_cbase_module_result_t module_load(emf_cbase_module_handle_t module_handle)
emf_cbase_module_result_t module_unload(emf_cbase_module_handle_t module_handle)
emf_cbase_module_result_t module_initialize(emf_cbase_module_handle_t module_handle)
emf_cbase_module_result_t module_terminate(emf_cbase_module_handle_t module_handle)
emf_cbase_module_result_t module_add_dependency(
        emf_cbase_module_handle_t module_handle,
        const emf_cbase_interface_descriptor_t* interface_descriptor)
emf_cbase_module_result_t module_remove_dependency(
        emf_cbase_module_handle_t module_handle,
        const emf_cbase_interface_descriptor_t* interface_descriptor)
emf_cbase_module_result_t module_export_interface(
        emf_cbase_module_handle_t module_handle,
        const emf_cbase_interface_descriptor_t* interface_descriptor)
emf_cbase_interface_descriptor_const_span_result_t module_get_load_dependencies(
        emf_cbase_module_handle_t module_handle)
emf_cbase_interface_descriptor_const_span_result_t module_get_runtime_dependencies(
        emf_cbase_module_handle_t module_handle)
emf_cbase_interface_descriptor_const_span_result_t module_get_exportable_interfaces(
        emf_cbase_module_handle_t module_handle)
emf_cbase_module_status_result_t module_fetch_status(emf_cbase_module_handle_t module_handle)
emf_cbase_os_path_char_result_t module_get_module_path(emf_cbase_module_handle_t module_handle)
emf_cbase_module_info_ptr_result_t module_get_module_info(emf_cbase_module_handle_t module_handle)
emf_cbase_module_interface_result_t module_get_interface(
        emf_cbase_module_handle_t module_handle,
        const emf_cbase_interface_descriptor_t* interface_descriptor)
```

#### Module api functions

##### Module api loader management

###### module_register_loader

> ```c
> emf_cbase_module_loader_handle_result_t module_register_loader(
>       const emf_cbase_module_loader_interface_t* loader_interface,
>       const emf_cbase_module_type_t* module_type)
> ```
>
> - Effects: Registers a new module loader.
    Module types starting with `__` are reserved for future use.
> - Mandates: `loader_interface != NULL && module_type != NULL`.
> - Failure: The function fails if `module_type` already exists.
> - Returns: Handle on success, error otherwise.

---

###### module_unregister_loader

> ```c
> emf_cbase_module_result_t module_unregister_loader(
>       emf_cbase_module_loader_handle_t loader_handle)
> ```
>
> - Effects: Unregisters an existing module loader.
    Unregistering a module loader also unloads the modules it loaded.
> - Mandates: `loader_interface != NULL && module_type != NULL`.
> - Failure: The function fails if `loader_handle` is invalid.
> - Returns: Error on failure.

---

###### module_get_loader_interface

> ```c
> emf_cbase_module_loader_interface_result_t module_get_loader_interface(
>       emf_cbase_module_loader_handle_t loader_handle)
> ```
>
> - Effects: Fetches the interface of a module loader.
> - Failure: The function fails if `loader_handle` is invalid.
> - Returns: Interface on success, error otherwise.

---

###### module_get_loader_handle_from_type

> ```c
> emf_cbase_module_loader_handle_result_t module_get_loader_handle_from_type(
>       const emf_cbase_module_type_t* module_type)
> ```
>
> - Effects: Fetches the handle of the loader associated with a module type.
> - Mandates: `module_type != NULL`.
> - Failure: The function fails if `module_type` does not exist.
> - Returns: Handle on success, error otherwise.

---

###### module_get_loader_handle_from_module

> ```c
> emf_cbase_module_loader_handle_result_t module_get_loader_handle_from_module(
>       emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Fetches the handle of the loader linked with the module handle.
> - Failure: The function fails if `module_handle` is invalid.
> - Returns: Handle on success, error otherwise.

##### Module api queries

###### module_get_num_modules

> ```c
> size_t module_get_num_modules()
> ```
>
> - Effects: Fetches the number of loaded modules.
> - Returns: Number of modules.

---

###### module_get_num_loaders

> ```c
> size_t module_get_num_loaders()
> ```
>
> - Effects: Fetches the number of loaders.
> - Returns: Number of module loaders.

---

###### module_get_num_exported_interfaces

> ```c
> size_t module_get_num_exported_interfaces()
> ```
>
> - Effects: Fetches the number of exported interfaces.
> - Returns: Number of exported interfaces.

---

###### module_module_exists

> ```c
> emf_cbase_bool_t module_module_exists(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Checks if a module exists.
> - Returns: `emf_cbase_bool_true` if it exists, `emf_cbase_bool_false` otherwise.

---

###### module_type_exists

> ```c
> emf_cbase_bool_t module_type_exists(const emf_cbase_module_type_t* module_type)
> ```
>
> - Effects: Checks if a module type exists.
> - Mandates: `module_type != NULL`.
> - Returns: `emf_cbase_bool_true` if it exists, `emf_cbase_bool_false` otherwise.

---

###### module_exported_interface_exists

> ```c
> emf_cbase_bool_t module_exported_interface_exists(
>       const emf_cbase_interface_descriptor_t* interface)
> ```
>
> - Effects: Checks whether an exported interface exists.
> - Mandates: `interface != NULL`.
> - Returns: `emf_cbase_bool_true` if it exists, `emf_cbase_bool_false` otherwise.

---

###### module_get_modules

> ```c
> emf_cbase_module_size_result_t module_get_modules(
>       emf_cbase_module_info_span_t* buffer)
> ```
>
> - Effects: Copies the available module info into a buffer.
> - Failure: Fails if `buffer->length < module_get_num_modules()`.
> - Mandates: `buffer != NULL && buffer->data != NULL`.
> - Returns: Number if written module info on success, error otherwise.

---

###### module_get_module_types

> ```c
> emf_cbase_module_size_result_t module_get_module_types(
>       emf_cbase_module_type_span_t* buffer)
> ```
>
> - Effects: Copies the available module types into a buffer.
> - Failure: Fails if `buffer->length < module_get_num_loaders()`.
> - Mandates: `buffer != NULL && buffer->data != NULL`.
> - Returns: Number if written module types on success, error otherwise.

---

###### module_get_exported_interfaces

> ```c
> emf_cbase_module_size_result_t module_get_exported_interfaces(
>       emf_cbase_interface_descriptor_span_t* buffer)
> ```
>
> - Effects: Copies the descriptors of the exported interfaces into a buffer.
> - Failure: Fails if `buffer->length < module_get_num_exported_interfaces()`.
> - Mandates: `buffer != NULL && buffer->data != NULL`.
> - Returns: Number if written descriptors on success, error otherwise.

---

###### module_get_exported_interface_handle

> ```c
> emf_cbase_module_handle_result_t module_get_exported_interface_handle(
>       const emf_cbase_interface_descriptor_t* interface)
> ```
>
> - Effects: Fetches the module handle of the exported interface.
> - Failure: Fails if `interface` does not exist.
> - Mandates: `interface != NULL`.
> - Returns: Module handle on success, error otherwise.

##### Module api internal module management

###### module_create_module_handle

> ```c
> emf_cbase_module_handle_t module_create_module_handle()
> ```
>
> - Effects: Creates a new unlinked module handle.
> - Remarks: The handle must be linked before use.
> - Returns: Module handle.

---

###### module_remove_module_handle

> ```c
> emf_cbase_module_result_t module_remove_module_handle(
>       emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Removes an existing module handle.
> - Failure: Fails if `module_handle` is invalid.
> - Remarks: Removing the handle does not unload the module.
> - Returns: Error on failure.

---

###### module_link_module

> ```c
> emf_cbase_module_result_t module_link_module(
>       emf_cbase_module_handle_t module_handle,
>       emf_cbase_module_loader_handle_t loader_handle,
>       emf_cbase_internal_module_handle_t loader_module_handle)
> ```
>
> - Effects: Links a module handle to an internal module handle.
> - Failure: Fails if `module_handle` or`loader_handle` are invalid.
> - Remarks: Incorrect usage can lead to dangling handles or use-after-free errors.
> - Returns: Error on failure.

---

###### module_get_internal_module_handle

> ```c
> emf_cbase_internal_module_handle_result_t module_get_internal_module_handle(
>       emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Fetches the internal handle linked with the module handle.
> - Failure: Fails if `module_handle` is invalid.
> - Returns: Internal handle on success, error otherwise.

##### Module api module management

###### module_add_module

> ```c
> emf_cbase_module_handle_result_t module_add_module(
>       emf_cbase_module_loader_handle_t loader_handle,
>       const emf_cbase_os_path_char_t* module_path)
> ```
>
> - Effects: Adds a new module.
> - Failure: Fails if `loader_handle` or `module_path` is invalid or
    the type of the module can not be loaded with the loader.
> - Returns: Module handle on success, error otherwise.

---

###### module_remove_module

> ```c
> emf_cbase_module_result_t module_remove_module(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Removes a module.
> - Failure: Fails if `module_handle` is invalid or the module is not in an unloaded state.
> - Returns: Error on failure.

---

###### module_load

> ```c
> emf_cbase_module_result_t module_load(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Loads a module.
> - Failure: Fails if `module_handle` is invalid, the load dependencies of the
    module are not exported or the module is not in an unloaded state.
> - Returns: Error on failure.

---

###### module_unload

> ```c
> emf_cbase_module_result_t module_unload(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Unloads a module.
> - Failure: Fails if `module_handle` is invalid or the module is in an unloaded or ready state.
> - Returns: Error on failure.

---

###### module_initialize

> ```c
> emf_cbase_module_result_t module_initialize(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Initializes a module.
> - Failure: Fails if `module_handle` is invalid, the runtime dependencies of the module are not
    exported or the module is not in a loaded state.
> - Returns: Error on failure.

---

###### module_terminate

> ```c
> emf_cbase_module_result_t module_terminate(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Terminates a module. Terminating a module also removes the interfaces it exported.
    The modules that depend on the module are terminated. If they list the module as a load dependency,
    they are also unloaded.
> - Failure: Fails if `module_handle` is invalid or the module is not in a ready state.
> - Returns: Error on failure.

---

###### module_add_dependency

> ```c
> emf_cbase_module_result_t module_add_dependency(
>       emf_cbase_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```
>
> - Effects: Registers a new runtime dependency of the module.
> - Failure: Fails if `module_handle` is invalid.
> - Mandates: `interface_descriptor != NULL`.
> - Returns: Error on failure.

---

###### module_remove_dependency

> ```c
> emf_cbase_module_result_t module_remove_dependency(
>       emf_cbase_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```
>
> - Effects: Removes an existing runtime dependency from the module.
> - Failure: Fails if `module_handle` is invalid.
> - Mandates: `interface_descriptor != NULL`.
> - Returns: Error on failure.

---

###### module_export_interface

> ```c
> emf_cbase_module_result_t module_export_interface(
>       emf_cbase_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```
>
> - Effects: Exports an interface of a module.
> - Failure: Fails if `module_handle` is invalid, `interface_descriptor` is already exported,
    `interface_descriptor` is not contained in the module or the module is not yet initialized.
> - Mandates: `interface_descriptor != NULL`.
> - Returns: Error on failure.

---

###### module_get_load_dependencies

> ```c
> emf_cbase_interface_descriptor_const_span_result_t module_get_load_dependencies(
>       emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Fetches the load dependencies of a module.
> - Failure: Fails if `module_handle` is invalid.
> - Returns: Load dependencies on success, error otherwise.

---

###### module_get_runtime_dependencies

> ```c
> emf_cbase_interface_descriptor_const_span_result_t module_get_runtime_dependencies(
>       emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Fetches the runtime dependencies of a module.
> - Failure: Fails if `module_handle` is invalid or the module is not yet loaded.
> - Returns: Runtime dependencies on success, error otherwise.

---

###### module_get_exportable_interfaces

> ```c
> emf_cbase_interface_descriptor_const_span_result_t module_get_exportable_interfaces(
>       emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Fetches the exportable interfaces of a module.
> - Failure: Fails if `module_handle` is invalid or the module is not yet loaded.
> - Returns: Exportable interfaces on success, error otherwise.

---

###### module_fetch_status

> ```c
> emf_cbase_module_status_result_t module_fetch_status(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Fetches the load status of a module.
> - Failure: Fails if `module_handle` is invalid.
> - Returns: Module status on success, error otherwise.

---

###### module_get_module_path

> ```c
> emf_cbase_os_path_char_result_t module_get_module_path(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Fetches the path a module was loaded from.
> - Failure: Fails if `module_handle` is invalid or the module is not yet loaded.
> - Returns: Module path on success, error otherwise.

---

###### module_get_module_info

> ```c
> emf_cbase_module_info_ptr_result_t module_get_module_info(emf_cbase_module_handle_t module_handle)
> ```
>
> - Effects: Fetches the module info from a module.
> - Failure: Fails if `module_handle` is invalid or the module is not yet loaded.
> - Returns: Module info on success, error otherwise.

---

###### module_get_interface

> ```c
> emf_cbase_module_interface_result_t module_get_interface(
>       emf_cbase_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```
>
> - Effects: Fetches an interface from a module.
> - Failure: Fails if `module_handle` is invalid, the module is not in a ready state
    or the interface is not contained in the module.
> - Mandates: `interface_descriptor != NULL`.
> - Returns: Interface on success, error otherwise.

### Version api

[version-api]: #version-api

The version api implements the versioning scheme specified in
the [versioning-specification RFC](0004-versioning-specification.md).

#### Version api example

```c
/// `base_interf` is a structure containing the `emf-core-base` interface.
/// `base_handle` is a handle to the `emf-core-base` module.

emf_cbase_version_t v1 = base_interf->version_new_short_fn(base_handle, 1, 2, 3);

const char* v2_string = "1.2.3-beta.5+54845652";
emf_cbase_version_const_string_buffer_t v2_string_buff = {
    .data = v2_string, 
    .length = strlen(v2_string)
};
emf_cbase_version_result_t v2_res = 
        base_interf->version_from_string_fn(base_handle, &v2_string_buff);
if (v2_res.has_error) {
    base_interf->sys_lock_fn(base_handle);
    base_interf->sys_panic_fn(base_handle, "Could not construct version from string.");
    base_interf->sys_unlock_fn(base_handle);
}
emf_cbase_version_t v2 = v2_res.result;

if (base_interf->version_compare_weak_fn(base_handle, &v1, &v2) != 0) {
    base_interf->sys_lock_fn(base_handle);
    base_interf->sys_panic_fn(base_handle, "Should not happen.");
    base_interf->sys_unlock_fn(base_handle);
}
```

#### Version api overview

```c
// construction
emf_cbase_version_t version_new_short(
        int32_t major, 
        int32_t minor,
        int32_t patch)
emf_cbase_version_t version_new_long(
        int32_t major, 
        int32_t minor,
        int32_t patch, 
        emf_cbase_version_release_t release_type, 
        int8_t release_number)
emf_cbase_version_t version_new_full(
        int32_t major, 
        int32_t minor,
        int32_t patch, 
        emf_cbase_version_release_t release_type, 
        int8_t release_number, 
        int64_t build)
emf_cbase_version_result_t version_from_string(const emf_cbase_version_const_string_buffer_t* version_string)

// strings
size_t version_string_length_short(const emf_cbase_version_t* version)
size_t version_string_length_long(const emf_cbase_version_t* version)
size_t version_string_length_full(const emf_cbase_version_t* version)
emf_cbase_version_size_result_t version_as_string_short(
        const emf_cbase_version_t* version,
        emf_cbase_version_string_buffer_t* buffer)
emf_cbase_version_size_result_t version_as_string_long(
        const emf_cbase_version_t* version,
        emf_cbase_version_string_buffer_t* buffer)
emf_cbase_version_size_result_t version_as_string_full(
        const emf_cbase_version_t* version,
        emf_cbase_version_string_buffer_t* buffer)
emf_cbase_bool_t version_string_is_valid(const emf_cbase_version_const_string_buffer_t* version_string)

// comparisons
int32_t version_compare(const emf_cbase_version_t* lhs, const emf_cbase_version_t* rhs)
int32_t version_compare_weak(const emf_cbase_version_t* lhs, const emf_cbase_version_t* rhs)
int32_t version_compare_strong(const emf_cbase_version_t* lhs, const emf_cbase_version_t* rhs)
emf_cbase_bool_t version_is_compatible(const emf_cbase_version_t* lhs, const emf_cbase_version_t* rhs)
```

#### Version api functions

##### Version api construction

###### version_new_short

> ```c
> emf_cbase_version_t version_new_short(
>       int32_t major, 
>       int32_t minor,
>       int32_t patch)
> ```
>
> - Effects: Constructs a new version.
    Constructs a new version with `major`, `minor` and `patch` and sets the rest to `0`.
> - Returns: Constructed version.

---

###### version_new_long

> ```c
> emf_cbase_version_t version_new_long(
>       int32_t major, 
>       int32_t minor,
>       int32_t patch, 
>       emf_cbase_version_release_t release_type, 
>       int8_t release_number)
> ```
>
> - Effects: Constructs a new version.
    Constructs a new version with `major`, `minor`, `patch`, `release_type` and
    `release_number` and sets the rest to `0`.
> - Returns: Constructed version.

---

###### version_new_full

> ```c
> emf_cbase_version_t version_new_full(
>       int32_t major, 
>       int32_t minor,
>       int32_t patch, 
>       emf_cbase_version_release_t release_type, 
>       int8_t release_number, 
>       int64_t build)
> ```
>
> - Effects: Constructs a new version.
    Constructs a new version with `major`, `minor`, `patch`, `release_type`, `release_number` and `build`.
> - Returns: Constructed version.

---

###### version_from_string

> ```c
> emf_cbase_version_result_t version_from_string(const emf_cbase_version_const_string_buffer_t* version_string)
> ```
>
> - Effects: Constructs a version from a string.
> - Mandates: `version_string != NULL && version_string->data != NULL`.
> - Failure: Fails if `version_string_is_valid(version_string) == emf_cbase_bool_false`.
> - Returns: Constructed version.

##### Version api strings

###### version_string_length_short

> ```c
> size_t version_string_length_short(const emf_cbase_version_t* version)
> ```
>
> - Effects: Computes the length of the short version string.
> - Mandates: `version != NULL`.
> - Returns: Length of the string.

---

###### version_string_length_long

> ```c
> size_t version_string_length_long(const emf_cbase_version_t* version)
> ```
>
> - Effects: Computes the length of the long version string.
> - Mandates: `version != NULL`.
> - Returns: Length of the string.

---

###### version_string_length_full

> ```c
> size_t version_string_length_full(const emf_cbase_version_t* version)
> ```
>
> - Effects: Computes the length of the full version string.
> - Mandates: `version != NULL`.
> - Returns: Length of the string.

---

###### version_as_string_short

> ```c
> emf_cbase_version_size_result_t version_as_string_short(
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer)
> ```
>
> - Effects: Represents the version as a short string.
> - Mandates: `version != NULL && buffer != NULL && buffer->data != NULL`.
> - Failure: This function fails if `buffer->length < version_string_length_short(version)`.
> - Returns: Number of written characters on success, error otherwise.

---

###### version_as_string_long

> ```c
> emf_cbase_version_size_result_t version_as_string_long(
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer)
> ```
>
> - Effects: Represents the version as a long string.
> - Mandates: `version != NULL && buffer != NULL && buffer->data != NULL`.
> - Failure: This function fails if `buffer->length < version_string_length_long(version)`.
> - Returns: Number of written characters on success, error otherwise.

---

###### version_as_string_full

> ```c
> emf_cbase_version_size_result_t version_as_string_full(
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer)
> ```
>
> - Effects: Represents the version as a full string.
> - Mandates: `version != NULL && buffer != NULL && buffer->data != NULL`.
> - Failure: This function fails if `buffer->length < version_string_length_full(version)`.
> - Returns: Number of written characters on success, error otherwise.

---

###### version_string_is_valid

> ```c
> emf_cbase_bool_t version_string_is_valid(const emf_cbase_version_const_string_buffer_t* version_string)
> ```
>
> - Effects: Checks whether the version string is valid.
> - Mandates: `version_string != NULL && version_string->data != NULL`.
> - Returns: `emf_cbase_bool_true` if the string is valid, `emf_cbase_bool_false` otherwise.

## Reference

[reference]: #reference

### Conventions

For the sake of brevity, we will introduce the following macros:

#### BASE_FN_T

> ```c
> #define BASE_FN_T(NAME, RET_T, ...) \  
>   typedef RET_T(*NAME)(emf_cbase_t*, __VA_ARGS__);
> ```
>
> - Description: A pointer to an interface function.

---

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

#### FN_T

> ```c
> #define FN_T(NAME, RET_T, ...) \  
>   typedef RET_T(*NAME)(__VA_ARGS__);
> ```
>
> - Description: A pointer to a function.

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
> - Description: Character used by the OS to represent a path.

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
> - Description: An enum describing a Boolean value.
>
> | Name                     | Value | Description  |
> | ------------------------ | ----- | ------------ |
> | **emf_cbase_bool_false** | `0`   | False value. |
> | **emf_cbase_bool_true**  | `1`   | True value.  |

---

#### emf_cbase_fn_ptr_id_t

> ```c
> typedef enum emf_cbase_fn_ptr_id_t : int32_t {
>       emf_cbase_fn_ptr_id_sys_shutdown = 1,
>       emf_cbase_fn_ptr_id_sys_panic = 2,
>       emf_cbase_fn_ptr_id_sys_has_function = 3,
>       emf_cbase_fn_ptr_id_sys_get_function = 4,
>       emf_cbase_fn_ptr_id_sys_lock = 5,
>       emf_cbase_fn_ptr_id_sys_try_lock = 6,
>       emf_cbase_fn_ptr_id_sys_unlock = 7,
>       emf_cbase_fn_ptr_id_sys_get_sync_handler = 8,
>       emf_cbase_fn_ptr_id_sys_set_sync_handler = 9,
> 
>       emf_cbase_fn_ptr_id_version_new_short = 101,
>       emf_cbase_fn_ptr_id_version_new_long = 102,
>       emf_cbase_fn_ptr_id_version_new_full = 103,
>       emf_cbase_fn_ptr_id_version_from_string = 104,
>       emf_cbase_fn_ptr_id_version_string_length_short = 105,
>       emf_cbase_fn_ptr_id_version_string_length_long = 106,
>       emf_cbase_fn_ptr_id_version_string_length_full = 107,
>       emf_cbase_fn_ptr_id_version_as_string_short = 108,
>       emf_cbase_fn_ptr_id_version_as_string_long = 109,
>       emf_cbase_fn_ptr_id_version_as_string_full = 110,
>       emf_cbase_fn_ptr_id_version_string_is_valid = 111,
>       emf_cbase_fn_ptr_id_version_compare = 112,
>       emf_cbase_fn_ptr_id_version_compare_weak = 113,
>       emf_cbase_fn_ptr_id_version_compare_strong = 114,
>       emf_cbase_fn_ptr_id_version_is_compatible = 115,
> 
>       emf_cbase_fn_ptr_id_library_register_loader = 201,
>       emf_cbase_fn_ptr_id_library_unregister_loader = 202,
>       emf_cbase_fn_ptr_id_library_get_loader_interface = 203,
>       emf_cbase_fn_ptr_id_library_get_loader_handle_from_type = 204,
>       emf_cbase_fn_ptr_id_library_get_loader_handle_from_library = 205,
>       emf_cbase_fn_ptr_id_library_get_num_loaders = 206,
>       emf_cbase_fn_ptr_id_library_library_exists = 207,
>       emf_cbase_fn_ptr_id_library_type_exists = 208,
>       emf_cbase_fn_ptr_id_library_get_library_types = 209,
>       emf_cbase_fn_ptr_id_library_create_library_handle = 210,
>       emf_cbase_fn_ptr_id_library_remove_library_handle = 211,
>       emf_cbase_fn_ptr_id_library_link_library = 212,
>       emf_cbase_fn_ptr_id_library_get_internal_library_handle = 213,
>       emf_cbase_fn_ptr_id_library_load = 214,
>       emf_cbase_fn_ptr_id_library_unload = 215,
>       emf_cbase_fn_ptr_id_library_get_data_symbol = 216,
>       emf_cbase_fn_ptr_id_library_get_function_symbol = 217,
> 
>       module_register_loader = 301,
>       module_unregister_loader = 302,
>       module_get_loader_interface = 303,
>       module_get_loader_handle_from_type = 304,
>       module_get_loader_handle_from_module = 305,
>       module_get_num_modules = 306,
>       module_get_num_loaders = 307,
>       module_get_num_exported_interfaces = 308,
>       module_module_exists = 309,
>       module_type_exists = 310,
>       module_exported_interface_exists = 311,
>       module_get_modules = 312,
>       module_get_module_types = 313,
>       module_get_exported_interfaces = 314,
>       module_get_exported_interface_handle = 315,
>       module_create_module_handle = 316,
>       module_remove_module_handle = 317,
>       module_link_module = 318,
>       module_get_internal_module_handle = 319,
>       module_add_module = 320,
>       module_remove_module = 321,
>       module_load = 322,
>       module_unload = 323,
>       module_initialize = 324,
>       module_terminate = 325,
>       module_add_dependency = 326,
>       module_remove_dependency = 327,
>       module_export_interface = 328,
>       module_get_load_dependencies = 329,
>       module_get_runtime_dependencies = 330,
>       module_get_exportable_interfaces = 331,
>       module_fetch_status = 332,
>       module_get_module_path = 333,
>       module_get_module_info = 334,
>       module_get_interface = 335,
> }
> ```
> 
> - Description: Id's of the exported functions.
    The values `0-1000` are reserved for future use.

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

#### emf_cbase_interface_t

> ```c
> typedef struct emf_cbase_interface_t {
>       emf_cbase_version_t interface_version;
>       emf_cbase_t* cbase;
> 
>       emf_cbase_sys_shutdown_fn_t sys_shutdown_fn;
>       emf_cbase_sys_panic_fn_t sys_panic_fn;
>       emf_cbase_sys_has_function_fn_t sys_has_function_fn;
>       emf_cbase_sys_get_function_fn_t sys_get_function_fn;
>       emf_cbase_sys_lock_fn_t sys_lock_fn;
>       emf_cbase_sys_try_lock_fn_t sys_try_lock_fn;
>       emf_cbase_sys_unlock_fn_t sys_unlock_fn;
>       emf_cbase_sys_get_sync_handler_fn_t sys_get_sync_handler_fn;
>       emf_cbase_sys_set_sync_handler_fn_t sys_set_sync_handler_fn;
>
>       emf_cbase_version_new_short_fn_t version_new_short_fn;
>       emf_cbase_version_new_long_fn_t version_new_long_fn;
>       emf_cbase_version_new_full_fn_t version_new_full_fn;
>       emf_cbase_version_from_string_fn_t version_from_string_fn;
>       emf_cbase_version_string_length_short_fn_t version_string_length_short_fn;
>       emf_cbase_version_string_length_long_fn_t version_string_length_long_fn;
>       emf_cbase_version_string_length_full_fn_t version_string_length_full_fn;
>       emf_cbase_version_as_string_short_fn_t version_as_string_short_fn;
>       emf_cbase_version_as_string_long_fn_t version_as_string_long_fn;
>       emf_cbase_version_as_string_full_fn_t version_as_string_full_fn;
>       emf_cbase_version_string_is_valid_fn_t version_string_is_valid_fn;
>       emf_cbase_version_compare_fn_t version_compare_fn;
>       emf_cbase_version_compare_weak_fn_t version_compare_weak_fn;
>       emf_cbase_version_compare_strong_fn_t version_compare_strong_fn;
>       emf_cbase_version_is_compatible_fn_t version_is_compatible_fn;
>
>       emf_cbase_library_register_loader_fn_t library_register_loader_fn;
>       emf_cbase_library_unregister_loader_fn_t library_unregister_loader_fn;
>       emf_cbase_library_get_loader_interface_fn_t library_get_loader_interface_fn;
>       emf_cbase_library_get_loader_handle_from_type_fn_t library_get_loader_handle_from_type_fn;
>       emf_cbase_library_get_loader_handle_from_library_fn_t library_get_loader_handle_from_library_fn;
>       emf_cbase_library_get_num_loaders_fn_t library_get_num_loaders_fn;
>       emf_cbase_library_library_exists_fn_t library_library_exists_fn;
>       emf_cbase_library_type_exists_fn_t library_type_exists_fn;
>       emf_cbase_library_get_library_types_fn_t library_get_library_types_fn;
>       emf_cbase_library_create_library_handle_fn_t library_create_library_handle_fn;
>       emf_cbase_library_remove_library_handle_fn_t library_remove_library_handle_fn;
>       emf_cbase_library_link_library_fn_t library_link_library_fn;
>       emf_cbase_library_get_internal_library_handle_fn_t library_get_internal_library_handle_fn;
>       emf_cbase_library_load_fn_t library_load_fn;
>       emf_cbase_library_unload_fn_t library_unload_fn;
>       emf_cbase_library_get_data_symbol_fn_t library_get_data_symbol_fn;
>       emf_cbase_library_get_function_symbol_fn_t library_get_function_symbol_fn;
>
>       emf_cbase_module_register_loader_fn_t module_register_loader_fn;
>       emf_cbase_module_unregister_loader_fn_t module_unregister_loader_fn;
>       emf_cbase_module_get_loader_interface_fn_t module_get_loader_interface_fn;
>       emf_cbase_module_get_loader_handle_from_type_fn_t module_get_loader_handle_from_type_fn;
>       emf_cbase_module_get_loader_handle_from_module_fn_t module_get_loader_handle_from_module_fn;
>       emf_cbase_module_get_num_modules_fn_t module_get_num_modules_fn;
>       emf_cbase_module_get_num_loaders_fn_t module_get_num_loaders_fn;
>       emf_cbase_module_get_num_exported_interfaces_fn_t module_get_num_exported_interfaces_fn;
>       emf_cbase_module_module_exists_fn_t module_module_exists_fn;
>       emf_cbase_module_type_exists_fn_t module_type_exists_fn;
>       emf_cbase_module_exported_interface_exists_fn_t module_exported_interface_exists_fn;
>       emf_cbase_module_get_modules_fn_t module_get_modules_fn;
>       emf_cbase_module_get_module_types_fn_t module_get_module_types_fn;
>       emf_cbase_module_get_exported_interfaces_fn_t module_get_exported_interfaces_fn;
>       emf_cbase_module_get_exported_interface_handle_fn_t module_get_exported_interface_handle_fn;
>       emf_cbase_module_create_module_handle_fn_t module_create_module_handle_fn;
>       emf_cbase_module_remove_module_handle_fn_t module_remove_module_handle_fn;
>       emf_cbase_module_link_module_fn_t module_link_module_fn;
>       emf_cbase_module_get_internal_module_handle_fn_t module_get_internal_module_handle_fn;
>       emf_cbase_module_add_module_fn_t module_add_module_fn;
>       emf_cbase_module_remove_module_fn_t module_remove_module_fn;
>       emf_cbase_module_load_fn_t module_load_fn;
>       emf_cbase_module_unload_fn_t module_unload_fn;
>       emf_cbase_module_initialize_fn_t module_initialize_fn;
>       emf_cbase_module_terminate_fn_t module_terminate_fn;
>       emf_cbase_module_add_dependency_fn_t module_add_dependency_fn;
>       emf_cbase_module_remove_dependency_fn_t module_remove_dependency_fn;
>       emf_cbase_module_export_interface_fn_t module_export_interface_fn;
>       emf_cbase_module_get_load_dependencies_fn_t module_get_load_dependencies_fn;
>       emf_cbase_module_get_runtime_dependencies_fn_t module_get_runtime_dependencies_fn;
>       emf_cbase_module_get_exportable_interfaces_fn_t module_get_exportable_interfaces_fn;
>       emf_cbase_module_fetch_status_fn_t module_fetch_status_fn;
>       emf_cbase_module_get_module_path_fn_t module_get_module_path_fn;
>       emf_cbase_module_get_module_info_fn_t module_get_module_info_fn;
>       emf_cbase_module_get_interface_fn_t module_get_interface_fn;
> };
> ```
>
> - Description: `emf-core-base` interface.

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
> RESULT_T(emf_cbase_library_loader_interface_result_t,
>     const emf_cbase_library_loader_interface_t*, emf_cbase_library_error_t)
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
> typedef struct emf_cbase_internal_library_handle_t {
>     intptr_t id;
> } emf_cbase_internal_library_handle_t;
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
> - Description: Span of module info.

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
>     emf_cbase_module_loader_interface_load_fn_t load_fn;
>     emf_cbase_module_loader_interface_unload_fn_t unload_fn;
>     emf_cbase_module_loader_interface_initialize_fn_t initialize_fn;
>     emf_cbase_module_loader_interface_terminate_fn_t terminate_fn;
>     emf_cbase_module_loader_interface_fetch_status_fn_t fetch_status_fn;
>     emf_cbase_module_loader_interface_get_interface_fn_t get_interface_fn;
>     emf_cbase_module_loader_interface_get_module_info_fn_t get_module_info_fn;
>     emf_cbase_module_loader_interface_get_module_path_fn_t get_module_path_fn;
>     emf_cbase_module_loader_interface_get_load_dependencies_fn_t get_load_dependencies_fn;
>     emf_cbase_module_loader_interface_get_runtime_dependencies_fn_t get_runtime_dependencies_fn;
>     emf_cbase_module_loader_interface_get_exportable_interfaces_fn_t get_exportable_interfaces_fn;
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
> typedef struct emf_cbase_internal_module_handle_t {
>     intptr_t id;
> } emf_cbase_internal_module_handle_t;
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
>     emf_cbase_native_module_interface_get_interface_fn_t get_interface_fn;
>     emf_cbase_native_module_interface_get_module_info_fn_t get_module_info_fn;
>     emf_cbase_native_module_interface_get_load_dependencies_fn_t get_load_dependencies_fn;
>     emf_cbase_native_module_interface_get_runtime_dependencies_fn_t get_runtime_dependencies_fn;
>     emf_cbase_native_module_interface_get_exportable_interfaces_fn_t get_exportable_interfaces_fn;
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
> - Description: Opaque structure representing a synchronization handler.

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
> - Description: Interface of a synchronization handler.

### Types

[types]: #types

#### emf_cbase_fn_t

> ```c
> FN_T(emf_cbase_fn_t, void, void)
> ```

---

#### emf_cbase_library_create_library_handle_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_create_library_handle_fn_t, emf_cbase_library_handle_t)
> ```

---

#### emf_cbase_library_get_data_symbol_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_get_data_symbol_fn_t, emf_cbase_library_data_symbol_result_t, 
>       emf_cbase_library_handle_t library_handle, const char* symbol_name)
> ```

---

#### emf_cbase_library_get_function_symbol_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_get_function_symbol_fn_t, emf_cbase_library_fn_symbol_result_t, 
>       emf_cbase_library_handle_t library_handle, const char* symbol_name)
> ```

---

#### emf_cbase_library_get_internal_library_handle_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_get_internal_library_handle_fn_t, 
>       emf_cbase_internal_library_handle_result_t, emf_cbase_library_handle_t library_handle)
> ```

---

#### emf_cbase_library_get_library_types_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_get_library_types_fn_t, 
>       emf_cbase_library_size_result_t, emf_cbase_library_type_span_t* buffer)
> ```

---

#### emf_cbase_library_get_loader_handle_from_library_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_get_loader_handle_from_library_fn_t,
>       emf_cbase_library_loader_handle_result_t, emf_cbase_library_handle_t library_handle)
> ```

---

#### emf_cbase_library_get_loader_handle_from_type_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_get_loader_handle_from_type_fn_t,
>       emf_cbase_library_loader_handle_result_t, const emf_cbase_library_type_t* library_type)
> ```

---

#### emf_cbase_library_get_loader_interface_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_get_loader_interface_fn_t,
>       emf_cbase_library_loader_interface_result_t, emf_cbase_library_loader_handle_t loader_handle)
> ```

---

#### emf_cbase_library_get_num_loaders_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_get_num_loaders_fn_t, size_t, void)
> ```

---

#### emf_cbase_library_library_exists_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_library_exists_fn_t, emf_cbase_bool_t, emf_cbase_library_handle_t library_handle)
> ```

---

#### emf_cbase_library_link_library_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_link_library_fn_t,
>       emf_cbase_library_result_t,
>       emf_cbase_library_handle_t library_handle, 
>       emf_cbase_library_loader_handle_t loader_handle,
>       emf_cbase_internal_library_handle_t internal_handle)
> ```

---

#### emf_cbase_library_load_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_load_fn_t, 
>       emf_cbase_library_handle_result_t,
>       emf_cbase_library_loader_handle_t loader_handle, 
>       const emf_cbase_os_path_char_t* library_path)
> ```

---

#### emf_cbase_library_loader_interface_get_data_symbol_fn_t

> ```c
> FN_T(emf_cbase_library_loader_interface_get_data_symbol_fn_t,
>       emf_cbase_library_data_symbol_result_t,
>       emf_cbase_library_loader_t* library_loader,
>       emf_cbase_internal_library_handle_t library_handle,
>       const char* symbol_name)
> ```

---

#### emf_cbase_library_loader_interface_get_function_symbol_fn_t

> ```c
> FN_T(emf_cbase_library_loader_interface_get_function_symbol_fn_t,
>       emf_cbase_library_fn_symbol_result_t,
>       emf_cbase_library_loader_t* library_loader,
>       emf_cbase_internal_library_handle_t library_handle,
>       const char* symbol_name)
> ```

---

#### emf_cbase_library_loader_interface_get_internal_interface_fn_t

> ```c
> FN_T(emf_cbase_library_loader_interface_get_internal_interface_fn_t,
>       const void*,
>        emf_cbase_library_loader_t* library_loader)
> ```

---

#### emf_cbase_library_loader_interface_load_fn_t

> ```c
> FN_T(emf_cbase_library_loader_interface_load_fn_t,
>       emf_cbase_internal_library_handle_result_t,
>       emf_cbase_library_loader_t* library_loader,
>       const emf_cbase_os_path_char_t* library_path)
> ```

---

#### emf_cbase_library_loader_interface_unload_fn_t

> ```c
> FN_T(emf_cbase_library_loader_interface_unload_fn_t, 
>       emf_cbase_library_result_t,
>       emf_cbase_library_loader_t* library_loader,
>       emf_cbase_internal_library_handle_t library_handle)
> ```

---

#### emf_cbase_library_register_loader_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_register_loader_fn_t,
>       emf_cbase_library_loader_handle_result_t,
>       const emf_cbase_library_loader_interface_t* loader_interface, 
>       const emf_cbase_library_type_t* library_type)
> ```

---

#### emf_cbase_library_remove_library_handle_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_remove_library_handle_fn_t,
>       emf_cbase_library_result_t,
>       emf_cbase_library_handle_t library_handle)
> ```

---

#### emf_cbase_library_type_exists_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_type_exists_fn_t, 
>       emf_cbase_bool_t,
>       emf_cbase_library_handle_t library_handle)
> ```

---

#### emf_cbase_library_unload_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_unload_fn_t,
>       emf_cbase_library_result_t,
>       emf_cbase_library_handle_t library_handle)
> ```

---

#### emf_cbase_library_unregister_loader_fn_t

> ```c
> BASE_FN_T(emf_cbase_library_unregister_loader_fn_t,
>       emf_cbase_library_result_t,
>       emf_cbase_library_loader_handle_t loader_handle)
> ```

---

#### emf_cbase_module_add_dependency_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_add_dependency_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```

---

#### emf_cbase_module_add_module_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_add_module_fn_t,
>       emf_cbase_module_handle_result_t,
>       emf_cbase_module_loader_handle_t loader_handle,
>       const emf_cbase_os_path_char_t* module_path)
> ```

---

#### emf_cbase_module_create_module_handle_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_create_module_handle_fn_t,
>       emf_cbase_module_handle_t,
>       void)
> ```

---

#### emf_cbase_module_export_interface_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_export_interface_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```

---

#### emf_cbase_module_exported_interface_exists_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_exported_interface_exists_fn_t,
>       emf_cbase_bool_t,
>       const emf_cbase_interface_descriptor_t* interface)
> ```

---

#### emf_cbase_module_fetch_status_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_fetch_status_fn_t,
>       emf_cbase_module_status_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_get_exportable_interfaces_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_exportable_interfaces_fn_t,
>       emf_cbase_interface_descriptor_const_span_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_get_exported_interface_handle_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_exported_interface_handle_fn_t,
>       emf_cbase_module_handle_result_t,
>       const emf_cbase_interface_descriptor_t* interface)
> ```

---

#### emf_cbase_module_get_exported_interfaces_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_exported_interfaces_fn_t,
>       emf_cbase_module_size_result_t,
>       emf_cbase_interface_descriptor_span_t* buffer)
> ```

---

#### emf_cbase_module_get_interface_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_interface_fn_t,
>       emf_cbase_module_interface_result_t,
>       emf_cbase_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```

---

#### emf_cbase_module_get_internal_module_handle_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_internal_module_handle_fn_t,
>       emf_cbase_internal_module_handle_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_get_load_dependencies_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_load_dependencies_fn_t,
>       emf_cbase_interface_descriptor_const_span_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_get_loader_handle_from_module_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_loader_handle_from_module_fn_t,
>       emf_cbase_module_loader_handle_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_get_loader_handle_from_type_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_loader_handle_from_type_fn_t,
>       emf_cbase_module_loader_handle_result_t,
>       const emf_cbase_module_type_t* module_type)
> ```

---

#### emf_cbase_module_get_loader_interface_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_loader_interface_fn_t,
>       emf_cbase_module_loader_interface_result_t,
>       emf_cbase_module_loader_handle_t loader_handle)
> ```

---

#### emf_cbase_module_get_module_info_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_module_info_fn_t,
>       emf_cbase_module_info_ptr_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_get_module_path_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_module_path_fn_t,
>       emf_cbase_os_path_char_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_get_module_types_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_module_types_fn_t,
>       emf_cbase_module_size_result_t,
>       emf_cbase_module_type_span_t* buffer)
> ```

---

#### emf_cbase_module_get_modules_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_modules_fn_t,
>       emf_cbase_module_size_result_t,
>       emf_cbase_module_info_span_t* buffer)
> ```

---

#### emf_cbase_module_get_num_exported_interfaces_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_num_exported_interfaces_fn_t, size_t, void)
> ```

---

#### emf_cbase_module_get_num_loaders_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_num_loaders_fn_t, size_t, void)
> ```

---

#### emf_cbase_module_get_num_modules_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_num_modules_fn_t, size_t, void)
> ```

---

#### emf_cbase_module_get_runtime_dependencies_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_get_runtime_dependencies_fn_t,
>       emf_cbase_interface_descriptor_const_span_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_initialize_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_initialize_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_link_module_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_link_module_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle,
>       emf_cbase_module_loader_handle_t loader_handle,
>       emf_cbase_internal_module_handle_t loader_module_handle)
> ```

---

#### emf_cbase_module_load_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_load_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_add_module_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_add_module_fn_t,
>       emf_cbase_internal_module_handle_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       const emf_cbase_os_path_char_t* module_path)
> ```

---

#### emf_cbase_module_loader_interface_fetch_status_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_fetch_status_fn_t,
>       emf_cbase_module_status_result_t,
>       emf_cbase_module_loader_t* module_loader,
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_get_exportable_interfaces_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_get_exportable_interfaces_fn_t,
>       emf_cbase_interface_descriptor_const_span_result_t,
>       emf_cbase_module_loader_t* module_loader,
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_get_interface_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_get_interface_fn_t,
>       emf_cbase_module_interface_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```

---

#### emf_cbase_module_loader_interface_get_internal_interface_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_get_internal_interface_fn_t,
>       const void*,
>       emf_cbase_module_loader_t* module_loader)
> ```

---

#### emf_cbase_module_loader_interface_get_load_dependencies_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_get_load_dependencies_fn_t,
>       emf_cbase_interface_descriptor_const_span_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_get_module_info_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_get_module_info_fn_t,
>       emf_cbase_module_info_ptr_result_t,
>       emf_cbase_module_loader_t* module_loader,
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_get_module_path_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_get_module_path_fn_t,
>       emf_cbase_os_path_char_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_get_runtime_dependencies_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_get_runtime_dependencies_fn_t,
>       emf_cbase_interface_descriptor_const_span_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_initialize_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_initialize_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_load_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_load_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_remove_module_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_remove_module_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_terminate_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_terminate_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_loader_interface_unload_fn_t

> ```c
> FN_T(emf_cbase_module_loader_interface_unload_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_loader_t* module_loader, 
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_module_exists_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_module_exists_fn_t,
>       emf_cbase_bool_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_register_loader_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_register_loader_fn_t,
>       emf_cbase_module_loader_handle_result_t,
>       const emf_cbase_module_loader_interface_t* loader_interface,
>       const emf_cbase_module_type_t* module_type)
> ```

---

#### emf_cbase_module_remove_dependency_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_remove_dependency_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```

---

#### emf_cbase_module_remove_module_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_remove_module_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_remove_module_handle_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_remove_module_handle_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_terminate_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_terminate_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_type_exists_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_type_exists_fn_t,
>       emf_cbase_bool_t,
>       const emf_cbase_module_type_t* module_type)
> ```

---

#### emf_cbase_module_unload_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_unload_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_handle_t module_handle)
> ```

---

#### emf_cbase_module_unregister_loader_fn_t

> ```c
> BASE_FN_T(emf_cbase_module_unregister_loader_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_module_loader_handle_t loader_handle)
> ```

---

#### emf_cbase_native_library_loader_interface_load_ext_fn_t

> ```c
> #if defined(Win32) || defined(_WIN32)
> FN_T(emf_cbase_native_library_loader_interface_load_ext_fn_t,
>       emf_cbase_internal_library_handle_result_t,
>       emf_cbase_library_loader_t*, library_loader,
>       const emf_cbase_os_path_char_t* library_path,
>       void* h_file,
>       uint32_t flags)
> #else
> FN_T(emf_cbase_native_library_loader_interface_load_ext_fn_t,
>       emf_cbase_internal_library_handle_result_t,
>       emf_cbase_library_loader_t*, library_loader,
>       const emf_cbase_os_path_char_t* library_path,
>       int flags)
> #endif // defined(Win32) || defined(_WIN32)
> ```

---

#### emf_cbase_native_module_interface_get_exportable_interfaces_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_get_exportable_interfaces_fn_t,
>       emf_cbase_interface_descriptor_const_span_result_t,
>       emf_cbase_native_module_t* module)
> ```

---

#### emf_cbase_native_module_interface_get_interface_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_get_interface_fn_t,
>       emf_cbase_module_interface_result_t,
>       emf_cbase_native_module_t* module,
>       const emf_cbase_interface_descriptor_t* interface_descriptor)
> ```

---

#### emf_cbase_native_module_interface_get_load_dependencies_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_get_load_dependencies_fn_t,
>       emf_cbase_interface_descriptor_const_span_t, void)
> ```

---

#### emf_cbase_native_module_interface_get_module_info_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_get_module_info_fn_t,
>       emf_cbase_module_info_ptr_result_t,
>       emf_cbase_native_module_t* module)
> ```

---

#### emf_cbase_native_module_interface_get_runtime_dependencies_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_get_runtime_dependencies_fn_t,
>       emf_cbase_interface_descriptor_const_span_result_t,
>       emf_cbase_native_module_t* module)
> ```

---

#### emf_cbase_native_module_interface_initialize_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_initialize_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_native_module_t* module)
> ```

---

#### emf_cbase_native_module_interface_load_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_load_fn_t,
>       emf_cbase_native_module_ptr_result_t,
>       emf_cbase_module_handle_t module_handle,
>       emf_cbase_t* base_module,
>       emf_cbase_sys_has_function_fn_t has_function_fn,
>       emf_cbase_sys_get_function_fn_t get_function_fn)
> ```

---

#### emf_cbase_native_module_interface_terminate_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_terminate_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_native_module_t* module)
> ```

---

#### emf_cbase_native_module_interface_unload_fn_t

> ```c
> FN_T(emf_cbase_native_module_interface_unload_fn_t,
>       emf_cbase_module_result_t,
>       emf_cbase_native_module_t* module)
> ```

---

#### emf_cbase_native_module_loader_interface_get_native_module_fn_t

> ```c
> FN_T(emf_cbase_native_module_loader_interface_get_native_module_fn_t,
>       emf_cbase_native_module_ptr_result_t,
>       emf_cbase_module_loader_t* module_loader,
>       emf_cbase_internal_module_handle_t module_handle)
> ```

---

#### emf_cbase_os_path_char_t

> ```c
> typedef EMF_CBASE_OS_PATH_CHAR emf_cbase_os_path_char_t;
> ```

---

#### emf_cbase_sync_handler_lock_fn_t

> ```c
> FN_T(emf_cbase_sync_handler_lock_fn_t, void, 
>       emf_cbase_sync_handler_t* sync_handler)
> ```

---

#### emf_cbase_sync_handler_try_lock_fn_t

> ```c
> FN_T(emf_cbase_sync_handler_try_lock_fn_t, emf_cbase_bool_t, 
>       emf_cbase_sync_handler_t* sync_handler)
> ```

---

#### emf_cbase_sync_handler_unlock_fn_t

> ```c
> FN_T(emf_cbase_sync_handler_unlock_fn_t, void, 
>       emf_cbase_sync_handler_t* sync_handler)
> ```

---

#### emf_cbase_sys_get_function_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_get_function_fn_t,
>       emf_cbase_sys_fn_optional_t,
>       emf_cbase_fn_ptr_id_t fn_id)
> ```

---

#### emf_cbase_sys_get_sync_handler_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_get_sync_handler_fn_t,
>       const emf_cbase_sync_handler_interface_t*,
>       void)
> ```

---

#### emf_cbase_sys_has_function_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_has_function_fn_t, 
>       emf_cbase_bool_t,
>       emf_cbase_fn_ptr_id_t fn_id)
> ```

---

#### emf_cbase_sys_lock_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_lock_fn_t, void, void)
> ```

---

#### emf_cbase_sys_panic_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_panic_fn_t, void, void)
> ```

---

#### emf_cbase_sys_set_sync_handler_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_set_sync_handler_fn_t, void, 
>       const emf_cbase_sync_handler_interface_t* sync_handler)
> ```

---

#### emf_cbase_sys_shutdown_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_shutdown_fn_t, void, void)
> ```

---

#### emf_cbase_sys_try_lock_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_try_lock_fn_t, emf_cbase_bool_t, void)
> ```

---

#### emf_cbase_sys_unlock_fn_t

> ```c
> BASE_FN_T(emf_cbase_sys_unlock_fn_t, void, void)
> ```

---

#### emf_cbase_version_as_string_full_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_as_string_full_fn_t,
>       emf_cbase_version_size_result_t,
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer)
> ```

---

#### emf_cbase_version_as_string_long_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_as_string_long_fn_t,
>       emf_cbase_version_size_result_t,
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer)
> ```

---

#### emf_cbase_version_as_string_short_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_as_string_short_fn_t,
>       emf_cbase_version_size_result_t,
>       const emf_cbase_version_t* version,
>       emf_cbase_version_string_buffer_t* buffer)
> ```

---

#### emf_cbase_version_compare_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_compare_fn_t,
>       int32_t,
>       const emf_cbase_version_t* lhs,
>       const emf_cbase_version_t* rhs)
> ```

---

#### emf_cbase_version_compare_strong_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_compare_strong_fn_t,
>       int32_t,
>       const emf_cbase_version_t* lhs,
>       const emf_cbase_version_t* rhs)
> ```

---

#### emf_cbase_version_compare_weak_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_compare_weak_fn_t,
>       int32_t,
>       const emf_cbase_version_t* lhs,
>       const emf_cbase_version_t* rhs)
> ```

---

#### emf_cbase_version_from_string_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_from_string_fn_t,
>       emf_cbase_version_result_t,
>       const emf_cbase_version_const_string_buffer_t* version_string)
> ```

---

#### emf_cbase_version_is_compatible_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_is_compatible_fn_t,
>       emf_cbase_bool_t,
>       const emf_cbase_version_t* lhs,
>       const emf_cbase_version_t* rhs)
> ```

---

#### emf_cbase_version_new_full_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_new_full_fn_t,
>       emf_cbase_version_t,
>       int32_t major, 
>       int32_t minor,
>       int32_t patch,
>       emf_cbase_version_release_t release_type,
>       int8_t release_number,
>       int64_t build)
> ```

---

#### emf_cbase_version_new_long_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_new_long_fn_t,
>       emf_cbase_version_t,
>       int32_t major, 
>       int32_t minor,
>       int32_t patch,
>       emf_cbase_version_release_t release_type,
>       int8_t release_number)
> ```

---

#### emf_cbase_version_new_short_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_new_short_fn_t,
>       emf_cbase_version_t,
>       int32_t major, 
>       int32_t minor,
>       int32_t patch)
> ```

---

#### emf_cbase_version_string_is_valid_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_string_is_valid_fn_t,
>       emf_cbase_bool_t,
>       const emf_cbase_version_const_string_buffer_t* version_string)
> ```

---

#### emf_cbase_version_string_length_full_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_string_length_full_fn_t,
>       size_t,
>       const emf_cbase_version_t* version)
> ```

---

#### emf_cbase_version_string_length_long_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_string_length_long_fn_t,
>       size_t,
>       const emf_cbase_version_t* version)
> ```

---

#### emf_cbase_version_string_length_short_fn_t

> ```c
> BASE_FN_T(emf_cbase_version_string_length_short_fn_t,
>       size_t,
>       const emf_cbase_version_t* version)
> ```

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
