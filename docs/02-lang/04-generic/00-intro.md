---
title: 2.4. Generics and Macros
sidebar_class_name: hidden-sidebar-item
---

Similarly to types, reference lifetimes are essentially static contracts determined at compile time, which typically do not exist and therefore cannot be manipulated at runtime in traditional statically typed languages. Generic programming provides a way to make these rigid contracts programmable, based on the compiler supporting parameterised types and lifetimes used in contract logic to generate flexible contracts. While traditional code is executed by the machine and reusable across different runtime data, generic code is executed by the compiler and reusable across different static contracts (particularly types). Unlike dynamic types and decorators relying on runtime metadata, generic programming operates purely at compile time for zero-cost abstractions — the priority of Rust's design philosophy.
