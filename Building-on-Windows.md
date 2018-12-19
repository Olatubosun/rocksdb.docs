# Building on Windows

This is a simple step-by-step explanation of how I was able to build RocksDB (or RocksJava) and all of the 3rd-party libraries on Microsoft Windows 10. The Windows build system was already in place, however it took some trial-and-error for me to be able to build the 3rd-party libraries and incorporate them into the build.

## Pre-requisites
1. Microsoft Visual Studio 2015 (Community) with "Desktop development with C++" installed
2. [CMake](https://cmake.org/)
3. [Git](https://git-scm.com/downloads) - I used the Windows Git Bash.
4. [Mercurial](https://www.mercurial-scm.org/wiki/Download) - I used the 64bit MSI installer
5. wget

## Steps

Create a directory somewhere on your machine that will be used a container for both the RocksDB source code and that of its 3rd-party dependencies. On my machine I used `C:\Users\aretter\code`, from hereon in I will just refer to it as `%CODE_HOME%`; which can be set as an environment variable, i.e. `SET CODE_HOME=C:\Users\aretter\code`.

All of the following is executed from the "**Developer Command Prompt for VS2015**":

### Build GFlags
```
cd %CODE_HOME%
wget https://github.com/gflags/gflags/archive/v2.2.0.zip
unzip v2.2.0.zip
cd gflags-2.2.0
mkdir target
cd target
cmake -G "Visual Studio 14 Win64" ..
```

Open the project in Visual Studio, create a new x64 Platform by copying the Win32 platform and selecting x64 CPU. Close Visual Studio.

```
msbuild gflags.sln /p:Configuration=Debug /p:Platform=x64
msbuild gflags.sln /p:Configuration=Release /p:Platform=x64
```

The resultant static library can be found in `%CODE_HOME%\gflags-2.2.0\target\lib\Debug\gflags_static.lib` or `%CODE_HOME%\gflags-2.2.0\target\lib\Release\gflags_static.lib`.


### Build Snappy
```
cd %CODE_HOME%
wget https://github.com/google/snappy/archive/1.1.7.zip
unzip 1.1.7.zip
cd snappy-1.1.7
mkdir build
cd build
cmake ..
msbuild Snappy.sln /p:Configuration=Debug /p:Platform=x64
msbuild Snappy.sln /p:Configuration=Release /p:Platform=x64
```

The resultant static library can be found in `%CODE_HOME%\snappy-1.1.7\build\Debug\snappy.lib` or `%CODE_HOME%\snappy-1.1.7\build\Release\snappy.lib`.


### Build LZ4
```
cd %CODE_HOME%
wget https://github.com/lz4/lz4/archive/v1.7.5.zip
unzip v1.7.5.zip
cd lz4-1.7.5
cd visual\VS2010
devenv lz4.sln /upgrade
msbuild lz4.sln /p:Configuration=Debug /p:Platform=x64
msbuild lz4.sln /p:Configuration=Release /p:Platform=x64
```

The resultant static library can be found in `%CODE_HOME%\lz4-1.7.5\visual\VS2010\bin\x64_Debug\liblz4_static.lib` or `%CODE_HOME%\lz4-1.7.5\visual\VS2010\bin\x64_Release\liblz4_static.lib`.


### Build ZLib
```
cd %CODE_HOME%
wget http://zlib.net/zlib1211.zip
unzip zlib1211.zip
cd zlib-1.2.11\contrib\vstudio\vc14
```

Edit the file ` zlibvc.vcxproj`, changing `<command>cd ..\..\contrib\masmx64 bld_ml64.bat</command>` to `<command>cd ..\..\masmx64 bld_ml64.bat</command>`.
Add a new line after masmx64.

```
"C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\bin\amd64_x86\vcvarsamd64_x86.bat"
msbuild zlibvc.sln /p:Configuration=Debug /p:Platform=x64
msbuild zlibvc.sln /p:Configuration=Release /p:Platform=x64
copy x64\ZlibDllDebug\zlibwapi.lib x64\ZlibStatDebug\
copy x64\ZlibDllRelease\zlibwapi.lib x64\ZlibStatRelease\
```

The resultant static library can be found in `%CODE_HOME%\zlib-1.2.11\contrib\vstudio\vc14\x64\ZlibStatDebug\zlibstat.lib` or `%CODE_HOME%\zlib-1.2.11\contrib\vstudio\vc14\x64\ZlibStatRelease\zlibstat.lib`.

### Build ZLib

wget https://github.com/facebook/zstd/archive/v1.3.7.zip
unzip v1.3.7.zip
cd zstd-1.3.7/build/VS2010
devenv zstd.sln /upgrade
msbuild zstd.sln /p:Configuration=Debug /p:Platform=x64
msbuild zstd.sln /p:Configuration=Release /p:Platform=x64

The resultant static library can be found in `%CODE_HOME%\zstd-1.3.7\/build/VS2010/bin/x64_Debug/libzstd_static.lib` or `%CODE_HOME%\zstd-1.3.7\/build/VS2010/bin/x64_Release/libzstd_static.lib`.


### Build RocksDB
```
cd %CODE_HOME%
git clone https://github.com/facebook/rocksdb.git
cd rocksdb
```

Edit the file `%CODE_HOME%\rocksdb\thirdparty.inc` to have these changes:

```
set(GFLAGS_HOME $ENV{THIRDPARTY_HOME}/gflags-2.2.0)
set(GFLAGS_INCLUDE ${GFLAGS_HOME}/target/include)
set(GFLAGS_LIB_DEBUG ${GFLAGS_HOME}/target/lib/Debug/gflags_static.lib)
set(GFLAGS_LIB_RELEASE ${GFLAGS_HOME}/target/lib/Release/gflags_static.lib)

set(SNAPPY_HOME $ENV{THIRDPARTY_HOME}/snappy-1.1.7)
set(SNAPPY_INCLUDE ${SNAPPY_HOME} ${SNAPPY_HOME}/build)
set(SNAPPY_LIB_DEBUG ${SNAPPY_HOME}/build/Debug/snappy.lib)
set(SNAPPY_LIB_RELEASE ${SNAPPY_HOME}/build/Release/snappy.lib)

set(LZ4_HOME $ENV{THIRDPARTY_HOME}/lz4-1.7.5)
set(LZ4_INCLUDE ${LZ4_HOME}/lib)
set(LZ4_LIB_DEBUG ${LZ4_HOME}/visual/VS2010/bin/x64_Debug/liblz4_static.lib)
set(LZ4_LIB_RELEASE ${LZ4_HOME}/visual/VS2010/bin/x64_Release/liblz4_static.lib)

set(ZLIB_HOME $ENV{THIRDPARTY_HOME}/zlib-1.2.11)
set(ZLIB_INCLUDE ${ZLIB_HOME})
set(ZLIB_LIB_DEBUG ${ZLIB_HOME}/contrib/vstudio/vc14/x64/ZlibStatDebug/zlibstat.lib)
set(ZLIB_LIB_RELEASE ${ZLIB_HOME}/contrib/vstudio/vc14/x64/ZlibStatRelease/zlibstat.lib)

set(ZSTD_HOME $ENV{THIRDPARTY_HOME}/zstd-1.3.7)
set(ZSTD_INCLUDE ${ZSTD_HOME}/lib)
set(ZSTD_LIB_DEBUG ${ZSTD_HOME}/build/VS2010/bin/x64_Debug/libzstd_static.lib)
set(ZSTD_LIB_RELEASE ${ZSTD_HOME}/build/VS2010/bin/x64_Release/libzstd_static.lib)
```

And then finally to compile RocksDB:

* **NOTE**: The default CMake build will generate MSBuild project files which include the `/arch:AVX2` flag. If you have this CPU extension instruction set, then the generated binaries will also only work on other CPU's with AVX2. If you want to create a build which has no specific CPU extensions, then you should also pass the `-DPORTABLE=1` flag in the `cmake` arguments below.

```
mkdir build
cd build
set THIRDPARTY_HOME=%CODE_HOME%
set JAVA_HOME="C:\Program Files\Java\jdk1.7.0_80"
cmake -G "Visual Studio 14 Win64" -DJNI=1 -DGFLAGS=1 -DSNAPPY=1 -DLZ4=1 -DZLIB=1 -DZSTD=1 -DXPRESS=1 ..
msbuild rocksdb.sln /p:Configuration=Release
```
