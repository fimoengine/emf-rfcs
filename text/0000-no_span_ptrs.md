- Feature Name: `no_span_ptrs`
- Start Date: 2021-05-22
- RFC PR: [fimoengine/emf-rfcs#0000](https://github.com/fimoengine/emf-rfcs/pull/0000)

# Summary

[summary]: #summary

Change function definitions to pass span types by value.

# Motivation

[motivation]: #motivation

Spans are fat pointers to a contiguous sequence of elements, with a known length. They consist of a pointer to the start
of the sequence, and the number of elements in the sequence. Because of their small size, it is more efficient to pass
them by value, instead of by reference.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Given a span type, any function definition which operates on the elements inside the span, instead of the span itself,
is modified to accept and return the span by value.
