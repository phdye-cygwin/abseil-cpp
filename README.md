# Abseil - C++ Common Libraries

## Cygwin Port (`cygwin/support` branch)

This branch adds Cygwin support to Abseil LTS 20250512.1.

Abseil does not officially support Cygwin. An upstream effort is underway ([abseil/abseil-cpp#2012](https://github.com/abseil/abseil-cpp/pull/2012), Carlo Bramini) but has not yet been merged. This branch carries 10 minimal patches -- 5 derived from that PR, 5 discovered during testing -- that bring the full test suite to 218/218 passing.

The patches exist to support a Cygwin build of [Protocol Buffers v34.1](https://github.com/protocolbuffers/protobuf), which depends on this version of Abseil. They are intended as a stopgap until Cygwin support lands upstream.

The end goal is `grpcio` (Python gRPC bindings) on Cygwin. The dependency chain is:

    psutil <- pypinfo <- grpcio <- protobuf <- abseil-cpp

### Building on Cygwin

Prerequisites: GCC 13+, CMake 3.16+, Cygwin x86_64.

GoogleTest must be available for the test suite. Either let CMake download
it (`ABSL_USE_GOOGLETEST_HEAD=ON`) or point to an installed copy
(`ABSL_USE_EXTERNAL_GOOGLETEST=ON`). The second option is required if you
plan to `cmake --install` and have downstream projects consume the result
via `find_package(absl)` -- see [Install note](#install-note) below.

**Option A** -- download GoogleTest at configure time:

```bash
mkdir build && cd build
cmake /path/to/abseil-cpp \
  -DCMAKE_CXX_STANDARD=17 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=1 -DGTEST_HAS_PTHREAD=1" \
  -DABSL_BUILD_TESTING=ON \
  -DABSL_USE_GOOGLETEST_HEAD=ON
cmake --build . --parallel 8
ctest --timeout 300
```

**Option B** -- use installed GoogleTest (required for install/release):

```bash
mkdir build && cd build
cmake /path/to/abseil-cpp \
  -DCMAKE_CXX_STANDARD=17 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH=/usr/local \
  -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=1 -DGTEST_HAS_PTHREAD=1" \
  -DABSL_BUILD_TESTING=ON \
  -DABSL_USE_EXTERNAL_GOOGLETEST=ON \
  -DABSL_FIND_GOOGLETEST=ON
cmake --build . --parallel 8
ctest --timeout 300
```

Both `-D` flags in `CMAKE_CXX_FLAGS` are required:

- **`_GLIBCXX_USE_CXX11_ABI=1`** -- Cygwin's libstdc++ defaults to the
  old COW string ABI (`sizeof(std::string)` = 8). Protobuf requires the
  SSO ABI (`sizeof(std::string)` = 32). Both Abseil and Protobuf must
  agree.

- **`GTEST_HAS_PTHREAD=1`** -- googletest's header auto-detection
  omits Cygwin from its pthread platform list. Without this flag,
  `libgmock.a` and test binaries disagree on the `Mutex` class layout,
  causing any gmock-based test to crash. Only needed when building tests.

### Binary release

Pre-built static libraries for Cygwin x86_64 (GCC 13.4.0, Release mode)
are available on the
[Releases](https://github.com/phdye-cygwin/abseil-cpp/releases) page,
tagged for use with Protobuf 7.35.0.

GoogleTest 1.16.0 is vendored inside the tarball under
`lib/absl/vendor/`. Abseil's cmake config references `GTest::gtest` and
`GTest::gmock` for test helper targets that protobuf's test suite requires;
the vendored copy satisfies these references automatically without
conflicting with any system-wide GoogleTest installation.

The tarball installs to `/usr/local`, which avoids conflicting with any
system abseil installed by the Cygwin package manager in `/usr`.

```bash
# Install
cd / && tar xzf abseil-cpp-20250512.1-cygwin-x86_64.tar.gz
```

This places:
- Libraries in `/usr/local/lib/libabsl_*.a`
- Headers in `/usr/local/include/absl/`
- CMake config in `/usr/local/lib/cmake/absl/`
- Vendored GoogleTest in `/usr/local/lib/absl/vendor/`
- pkg-config in `/usr/local/lib/pkgconfig/absl_*.pc`

To use from CMake:

```cmake
list(APPEND CMAKE_PREFIX_PATH /usr/local)
find_package(absl REQUIRED)
```

No separate `find_package(GTest)` is needed -- `abslConfig.cmake` finds
the vendored copy automatically. If your project already loads GTest before
finding abseil, the vendored copy is skipped.

### Building with a custom prefix

To install to a location other than `/usr/local`, add
`-DCMAKE_INSTALL_PREFIX` to the cmake configure step and install after
building:

```bash
mkdir build && cd build
cmake /path/to/abseil-cpp \
  -DCMAKE_CXX_STANDARD=17 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/path/to/prefix \
  -DCMAKE_PREFIX_PATH=/path/to/prefix \
  -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=1 -DGTEST_HAS_PTHREAD=1" \
  -DABSL_BUILD_TESTING=ON \
  -DABSL_USE_EXTERNAL_GOOGLETEST=ON \
  -DABSL_FIND_GOOGLETEST=ON
cmake --build . --parallel 8
ctest --timeout 300
cmake --install .
```

GoogleTest must be installed at the prefix before building abseil.
`CMAKE_PREFIX_PATH` tells cmake where to find it.

The cmake config files are relocatable -- they compute the prefix from
their own install path, so `find_package(absl)` works regardless of
where the tree is rooted, as long as the prefix is in
`CMAKE_PREFIX_PATH`:

```cmake
list(APPEND CMAKE_PREFIX_PATH /path/to/prefix)
find_package(absl REQUIRED)
```

### Downstream: Protobuf Cygwin port

The matching Protobuf build is at [phdye-cygwin/protobuf](https://github.com/phdye-cygwin/protobuf) (branch `cygwin/update-port`), targeting Protobuf 7.35.0. Protobuf's CMake build fetches Abseil 20250512.1 from GitHub; the patches here must be applied to the fetched source after `cmake` configure and before `cmake --build`.

### Known limitations

- **Build time**: ~19 minutes on an 8-core machine with `-j8`.
- **Test timeouts**: `ctest --timeout 300` is needed for two tests
  that are slow by design, not due to Cygwin:
  - `flags_sequence_lock_test` (~225s) -- 115 parameterized cases
    each running a concurrent read/write loop with a hardcoded
    5-second deadline.
  - `mutex_test` (~103s) -- 72 timeout-verification cases with
    deliberate sub-second sleeps (450ms, 900ms).
    These would exceed a 60-second timeout on any platform.

### Upstream tracking

When [abseil/abseil-cpp#2012](https://github.com/abseil/abseil-cpp/pull/2012)
merges, 5 of the 10 patches here become redundant (the 5 derived from
that PR). The remaining patches address issues not covered by PR #2012:

| Patch | Why it's separate |
|-------|-------------------|
| `config.h` (ABSL_HAVE_MMAP) | PR #2012 omits this; build fails without it |
| `log/internal/config.h` (Tid width) | Thread ID truncation in log output |
| `base/internal/raw_logging.cc` (POSIX write) | Raw log output silently discarded |
| `strings/charconv_test.cc` (NaN) | Test-only: Cygwin strtod() platform difference |
| `log/stripping_test.cc` (/proc/self/exe) | Test-only: Cygwin provides this like Linux |

### Patch summary

| File | Purpose |
|------|---------|
| `policy_checks.h` | Remove `#error "Cygwin is not supported"` |
| `attributes.h` | Disable weak symbols (PE/COFF) |
| `config.h` | Recognize Cygwin mmap support |
| `low_level_alloc.h` | Correct allocation capability flags |
| `sysinfo.cc` | Fix GetTID cast for pointer-sized `pthread_t` |
| `sysinfo.h` | Widen `pid_t` to hold thread handle |
| `log/internal/config.h` | Widen `Tid` to prevent thread ID truncation in logs |
| `base/internal/raw_logging.cc` | Enable POSIX `write()` for raw log output |
| `strings/charconv_test.cc` | Accommodate Cygwin `strtod()` NaN behavior |
| `log/stripping_test.cc` | Enable `/proc/self/exe` path |

---

The repository contains the Abseil C++ library code. Abseil is an open-source
collection of C++ code (compliant to C++17) designed to augment the C++
standard library.

## Table of Contents

- [Cygwin Port](#cygwin-port-cygwinsupport-branch)
- [About Abseil](#about)
- [Quickstart](#quickstart)
- [Building Abseil](#build)
- [Support](#support)
- [Codemap](#codemap)
- [Releases](#releases)
- [License](#license)
- [Links](#links)

<a name="about"></a>
## About Abseil

Abseil is an open-source collection of C++ library code designed to augment
the C++ standard library. The Abseil library code is collected from Google's
own C++ code base, has been extensively tested and used in production, and
is the same code we depend on in our daily coding lives.

In some cases, Abseil provides pieces missing from the C++ standard; in
others, Abseil provides alternatives to the standard for special needs
we've found through usage in the Google code base. We denote those cases
clearly within the library code we provide you.

Abseil is not meant to be a competitor to the standard library; we've
just found that many of these utilities serve a purpose within our code
base, and we now want to provide those resources to the C++ community as
a whole.

<a name="quickstart"></a>
## Quickstart

If you want to just get started, make sure you at least run through the
[Abseil Quickstart](https://abseil.io/docs/cpp/quickstart). The Quickstart
contains information about setting up your development environment, downloading
the Abseil code, running tests, and getting a simple binary working.

<a name="build"></a>
## Building Abseil

[Bazel](https://bazel.build) and [CMake](https://cmake.org/) are the official
build systems for Abseil.
See the [quickstart](https://abseil.io/docs/cpp/quickstart) for more information
on building Abseil using the Bazel build system.
If you require CMake support, please check the [CMake build
instructions](CMake/README.md) and [CMake
Quickstart](https://abseil.io/docs/cpp/quickstart-cmake).

<a name="support"></a>
## Support

Abseil follows Google's [Foundational C++ Support
Policy](https://opensource.google/documentation/policies/cplusplus-support). See
[this
table](https://github.com/google/oss-policies-info/blob/main/foundational-cxx-support-matrix.md)
for a list of currently supported versions compilers, platforms, and build
tools.

<a name="codemap"></a>
## Codemap

Abseil contains the following C++ library components:

* [`base`](absl/base/)
  <br /> The `base` library contains initialization code and other code which
  all other Abseil code depends on. Code within `base` may not depend on any
  other code (other than the C++ standard library).
* [`algorithm`](absl/algorithm/)
  <br /> The `algorithm` library contains additions to the C++ `<algorithm>`
  library and container-based versions of such algorithms.
* [`cleanup`](absl/cleanup/)
  <br /> The `cleanup` library contains the control-flow-construct-like type
  `absl::Cleanup` which is used for executing a callback on scope exit.
* [`container`](absl/container/)
  <br /> The `container` library contains additional STL-style containers,
  including Abseil's unordered "Swiss table" containers.
* [`crc`](absl/crc/) The `crc` library contains code for
  computing error-detecting cyclic redundancy checks on data.
* [`debugging`](absl/debugging/)
  <br /> The `debugging` library contains code useful for enabling leak
  checks, and stacktrace and symbolization utilities.
* [`flags`](absl/flags/)
  <br /> The `flags` library contains code for handling command line flags for
  libraries and binaries built with Abseil.
* [`hash`](absl/hash/)
  <br /> The `hash` library contains the hashing framework and default hash
  functor implementations for hashable types in Abseil.
* [`log`](absl/log/)
  <br /> The `log` library contains `LOG` and `CHECK` macros and facilities
  for writing logged messages out to disk, `stderr`, or user-extensible
  destinations.
* [`memory`](absl/memory/)
  <br /> The `memory` library contains memory management facilities that augment
  C++'s `<memory>` library.
* [`meta`](absl/meta/)
  <br /> The `meta` library contains type checks
  similar to those available in the C++ `<type_traits>` library.
* [`numeric`](absl/numeric/)
  <br /> The `numeric` library contains 128-bit integer types as well as
  implementations of C++20's bitwise math functions.
* [`profiling`](absl/profiling/)
  <br /> The `profiling` library contains utility code for profiling C++
  entities.  It is currently a private dependency of other Abseil libraries.
* [`random`](absl/random/)
  <br /> The `random` library contains functions for generating pseudorandom
  values.
* [`status`](absl/status/)
  <br /> The `status` library contains abstractions for error handling,
  specifically `absl::Status` and `absl::StatusOr<T>`.
* [`strings`](absl/strings/)
  <br /> The `strings` library contains a variety of strings routines and
  utilities.
* [`synchronization`](absl/synchronization/)
  <br /> The `synchronization` library contains concurrency primitives (Abseil's
  `absl::Mutex` class, an alternative to `std::mutex`) and a variety of
  synchronization abstractions.
* [`time`](absl/time/)
  <br /> The `time` library contains abstractions for computing with absolute
  points in time, durations of time, and formatting and parsing time within
  time zones.
* [`types`](absl/types/)
  <br /> The `types` library contains non-container utility types.
* [`utility`](absl/utility/)
  <br /> The `utility` library contains utility and helper code.

<a name="releases"></a>
## Releases

Abseil recommends users "live-at-head" (update to the latest commit from the
master branch as often as possible). However, we realize this philosophy doesn't
work for every project, so we also provide [Long Term Support
Releases](https://github.com/abseil/abseil-cpp/releases) to which we backport
fixes for severe bugs. See our [release
management](https://abseil.io/about/releases) document for more details.

<a name="license"></a>
## License

The Abseil C++ library is licensed under the terms of the Apache
license. See [LICENSE](LICENSE) for more information.

<a name="links"></a>
## Links

For more information about Abseil:

* Consult our [Abseil Introduction](https://abseil.io/about/intro)
* Read [Why Adopt Abseil](https://abseil.io/about/philosophy) to understand our
  design philosophy.
* Peruse our
  [Abseil Compatibility Guarantees](https://abseil.io/about/compatibility) to
  understand both what we promise to you, and what we expect of you in return.
