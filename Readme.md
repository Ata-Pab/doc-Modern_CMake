> **Reference**
> - [1]: [CMake 4.2.0-rc3 Documentation](https://cmake.org/cmake/help/latest/index.html)
> - [3]: [ESP-IDF Programming Guide v5](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html)
> - [4]: [Getting Started with CMake](https://cmake.org/getting-started/)
> - [6]: [An Introduction to Modern CMake](https://cliutils.gitlab.io/modern-cmake/)
> - [7]: [Modern CMake – Tips and Tricks](https://www.incredibuild.com/blog/modern-cmake-tips-and-tricks)

# **CMake Design**

Create a **starter repo** that builds:

* A single executable (`app`)
* A static library (`mylib`)
* A header + source shared between them

Then run basic CMake commands to build it.

## Folder layout

```
cmake-starter/
├── CMakeLists.txt        ← root CMake file
├── mylib/
│   ├── CMakeLists.txt
│   ├── include/
│   │   └── mylib.h
│   └── src/
│       └── mylib.c
└── app/
    ├── CMakeLists.txt
    └── main.c
```

---

## Root `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.15)
project(cmake_starter C)

# Set C standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Add subdirectories (library and app)
add_subdirectory(mylib)
add_subdirectory(app)
```

---

## `mylib/CMakeLists.txt`

```cmake
# Create the library target
add_library(mylib
    src/mylib.c
)

# Specify include directories for this target
target_include_directories(mylib
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# target_compile_definitions(mylib PUBLIC USE_MYLIB_FEATURE)
```

---

## `app/CMakeLists.txt`

```cmake
add_executable(app
    main.c
)

# Link our library to this executable
target_link_libraries(app PRIVATE mylib)
```

---

## `mylib/include/mylib.h`

```c
#ifndef MYLIB_H
#define MYLIB_H

void mylib_print_hello(void);

#endif // MYLIB_H
```

---

## `mylib/src/mylib.c`

```c
#include <stdio.h>
#include "mylib.h"

void mylib_print_hello(void)
{
    printf("Hello from mylib!\n");
}
```

---

## `app/main.c`

```c
#include "mylib.h"

int main(void)
{
    mylib_print_hello();
    return 0;
}
```

---

## How to build it?

From the `cmake-starter` root folder:

```bash
cmake -S . -B build
cmake --build build
```

This will ``configure`` the project, generate ``build`` files (Make/Ninja), ``compile`` both the library and executable

Then run:

```bash
./build/app/app
```

Output:

```
Hello from mylib!
```

---

## Explanations

| Concept                        | Explanation                                         |
| ------------------------------ | --------------------------------------------------- |
| `project()`                    | Defines your project name and language              |
| `add_executable()`             | Creates a build target for a program                |
| `add_library()`                | Creates a static/shared library                     |
| `add_subdirectory()`           | Includes subprojects                                |
| `target_link_libraries()`      | Links a library to another target                   |
| `target_include_directories()` | Declares header search paths                        |
| `CMAKE_CURRENT_SOURCE_DIR`     | The directory where the current CMakeLists lives    |
| Out-of-source build            | Keeps build artifacts in a separate `build/` folder |

# **Modern CMake**

## Target Properties, Visibility, and Build Types

### 1. The Target-centric approach

Old CMake used global variables like `INCLUDE_DIRECTORIES()` and `CMAKE_C_FLAGS`.
Modern CMake discourages this — instead, *everything belongs to a target.*

Think of a **target** (library, executable, etc.) as an object that carries:

* Include paths
* Compile definitions
* Compile options
* Linked libraries

When another target links to it, these properties propagate according to `PUBLIC`, `PRIVATE`, or `INTERFACE` rules.

---

### 2. `PUBLIC`, `PRIVATE`, and `INTERFACE`

| Keyword       | Applies to target itself? | Propagates to dependents? | Example use                                       |
| ------------- | ------------------------- | ------------------------- | ------------------------------------------------- |
| **PRIVATE**   | ✅ Yes                     | ❌ No                      | Internal-only flags, include dirs, etc.           |
| **PUBLIC**    | ✅ Yes                     | ✅ Yes                     | Headers or flags needed by both library and users |
| **INTERFACE** | ❌ No                      | ✅ Yes                     | Header-only or compile-only properties            |

---

### 3. Updating the example

We’ll modify `mylib/CMakeLists.txt`:

```cmake
add_library(mylib
    src/mylib.c
)

# Interface (public API) include dir
target_include_directories(mylib
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# Example of private compiler flag (affects only this target)
target_compile_options(mylib
    PRIVATE
        -Wall -Wextra
)

# Example of a public definition (visible to dependents)
target_compile_definitions(mylib
    PUBLIC
        MYLIB_VERSION=1
)
```

And `app/CMakeLists.txt` remains:

```cmake
add_executable(app main.c)
target_link_libraries(app PRIVATE mylib)
```

---

### 4. Build types and configurations

CMake supports **build types** such as `Debug`, `Release`, `RelWithDebInfo`, `MinSizeRel`.

You can check or set the active one:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
```

You can use different flags for each:

```cmake
set(CMAKE_C_FLAGS_DEBUG "-O0 -g")
set(CMAKE_C_FLAGS_RELEASE "-O3")
```

Better yet, use:

```cmake
target_compile_definitions(mylib PRIVATE
    $<$<CONFIG:Debug>:DEBUG_BUILD>
    $<$<CONFIG:Release>:NDEBUG>
)
```

That `$<...>` syntax is a **generator expression**, evaluated at build time — we’ll go deeper into it later.

---

### 5. Header-only library example (INTERFACE)

Sometimes you have utility headers only:

```cmake
add_library(utils INTERFACE)

target_include_directories(utils
    INTERFACE
        ${CMAKE_CURRENT_SOURCE_DIR}/include
)

target_compile_definitions(utils
    INTERFACE
        USE_UTILS=1
)
```

No `.c` files, no compilation. But any target linking `utils` gets these includes and defines.

---
