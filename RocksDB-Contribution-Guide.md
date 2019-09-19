# Basic Development Workflow

As most open-source projects in github, RocksDB contributors work on their fork, and send pull requests to RocksDB’s facebook repo. After a reviewer approves the pull request, a RocksDB team member at Facebook will merge it.


# How to Run Unit Tests

## build systems

RocksDB uses gtest. The makefile used for _GNU make_ has some supports to help developers run all unit tests in parallel, which will be introduced below. If you use cmake, you might need find your way to run all the unit tests (you are welcome to contribute build system to make it easier).

## run unit tests in parallel

In order to run unit tests in parallel, first install _GNU parallel_ on your host, and run run
```
make all check [-j] 
```
You can specify number of parallel tests to run using environment variable `J=1`, for example:
```
make J=64 all check -j
```

If you switch between release and debug build, normal or lite build, or compiler or compiler options, call `make clean` first. So here is a safe routine to run all tests:

```
make clean
make J=64 all check -j
```

## Debug single unit test failures

RocksDB uses _gtest_. You can running specific unit test by running the test binary contains it. If you use GNU make, the test binary will be just under your checkpoint. For example, test `DBBasicTest.OpenWhenOpen` is in binary `db_basic_test`, so just run
```
./db_basic_test
```
will run all tests in the binary.

gtest provides some useful command line parameters, and you can see them by calling `--help`:
```
./db_basic_test —help
```
 Here are some frequently used ones:

Run subset of tests using `--gtest_filter`. If you only want to run `DBBasicTest.OpenWhenOpen`, call
```
./db_basic_test —gtest_filter=“*DBBasicTest.OpenWhenOpen*”
```
By default, the test DB created by tests are cleared up even if test fails. You can try to preserve it by using `--gtest_throw_on_failure`. If you want to stop the debugger when assert fails, specify `--gtest_break_on_failure`.

By default, the temporary test files will be under /tmp/rocksdbtest-<number>/ (except when running in parallel they are under /dev/shm). You can override the location by using environment variable `TEST_TMPDIR`. For example:
```
TEST_TMPDIR=/dev/shm/my_dir ./db_basic_test
```
## Java Unit Tests

Sometimes we need to run Java tests too. Run
```
make jclean rocksdbjava jtest
```
You can put `-j` but sometimes it causes problem. Try to remove `-j` if you see problems.

## Some other build flavors

For more complicated code changes, we ask contributors to run more build flavors before sending the code review. The makefile for _GNU make_ has better pre-defined support for it, although it can be manually done in _CMake_ too.

To build with _AddressSanitizer (ASAN)_, set environment variable `COMPILE_WITH_ASAN`:
```
COMPILE_WITH_ASAN=1 make all check -j
```
To build with _ThreadSanitizer (TSAN)_, set environment variable `COMPILE_WITH_TSAN`:
```
COMPILE_WITH_TSAN=1 make all check -j
```
To run all `valgrind tests`:
```
make valgrind_test -j
```
To run _UndefinedBehaviorSanitizer (UBSAN)`, set environment variable `COMPILE_WITH_UBSAN`:
```
COMPILE_WITH_UBSAN=1 make all check -j
```
To run `llvm`'s analyzer, run
```
make analyze
```
# Code Style

RocksDB follows _Google C++ Style_: https://google.github.io/styleguide/cppguide.html . We limit each line to 80 characters.

Some formatting can be done by a formatter by running
```
build_tools/format-diff.sh
```
or simply `make format` if you use _GNU make_. If you lack of dependencies to run it, the script will print out instructions for you to install them. 

# Requirements Before Sending a Pull Request
## Add Unit Tests
Almost all code changes need to go with changes in unit tests to validate the change. For new features, new unit tests or tests scenarios need to be added even if it has been validated manually. This is to make sure future contributors can rerun the tests to validate their changes don't cause problem with the feature.

## Simple Changes
Pull requests for simple changes can be sent after running all unit tests with any build favor and see all tests pass. If any public interface is changed, or Java code involved, Java tests also need to be run.

## Complex Changes
If the change is complicated enough, ASAN, TSAN and valgrind need to be run on your local environment before sending the pull request. If you run ASAN with higher version of llvm with covers almost all the functionality of valgrind, valgrind tests can be skipped.

## Changes with Higher Risk or Some Unknowns
For changes with higher risks, other than running all tests with multiple flavors, a crash test cycle needs to be executed and see no failure. If crash test doesn't cover the new feature, consider to add it there.
To run all crash test, run
```
make crash_test -j
```
If you can't use _GNU make_, you can manually build db_stress binary, and run script:
```
  python -u tools/db_crashtest.py whitebox
  python -u tools/db_crashtest.py blackbox
  python -u tools/db_crashtest.py --simple whitebox
  python -u tools/db_crashtest.py --simple blackbox
  python -u tools/db_crashtest.py --cf_consistency blackbox
  python -u tools/db_crashtest.py --cf_consistency whitebox 
```
