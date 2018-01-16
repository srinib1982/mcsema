# McSema [![Slack Chat](http://empireslacking.herokuapp.com/badge.svg)](https://empireslacking.herokuapp.com/)

McSema is an executable lifter. It translates ("lifts") executable binaries from native machine code to LLVM bitcode. LLVM bitcode is an [intermediate representation](https://en.wikipedia.org/wiki/Intermediate_representation) form of a program that was originally created for the [retargetable LLVM compiler](https://llvm.org), but which is also very useful for performing program analysis methods that would not be possible to perform on an executable binary directly.

McSema enables analysts to find and retroactively harden binary programs against security bugs, independently validate vendor source code, and generate application tests with high code coverage. McSema isn’t just for static analysis. The lifted LLVM bitcode can also be [fuzzed with libFuzzer](https://github.com/trailofbits/mcsema/blob/master/docs/UsingLibFuzzer.md), an LLVM-based instrumented fuzzer that would otherwise require the target source code. The lifted bitcode can even be [compiled](https://github.com/trailofbits/mcsema/blob/master/docs/UsingLibFuzzer.md) back into a [runnable program](https://github.com/trailofbits/mcsema/blob/master/docs/McSemaWalkthrough.md)! This is a procedure known as static binary rewriting, binary translation, or binary recompilation.

McSema supports lifting both Linux (ELF) and Windows (PE) executables, and understands most x86 and amd64 instructions, including integer, X87, MMX, SSE and AVX operations. AARCH64 (ARMv8) instruction support is in active development.

Using McSema is a two-step process: control flow recovery, and instruction translation. Control flow recovery is performed using the `mcsema-disass` tool, which relies on IDA Pro to disassemble a binary file and produce a control flow graph. Instruction translation is then performed using the `mcsema-lift` tool, which converts the control flow graph into LLVM bitcode. Under the hood, the instruction translation capability of `mcsema-lift` is implemented in the [`remill` library](https://github.com/trailofbits/remill). The development of `remill` was a result of refactoring and improvements to McSema, and was first introduced with McSema version 2.0.0. Read more about `remill` [below](#faq).

McSema and `remill` were developed and are maintained by Trail of Bits, funded by and used in research for DARPA and the US Department of Defense.

## Build status

|       | master                                   |
| ----- | ---------------------------------------- |
| Linux | [![Build Status](https://travis-ci.org/trailofbits/mcsema.svg?branch=master)](https://travis-ci.org/trailofbits/mcsema) |

## Features

* Lifts 32- and 64-bit Linux ELF and Windows PE binaries to bitcode, including executables and shared libraries for each platform.
* Supports a large subset of x86 and x86-64 instructions, including most integer, X87, MMX, SSE, and AVX operations.
* McSema runs on Windows and Linux and has been tested on Windows 7, 10, Ubuntu (14.04, 16.04), and openSUSE.
* McSema can cross-lift: it can translate Linux binaries on Windows, or Windows binaries on Linux.
* Output bitcode is compatible with the LLVM toolchain (versions 3.5 and up).
* Translated bitcode can be analyzed or [recompiled as a new, working executable](docs/McSemaWalkthrough.md) with functionality identical to the original.

## Use-cases

Why would anyone translate binaries *back* to bitcode?

* **Binary Patching And Modification**. Lifting to LLVM IR lets you cleanly modify the target program. You can run obfuscation or hardening passes, add features, remove features, rewrite features, or even fix that pesky typo, grammatical error, or insane logic. When done, your new creation can be recompiled to a new binary sporting all those changes. In the [Cyber Grand Challenge](https://blog.trailofbits.com/2015/07/15/how-we-fared-in-the-cyber-grand-challenge/), we were able to use McSema to translate challenge binaries to bitcode, insert memory safety checks, and then re-emit working binaries.

* **Symbolic Execution with KLEE**. [KLEE](https://klee.github.io/) operates on LLVM bitcode, usually generated by providing source to the LLVM toolchain. McSema can lift a binary to LLVM bitcode, [permitting KLEE to operate on previously unavailable targets](https://blog.trailofbits.com/2014/12/04/close-encounters-with-symbolic-execution-part-2/). See our [walkthrough](examples/Maze/README.md) showing how to run KLEE on a symbolic maze.

* **Re-use existing LLVM-based tools**. KLEE is not the only tool that becomes available for use on bitcode. It is possible to run LLVM optimization passes and other LLVM-based tools like [libFuzzer](http://llvm.org/docs/LibFuzzer.html) on [lifted bitcode](docs/UsingLibFuzzer.md).

* **Analyze the binary rather than the source**. Source level analysis is great but not always possible (e.g. you don't have the source) and, even when it is available, it lacks compiler transformations, re-ordering, and optimizations. Analyzing the actual binary guarantees that you're analyzing the true executed behavior.

* **Write one set of analysis tools**. Lifting to LLVM IR means that one set of analysis tools can work on both the source and the binary. Maintaining a single set of tools saves development time and effort, and allows for a single set of better tools.

## Dependencies

| Name | Version | 
| ---- | ------- |
| [Git](https://git-scm.com/) | Latest |
| [CMake](https://cmake.org/) | 3.2+ |
| [Google Protobuf](https://github.com/google/protobuf) | 2.6.1 |
| [Google Flags](https://github.com/google/glog) | Latest |
| [Google Log](https://github.com/google/glog) | Latest |
| [Google Test](https://github.com/google/googletest) | Latest |
| [Intel XED](https://github.com/intelxed/xed) | Latest |
| [LLVM](http://llvm.org/) | 3.5+ |
| [Clang](http://clang.llvm.org/) | 3.5+ |
| [Python](https://www.python.org/) | 2.7 | 
| [Python Package Index](https://pypi.python.org/pypi) | Latest |
| [python-protobuf](https://pypi.python.org/pypi/protobuf) | 3.2.0 |
| [IDA Pro](https://www.hex-rays.com/products/ida) | 6.7+ |

## Getting and building the code

### On Linux

#### Step 1: Install dependencies

```shell
sudo apt-get update
sudo apt-get upgrade

sudo apt-get install \
     git \
     cmake \
     python2.7 python-pip \
     wget \
     build-essential \
     gcc-multilib g++-multilib \
     libtinfo-dev \
     lsb-release \
     realpath

sudo pip install --upgrade pip
sudo pip install 'protobuf==3.2.0'
```

##### Fixing IDA Pro's Python installation (Ubuntu 14.04)

Note: If you are using IDA on 64 bit Ubuntu and your IDA install does not use the system Python, you can add the `protobuf` library manually to IDA's zip of modules.

```shell
# Python module dir is generally in /usr/lib or /usr/local/lib
touch /path/to/python2.7/dist-packages/google/__init__.py
cd /path/to/lib/python2.7/dist-packages/
sudo zip -rv /path/to/ida-6.X/python/lib/python27.zip google/
sudo chown your_user:your_user /home/your_user/ida-6.7/python/lib/python27.zip
```

#### Step 2: Clone the repository

The next step is to clone the [Remill](https://github.com/trailofbits/remill) repository. We then clone the McSema repository into the `tools` subdirectory of Remill. This is kind of like how Clang and LLVM are distributed separately, and the Clang source code needs to be put into LLVM's tools directory.

**Notice that when building McSema, you should always use a specific Remill commit hash (the one we test). This hash can be found in the .remill_commit_id file**.

```shell
git clone --depth 1 https://github.com/trailofbits/mcsema.git
export REMILL_VERSION=`cat ./mcsema/.remill_commit_id`

git clone https://github.com/trailofbits/remill.git
cd remill
git checkout -b temp $REMILL_VERSION

mv ../mcsema tools
```

#### Step 3: Build McSema

McSema is a kind of sub-project of Remill, similar to how Clang is a sub-project of LLVM. To that end, we invoke Remill's build script to build both Remill and McSema. It will also download all remaining dependencies needed by Remill.

The following script will build Remill and McSema into the `remill-build` directory, which will be placed in the current working directory.

```shell
./remill/scripts/build.sh
```

This script accepts several additional command line options:

* `--prefix PATH`: Install files to `PATH`. By default, `PATH` is `/usr/local`.
* `--llvm-version MAJOR.MINOR`: Download pre-built dependencies for LLVM version MAJOR.MINOR. The default is to use LLVM 4.0.
* `--build-dir PATH`: Produce all intermediate build files in `PATH`. By default, `PATH` is `$CWD/remill-build`.
* `--use-system-compiler`: Compile Remill+McSema using the system compiler toolchain (typically the GCC).

#### Step 4: Install McSema

The next step is to build the code.

```shell
cd remill-build
sudo make install
```

This will install McSema _globally_. Once installed, you may use `mcsema-disass` for disassembling binaries, and `mcsema-lift-4.0` for lifting the disassembled binaries. If you specified `--llvm-version 3.6` to the `build.sh` script, then you would use `mcsema-lift-3.6`.

#### Step 5: Verifying Your McSema Installation

In order to verify that McSema works correctly as built, head on over to [the documentation on integration tests](docs/AddingNewTests.md). Check that you can run the tests and that they pass.

## Additional Documentation

* [McSema command line reference](docs/CommandLineReference.md)
* [Common Errors](docs/CommonErrors.md) and [Debugging Tips](docs/DebuggingTips.md)
* [How to add support for a new instruction](https://github.com/trailofbits/remill/blob/master/docs/ADD_AN_INSTRUCTION.md)
* [How to use McSema: A walkthrough](docs/McSemaWalkthrough.md)
* [Life of an instruction](docs/LifeOfAnInstruction.md)
* [Limitations](docs/Limitations.md)
* [Navigating the source code](docs/NavigatingTheCode.md)
* [Using McSema with libFuzzer](docs/UsingLibFuzzer.md)

## Getting help

If you are experiencing problems with McSema or just want to learn more and contribute, join the `#binary-lifting` channel of the [Empire Hacking Slack](https://empireslacking.herokuapp.com/). Alternatively, you can join our mailing list at [mcsema-dev@googlegroups.com](https://groups.google.com/forum/?hl=en#!forum/mcsema-dev) or email us privately at mcsema@trailofbits.com.

## FAQ

### How do you pronounce McSema and where did the name come from

This is a hotly contested issue. We must explore the etymology of the name to find an answer. The "Mc" in McSema was originally a contraction of the words "Machine Code," and the "sema" is short for "semantics." At that time, McSema used LLVM's instruction decoder to take machine code bytes, and turn them into `llvm::MCInst` data structures. It is possible that "MC" in that case is pronounced em-see. Alas, even those who understand the origin of the name pronounce it as if it were related to America's favorite fast food joint.

### Why do I need IDA Pro to use McSema

McSema's goal is binary to bitcode translation. Accurate disassembly and control flow recovery is a separate and difficult problem. IDA has already invested countless hours of engineering into getting disassembly right, and it only makes sense that we re-use existing work. We understand that not everyone can afford an IDA license. With the original release of McSema, we shipped our own recursive-descent disassembler. It was never as good as IDA, and it never would be. Maintaining the broken tool took away valuable development time from more important McSema work. We hope to eventually transition to more accessible control flow recovery front-ends, such as Binary Ninja (we have a branch with [experimental Binary Ninja support](https://github.com/trailofbits/mcsema/tree/binja_cfg_updates/tools/mcsema_disass/binja)). We very warmly welcome pull requests that implement support for new control flow recovery front-ends.

### What is Remill, and why does McSema need it

[Remill](https://github.com/trailofbits/remill) is a library that McSema uses to lift individual machine code instructions to LLVM IR. You can think of McSema being to Remill as Clang is to LLVM. Remill's scope is small: it focuses on instruction semantics only, and it provides semantics for x86, x86-64, and AArch64 instruction semantics. McSema's scope is much bigger: it focuses on lifting entire programs. To do so, McSema must lift the individual instructions, but there's a lot more to lifting programs than just the instructions; there are code and data cross-references, segments, etc.

### I'm a student and I'd like to contribute to McSema: how can I help

We would love to take you on as an intern to help improve McSema. We have several project ideas labelled `intern_project` in [the issues tracker](https://github.com/trailofbits/mcsema/issues?q=is%3Aissue+is%3Aopen+label%3Aintern_project). You are not limited to those items: if you think of a great feature you want in McSema, let us know and we will sponsor it. Simply contact us on our [Slack channel](https://empireslacking.herokuapp.com/) or via mcsema@trailofbits.com and let us know what you'd want to work on and why.
