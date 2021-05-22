- Feature Name: `os_path_string`
- Start Date: 2021-05-22
- RFC PR: [fimoengine/emf-rfcs#0014](https://github.com/fimoengine/emf-rfcs/pull/0014)

# Summary

[summary]: #summary

Introduces the `emf_cbase_os_path_string_t` type for representing path strings.

# Motivation

[motivation]: #motivation

Currently, there are two types of strings used in the project. Most of the strings are represented as fat pointers,
which contain a pointer to the start and its length. The second type are classic null-terminated strings. While compact,
a downside of C strings is, that counting the length of a string necessitates an iteration of the entire string. Because
of that, changing from a C string representation to a fat pointer representation is always an operation of linear time
complexity. Going from a fat pointer to a C string may be more efficient, if the last character is already a null byte.
Currently, the only type of strings implemented as C strings are paths. This RFC introduces a new fat pointer path
string, which replaces the old null-terminated string.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The project already defines the `emf_cbase_os_path_char_t`, which accounts for different path representations of
operating systems. This type can be used as a building block for the new `emf_cbase_os_path_string_t` type:

```c
typedef struct emf_cbase_os_path_string_t {
  const emf_cbase_os_path_char_t* data;
  size_t length;
} emf_cbase_os_path_string_t;
```

This new type is a pointer to the start of a string, which may be null-terminated, paired with the length of the string.
Functions and types, which use the old type are replaced with this new type.