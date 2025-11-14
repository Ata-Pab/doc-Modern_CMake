# What are Generator Expressions?

**Definition:**
Generator expressions (short: *genexes*) are special CMake syntax that looks like:

```
$<EXPRESSION:arguments>
```

They are **evaluated at *build-generation time*** (not configure time).

* Normal CMake commands (like `if()`, `set()`, `message()`) are evaluated **while generating** the build system.
* Generator expressions are evaluated **by the generator**, i.e., during build system generation — so they can adapt to configuration or target-specific contexts (e.g., Debug vs Release).

---

### Example Use Case

Let’s say you want to:

* Add `-O0` only in *Debug*,
* Add `-O3` only in *Release*.

Old CMake (wrong way):

```cmake
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  target_compile_options(mylib PRIVATE -O0)
else()
  target_compile_options(mylib PRIVATE -O3)
endif()
```

Modern CMake (using generator expressions):

```cmake
target_compile_options(mylib PRIVATE
    $<$<CONFIG:Debug>:-O0>
    $<$<CONFIG:Release>:-O3>
)
```

Cleaner and works for *multi-config generators* (e.g., Visual Studio, Xcode) that build both Debug and Release simultaneously.

---

# Common Patterns

### 1. Configuration-based selection

```cmake
$<$<CONFIG:Debug>:-DDEBUG_MODE>
$<$<CONFIG:Release>:-DNDEBUG>
```

### 2. Logical operators

You can combine with AND, OR, NOT:

```cmake
$<$<AND:$<CONFIG:Release>,$<CXX_COMPILER_ID:GNU>>:-O3>
```

### 3. File-specific properties

For example, specify compiler options only for certain source files:

```cmake
set_source_files_properties(file.c PROPERTIES
    COMPILE_OPTIONS "$<$<CONFIG:Debug>:-Wall>"
)
```

### 4. Target-based properties

Access target properties dynamically:

```cmake
$<TARGET_PROPERTY:mylib,INTERFACE_INCLUDE_DIRECTORIES>
```

# Real-world usage

### Conditional compile options:

```cmake
target_compile_options(app PRIVATE
    $<$<CONFIG:Debug>:-g3>
    $<$<CONFIG:Release>:-O3>
)
```

### Conditional definitions:

```cmake
target_compile_definitions(mylib PUBLIC
    $<$<CONFIG:Debug>:ENABLE_LOGGING>
)
```

### Conditional include dirs:

```cmake
target_include_directories(mylib PUBLIC
    $<$<CONFIG:Debug>:${CMAKE_CURRENT_SOURCE_DIR}/debug_headers>
)
```

---

### Advanced Example

```cmake
target_compile_options(mylib PRIVATE
    $<$<AND:$<CONFIG:Release>,$<CXX_COMPILER_ID:GNU>>:-Ofast>
)
```

→ Adds `-Ofast` only if **Release build + GCC compiler**.


## See the difference in action

In the existing `mylib/CMakeLists.txt`, add:

```cmake
target_compile_definitions(mylib PRIVATE
    $<$<CONFIG:Debug>:DEBUG_BUILD>
    $<$<CONFIG:Release>:NDEBUG>
)

target_compile_options(mylib PRIVATE
    $<$<CONFIG:Debug>:-Wall -O0>
    $<$<CONFIG:Release>:-O3>
)
```

Then build both versions:

```bash
cmake -S . -B build/Debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/Debug

cmake -S . -B build/Release -DCMAKE_BUILD_TYPE=Release
cmake --build build/Release
```

You’ll see different flags being passed depending on configuration.

## Pattern Descriptions

| Pattern                                   | Description                   | Example                                                  |
| ----------------------------------------- | ----------------------------- | -------------------------------------------------------- |
| `$<CONFIG:Debug>`                         | True if building Debug        | `$<$<CONFIG:Debug>:-DDEBUG>`                             |
| `$<BOOL:var>`                             | True if variable is non-empty | `$<$<BOOL:USE_UART>:-DUSE_UART>`                         |
| `$<STREQUAL:a,b>`                         | True if a==b                  | `$<$<STREQUAL:${CMAKE_SYSTEM_NAME},Linux>:-DLINUX>`      |
| `$<AND:...>` / `$<OR:...>` / `$<NOT:...>` | Logical operators             | `$<$<AND:$<CONFIG:Debug>,$<BOOL:ENABLE_LOGGING>>:-DLOG>` |
| `$<TARGET_PROPERTY:target,prop>`          | Fetch target property         | `$<TARGET_PROPERTY:mylib,INTERFACE_INCLUDE_DIRECTORIES>` |

---
