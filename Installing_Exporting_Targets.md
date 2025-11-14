# Installing & Exporting Targets

CMake’s **install** and **export** system is how modern projects package themselves so that other projects can use `find_package()` to locate and link them.

## 1. Installing a target

You can tell CMake where to place the built library and headers:

## `mylib/CMakeLists.txt`

```cmake
install(
    TARGETS mylib
    EXPORT mylibTargets
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)

install(
    DIRECTORY include/
    DESTINATION include
)
```

When you run:

```bash
cmake --install build --prefix ./install
```

You’ll get this structure:

```
install/
├── include/mylib.h
└── lib/libmylib.a
```

---

## 2. Exporting target definitions

To make your library importable by other CMake projects, you export its *target definitions*:

```cmake
install(
    EXPORT mylibTargets
    FILE mylibTargets.cmake
    NAMESPACE mylib::
    DESTINATION lib/cmake/mylib
)
```

This generates a file like `mylibTargets.cmake`, containing your target metadata (paths, include dirs, etc.).
The `NAMESPACE` (e.g. `mylib::`) ensures clean name scoping.

---

## 3. Creating a Config file

Projects typically ship a `*Config.cmake` file that points to the exported targets.
Add this to your `mylib/CMakeLists.txt`:

```cmake
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/mylibConfigVersion.cmake"
    VERSION 1.0
    COMPATIBILITY AnyNewerVersion
)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/mylibConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/mylibConfig.cmake
    @ONLY
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/mylibConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/mylibConfigVersion.cmake
    DESTINATION lib/cmake/mylib
)
```

Then create a small template file `mylibConfig.cmake.in`:

```cmake
@PACKAGE_INIT@

include("${CMAKE_CURRENT_LIST_DIR}/mylibTargets.cmake")
```

Now, once you’ve installed your library, other projects can simply do:

```cmake
find_package(mylib 1.0 REQUIRED)
target_link_libraries(app PRIVATE mylib::mylib)
```

---

## 4. Consumer project (for testing)

Create a new folder `consumer_project`:

```
consumer_project/
├── CMakeLists.txt
└── main.c
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.15)
project(consumer_project C)

find_package(mylib 1.0 REQUIRED)

add_executable(app main.c)
target_link_libraries(app PRIVATE mylib::mylib)
```

### main.c

```c
#include "mylib.h"

int main(void)
{
    mylib_print_hello();
    return 0;
}
```

Then configure:

```bash
cmake -S . -B build -DCMAKE_PREFIX_PATH=../cmake-starter/install
cmake --build build
```

If successful, you’ve built a reusable library package!


## Concept Summary

| Concept               | Description                                          |
| --------------------- | ---------------------------------------------------- |
| **install()**         | Copies built artifacts to an install prefix          |
| **EXPORT**            | Records target info into a `.cmake` file             |
| **Config.cmake**      | Lets `find_package()` locate and import your project |
| **CMAKE_PREFIX_PATH** | Tells CMake where to look for packages               |
| **NAMESPACE**         | Provides unique scoped names (e.g., `mylib::mylib`)  |

---

## Usage Tips

* Always export your targets with a **namespace** (`MyLib::`).
* Don’t manually write long `FindXYZ.cmake` unless needed; use `Config.cmake` pattern.
* When you install headers, maintain structure: `include/<projectname>/header.h`.
* Use `write_basic_package_version_file()` to define version compatibility.

---
