# Scala 3 Compatibility

This repository is created with purpose of finding suitable solution for Scala 3 compatibility issues, that are experianced by library maintainers who are stuck at version 3.0.2 if they want to support the widest possible range of users.

## Goals

- Newer version of the compiler should be able to generate output that can be consumed by older ones
- Library authors should be able to declare parts of API that require newer version of the language than the rest of the library
- Users should be able to safely use symbols added to the language API in `3.x`, when they are using `3.y` as their output version, as long as those symbols do not require language features added after `3.y`, even if `x > y`.

## Implementation

### Output version

We propose that compiler should accept flag `--scala-target` that accepts scala version as an argument. Specified output version needs to be lower or equal to the used version of the compiler. The default value for this flag is its maximal value - the version of the compiler itself.

Specifying output version makes sure that tasty files produced during compilation would have a version matching output version. It also makes sure that no symbol is used that was added to the language api in any version newer that specified output version. If the code is using any language feature that was added in version newer that output version, the compilation error should be rised. All new symbols added to stdlib in future versions need to be marked with some annotation specifying the version, eg. `@since("3.1")`.

### Local output version

There should be a possibility to mark any file in project to be using higher output version than what is specified in the project configuration. It can be implemented by top level import, such as:

```scala
import language.target.`3.1`
```

This should override other menas of specifying output version.

Symbol can reference symbol from other files if and only if file containing referencing symbols has higher or equal output version than file containg referenced symbols. This allows us to sort compiled sources at the start of the compilation. This means that if multiple files would be compiled together, compiler first processes files with output version 3.0, then those with 3.1 (remembering symbols from previous files), then those with 3.2 and so on.

This feature gives library maintainers possibility to gradualy extend APIs with features requireing newer compiler, without modifying core of the library that can be used with older compilers. This has some limitations, eg. it is not alowing for adding new supertype for existing type. On the other hand adding new types, new given intances and extension methods to exisitng types is possible and should be sufficient for relatively fast and stable improvement of library API.

Adding local output versions means that now it is normal situation for the compiler to encounter tasty files with versions newer than specified output version for current compilation run. The compiler shouldn't raise error but insted should just ignore those tasty files together with corresponding classfiles. In the future we may want to read classfiles to know what symbols can potentialy reside there and use that information for better error messages for not resolved references that can potentially be resolved with higher output version.

### Language API compatibility

Multiple symbols that are added to the standard libraries in new releases do not require any new features of the language. Even though they won't be available in projects with lower output version. To mitigate that with every new release of the compiler we can also release `compat` artifact, that would contain all symbols added between 3.0 and current version of the compiler. Every one of them would be defined in the file with output version matching the earliest version of the language that given symbol can be defined in. They would have the same qualified name as the "true" symbols.

Those `compat` artifact can be then added as an ordinary dependency to any project. This assures that the symbols are available both in compiletime and in the runtime. Those `compat` symbols needs to be marked in some way so the compiler recognizes them and ignores them if "true" symbols are also on the classpath. This case can occur when library `B` is dependeing on `A`, `B` has higher output version than `A` and `A` is using `compat` as a dependency. We can implement mentioned marking either by new annotation or by some information in the metadata of the `compat` artifact.
