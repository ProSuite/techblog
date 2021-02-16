---
layout: post
title: About .NET CLS Compliance
author: ujr
date: 2021-02-16
---

**Abstract.** CLS compliant libraries can be used with any language
that targets .NET. You opt-in to CLS compliance with an assembly-level
attribute. Once you opted for CLS compliance, you must mark exceptions
as such or the compiler will warn you.

## Introduction

The .NET *Common Language Runtime* (CLR) supports many
different types as described by the *Common Type System* (CTS).
But while many languages target .NET, not all languages support
all types of the CTS. The *Common Language Specification* (CLS)
describes a subset of required types that all languages that
target .NET must support. (It also describes a few other constraints;
for example, identifiers must not differ in case only.)

CLS compliance is for software libraries.
Those types and members of a library that are CLS compliant
can be used from all languages that target .NET—such is the idea.
The C# compiler can help with CLS compliance by issuing warnings
about non-compliant features.

## Declaring CLS compliance

The `CLSCompliantAttribute` controls CLS compliance.
The defaults are:

- assembly: not CLS compliant
- type: inherits compliance from its assembly
- member: inherits compliance from its type

An assembly can be marked CLS compliant with the
`[assembly: CLSCompliant(true)]` attribute, typically in the
project's *AssemblyInfo.cs* file. If an assembly is marked
CLS compliant, any types that are not CLS compliant must be
marked as such with `[CLSCompliant(false)]`, and similarly
for members of a type; otherwise, the compiler issues a warning.

## Should I go for CLS compliance?

- If your code is a library intended for wide use across .NET,
  then you should go for CLS compliance.
- If your code is a library that exposes many non-CLS-compliant
  types and methods (that is, stuff from a lower-level library),
  then think twice if partial CLS compliance is of any benefit.
- If your code is an application, then CLS compliance is hardly
  beneficial.
  
If you go for CLS compliance with your library, think about the
scoping of the exceptions. If most members (or the essential members)
of a class are not CLS compliant, it makes more sense to mark the
entire class non compliant instead of individual members.

## Decisions for ProSuite

By default, ProSuite assemblies are not CLS compliant. Some assemblies
may be declared CLS compliant if they obviously are compliant, that is,
when there is no more than a handful of exceptions. Reasoning:

1. Neither ArcObjects nor the ArcGIS Pro SDK are CLS compliant;
   we depend so heavily on those libraries that exceptions to
   CLS compliance become the rule. Not being CLS compliant spares
   us literally thousands of `[CLSCompliant(false)]` attributes
   scattered over the code.

2. We have no intention to use any language other than C#. Should
   this ever change, considerate addition of `[CLSCompliant(false)]`
   attributes at the right level of granularity should be a small task
   compared to the introduction of another language.

## References

- <https://docs.microsoft.com/en-us/dotnet/api/system.clscompliantattribute>  
  the `CLSCompliantAttribute` and basic information
- <https://docs.microsoft.com/en-us/dotnet/standard/base-types/common-type-system>
  describes the .NET Common Type System (CTS)
- <https://docs.microsoft.com/en-us/dotnet/standard/language-independence>  
  gives all CLS compliance rules (probably more than you ever want to know)
