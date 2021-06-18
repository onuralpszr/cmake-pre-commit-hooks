# pre-commit hooks

[![PyPI version](https://badge.fury.io/py/cmake-pre-commit-hooks.svg)](https://badge.fury.io/py/cmake-pre-commit-hooks) ![CI Build](https://github.com/Takishima/cmake-pre-commit-hooks/actions/workflows/ci.yml/badge.svg) ![CodeQL](https://github.com/Takishima/cmake-pre-commit-hooks/actions/workflows/codeql-analysis.yml/badge.svg)

This is a [pre-commit](https://pre-commit.com) hooks repo that integrates C/C++ linters/formatters to work with CMake-based projects.
> [clang-format](https://clang.llvm.org/docs/ClangFormatStyleOptions.html),
[clang-tidy](https://clang.llvm.org/extra/clang-tidy/),
[cppcheck](http://cppcheck.sourceforge.net/) and
[iwyu](https://include-what-you-use.org/)

It is largely based on the work found [here](https://github.com/pocc/pre-commit-hooks). The main difference with POCC's
pre-commit hooks is that the ones from this repository will do a CMake configuration step prior to running any
pre-commit hooks. This is done in order to have CMake generate the compilation database file that can then be used by
the various hooks (using the `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` CMake option).

This repository is only has Python-based pre-commit hooks.


## Example usage

Assuming that you have the following directory structure for your projects

    root
    ├── .pre-commit-config.yaml
    ├── CMakeLists.txt
    └── src
        └── err.cpp

with the following file contents:

__.pre-commit-config.yaml__

    repos:
      - repo: /Users/damien/code/pre-commit/cmake-pre-commit-hooks
        rev: 1.0.0
        hooks:
          - id: clang-format
          - id: clang-tidy
            args: [--checks=readability-magic-numbers,--warnings-as-errors=*]
          - id: cppcheck


__CMakeLists.txt__

    cmake_minimum_required(VERSION 3.15)
    project(LANGUAGE CXX)
    add_library(mylib STATIC src/err.cpp)

__src/err.cpp__

    #include <string>
    int main() { int i; return 10; }

Runnning pre-commit on the above project will lead to an output similar to this one:

    $ pre-commit run --all-files
    clang-format.............................................................Failed
    - hook id: clang-format
    - exit code: 1

    src/err.cpp
    ====================
    <  int main() { int i; return 10; }
    ---
    >  int main() {
    >    int i;
    >    return 10;
    >  }

    clang-tidy...............................................................Failed
    - hook id: clang-tidy
    - exit code: 1

    /tmp/temp/src/err.cpp:2:28: error: 10 is a magic number; consider replacing it with a named constant [readability-magic-numbers,-warnings-as-errors]
    int main() { int i; return 10; }
                               ^

    cppcheck.................................................................Failed
    - hook id: cppcheck
    - exit code: 1

    /tmp/temp/src/err.cpp:2:18: style: Unused variable: i [unusedVariable]
    int main() { int i; return 10; }
                     ^

Note that your mileage may vary depending on the version of the tools. The example above was generated using
`clang-format` 12.0.0, `clang-tidy` 12.0.0 and `cppcheck` 2.4.1.

## Using the Hooks

Python 3.6+ is required to use these hooks as all 3 invoking scripts are written in it. As this is also the minimum
version of pre-commit, this should not be an issue.

Running multiple hooks in parallel is currently supported by using the `fastener` Python package. If the hooks are run
in parallel, only one of the hooks will run the CMake configure step while the others will simply wait until the call to
CMake ends to continue. In the case where the hooks are run serially, all the hooks will be running the CMake configure
step. However, if nothing changed in your CMake configuration, this should not cost too much time.


### Installation

For installing the various utilities, refer to your package manager documentation. Some guidance can also be found
[here](https://github.com/pocc/pre-commit-hooks#installation).


### Hook Info

| Hook Info                                                                | Type                 | Languages                             |
| ------------------------------------------------------------------------ | -------------------- | ------------------------------------- |
| [clang-format](https://clang.llvm.org/docs/ClangFormatStyleOptions.html) | Formatter            | C, C++, ObjC                          |
| [clang-tidy](https://clang.llvm.org/extra/clang-tidy/)                   | Static code analyzer | C, C++, ObjC                          |
| [cppcheck](http://cppcheck.sourceforge.net/)                             | Static code analyzer | C, C++                                |


### Hook Option Comparison

| Hook Options                                                             | Fix In Place | Enable all Checks                             | Set key/value |
| ------------------------------------------------------------------------ | ------------ | --------------------------------------------- | --------------- |
| [clang-format](https://clang.llvm.org/docs/ClangFormatStyleOptions.html) | `-i`         |                   | |
| [clang-tidy](https://clang.llvm.org/extra/clang-tidy/)                   | `--fix-errors` [1] | `-checks=*` `-warnings-as-errors=*` [2] | |
| [cppcheck](http://cppcheck.sourceforge.net/)                             |  | `-enable=all` | |

[1]: `-fix` will fail if there are compiler errors. `-fix-errors` will `-fix` and fix compiler errors if it can, like missing semicolons.

[2]: Be careful with `-checks=*`.  can have self-contradictory rules in newer versions of llvm (9+): modernize wants to use [trailing return type](https://clang.llvm.org/extra/clang-tidy/checks/modernize-use-trailing-return-type.html) but Fuchsia [disallows it](https://clang.llvm.org/extra/clang-tidy/checks/fuchsia-trailing-return.html).


### The '--' doubledash option

Options after `--` like `-std=c++11` will be interpreted correctly for `clang-tidy`. Make sure they sequentially follow
the `--` argument in the hook's args list.


### Standalone Hooks

If you want to have predictable return codes for your C linters outside of pre-commit, these hooks are available via
[PyPI](https://pypi.org/project/CLinters/).  Install it with `pip install CLinters`.  They are named as `cmake-pc-$cmd-hook`, so
`clang-format` becomes `cmake-pc-clang-format-hook`.

If you want to run the tests below, you will need to install them from PyPI or locally with `pip install .`.
