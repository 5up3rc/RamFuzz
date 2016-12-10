# RamFuzz

RamFuzz is a fuzzer for individual method parameters in unit tests.  A unit test can use RamFuzz to generate random parameter values for methods under test.  The values are logged, and the log can be replayed to repeat the exact same test scenario.

Random values are not limited just to elemental types: RamFuzz can also automatically produce random objects of any class from the user's code. This allows it to fuzz methods under test that accept class parameters.  For example, if a method takes an `int`, a `struct S`, and a `class C` as parameters, RamFuzz can randomly generate them all!

To accomplish this, RamFuzz includes a code generator that reads the C++ source code and generates from it some test code.  This test code creates a class instance via a randomly chosen constructor, then proceeds to invoke a random sequence of instance methods with randomly generated arguments.  The result is a random class object that can be fed to any method taking that class as a paremeter.

Of course, this produces superficial tests that likely aren't very useful -- most methods don't take completely random parameters but constrain them in some way.  But if the quality of RamFuzz's randomness is good, then running tests repeatedly for a long time will cover a wide range of possible parameter values, likely including some perfectly valid tests.  And because RamFuzz test code logs the choices it made, the accumulated logs can serve as a training set for an AI algorithm that slowly learns how to generate proper parameter values.  Eventually, the AI will learn how to generate the right kinds of parameters for all methods under test.

The AI itself is not a part of RamFuzz, but RamFuzz provides a tool for collecting the training corpus -- see "How to Generate a Training Corpus" below.

RamFuzz is provided under the Apache 2.0 license.  It currently supports a non-trivial subset of C++ -- enough, for instance, to process the [Protocol Buffers](https://github.com/google/protobuf) source and significant parts of the [Apache Mesos](http://mesos.apache.org) source.  This subset will be expanded over time, but please see "Known Limitations" below for some notable missing features.

## How to Use the Code Generator

The `bin/ramfuzz` executable (in LLVM build, see "How to Build" below) generates test code from C++ headers declaring the classes under test.  Instructions for its use are in [main.cpp](main.cpp).  You can see examples in the [test](test) directory, where each `.hpp` file is processed by `bin/ramfuzz` and the result linked with the eponymous `.cpp` file during testing.  Basically, to get a random object of your class `FooBar`, you invoke `ramfuzz::runtime::spin_roulette<ramfuzz::rfFooBar::control>` and access its member `obj`.

`bin/ramfuzz` will output two files in the current directory: `fuzz.hpp` and `fuzz.cpp`.  The former contains test-code declarations, the latter definitions.

Currently, the generated test code doesn't contain a `main()` function (though this may change in the future).  For now, the user should provide `main()` that exercises the RamFuzz test code.  For an example, consider [`test/vec-ptr.cpp`](test/vec-ptr.cpp):

```c++
#include "fuzz.hpp" // Begin by including the declarations of RamFuzz test code.
using namespace ramfuzz::runtime;
int main(int argc, char* argv[]) {
  gen g(argc, argv); // This creates logs. See gen constructor comment in ramfuzz-rt.hpp.
  // Invoke spin_roulette for classes under test.
  return spin_roulette<ramfuzz::rfB::control>(g).obj.sum != 5; // Random instance of class B.
}
unsigned ::ramfuzz::runtime::spinlimit = 3; // See declaration in runtime/ramfuzz-rt.hpp.
```

Here the user invokes `spin_roulette()` for class `B`, the only class under test.  `spin_roulette()` creates a `B` instance using a randomly chosen constructor, then invokes a random sequence of public instance methods on it.  Methods are invoked with randomly generated parameter values, and some may be invoked multiple times, depending on the random number generator.  Every run of this executable exercises a different random scenario and logs it.

If there are multiple classes under test, `main()` will typically invoke `spin_roulette()` for all of them.

The code should be compiled as follows:
```
c++ -std=c++11 user_provided_main.cpp fuzz.cpp ramfuzz/runtime/ramfuzz-rt.cpp
```

### Known Limitations

C++ is a huge language, so the code generator is a work in progress.  Although improvements are made constantly, it currently can't handle the following important categories:
- most template code
- STL containers other than `vector` and `string`
- array arguments
- pure virtual methods that return a reference or a pointer (though other aspects of abstract base classes are supported)

These limitations will typically manifest themselves as ill-formed C++ on the output.

## How to Generate a Training Corpus

When the test executable is run, it creates a log file named (by default) `fuzzlog.t`.  This file's format is described in comments for `ramfuzz::runtime::gen` in [runtime/ramfuzz-rt.hpp](runtime/ramfuzz-rt.hpp); briefly, every line contains two numbers: the encoded source-code location and the generated random value.  The location number may be present on multiple lines, as there may be many dynamic executions of the same source location.  The values are logged in the order of their generation at run time.

An AI training algorithm may take these number pairs and look for patterns in values generated at various points in the test code.  For instance, a statistical classifier may construct a feature vector by using the first number as the index and second as the value.  (The location number may need to be uniquified by dynamic instance first.)

Due to certain technicalities, `fuzzlog.t` doesn't record the outcome (i.e., whether the executable succeeded or failed).  Instead, the exit status of the executable is used to signal success.  RamFuzz includes a script called [`gencorp.py`](gencorp.py), which can be used to run a test executable many times, capture the created logs, and sort them into successes and failures.  Please see its comments for usage.

## How to Build the Code Generator

1. **Get Clang:** the RamFuzz code genertor is a Clang tool, so first get and build Clang using [these instructions](http://clang.llvm.org/get_started.html).  RamFuzz is known to work with Clang/LLVM versions 3.8.1 and 4.0.0.

2. **Drop RamFuzz into Clang:** RamFuzz source is intended to go under `clang/tools/extra` and build from there (as described in [this](http://clang.llvm.org/docs/LibASTMatchersTutorial.html#step-1-create-a-clangtool) Clang tutorial).  Drop the top-level RamFuzz directory into `clang/tools/extra` and add it (using `add_subdirectory`) to `clang/tools/extra/CMakeLists.txt`.

3. **Rebuild Clang:** Now the standard LLVM build procedure should produce a `bin/ramfuzz` executable.

4. **Run Tests:** There are some end-to-end tests in the [`test`](test) directory -- see [`test.py`](test/test.py) there.  RamFuzz adds a new build target `ramfuzz-test`, which executes all RamFuzz tests.  This target depends on `bin/ramfuzz`, so `bin/ramfuzz` will be rebuilt before testing if it's out of date.

## Contributing

RamFuzz welcomes contributions by the community.  Each contributor will retain copyright over their code but must sign the developer certificate in the [CONTRIBUTORS](CONTRIBUTORS) file by adding their name/contact to the list and using the `-s` flag in all commits.

RamFuzz code relies on extensive Doxygen-style comments to provide guidance to the reader.  Each directory also contains a README file that briefly summarizes what's in it and where to start reading.

To avoid any hurdles to contribution, there is no formal coding style.  Don't worry about formalities, just write good code.  Look it over and ask yourself: is it simple to use, simple to read, and simple to modify?  If so, it's a welcome contribution to this project.
