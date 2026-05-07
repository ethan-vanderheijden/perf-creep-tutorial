# Tutorial: investigating a tiny performance regression with [`pt-fuser`](https://github.com/ethan-vanderheijden/pt-fuser)

This interactive tutorial will walk you through setting up your testing environment and investigating a real-world bug in MongoDB v7.0.0 with the help of our tool, [`pt-fuser`](https://github.com/ethan-vanderheijden/pt-fuser). You are encouraged to run all these commands on your own machine, and if you encounter any issues, please open an issue or a pull request!

### Table of Contents

<ol>
    <li><a href="#building-mongodb-v7.0.0">Building MongoDB v7.0.0</a></li>
    <li><a href="#setting-up-ycsb">Setting up YCSB</a></li>
    <li><a href="#setting-up-your-testing-environment">Setting up your testing environment</a></li>
    <li>
        <a href="#traditional-performance-analysis">Traditional Performance Analysis</a>
        <ul>
        <li><a href="#benchmarking-tools">Benchmarking metrics</a></li>
        <li><a href="#flamegraphs">Flamegraphs</a></li>
        <li><a href="#function-latency-instrumentation">Function latency instrumentation</a></li>
        </ul>
    </li>
    <li>
        <a href="#pt-fuser-workflow">pt-fuser Workflow</a>
        <ul>
        <li><a href="#collecting-filtering-and-merging-traces">Collecting, filtering, and merging traces</a></li>
        <li><a href="#visualizing-and-comparing-traces">Visualizing traces and performing a manual comparison</a></li>
        </ul>
    </li>
    <li><a href="#takeaways">Takeaways</a></li>
</ol>

## Building MongoDB v7.0.0

In this tutorial, we will investigate a real performance regression that affected MongoDB versions 4.7.0 through 7.1.0. We will build MongoDB v7.0.0 from source, which contains the regression, and then rebuild it after applying the fix. In the following sections, we will analyze the performance of both versions to see how we may detect such a tiny regression.

For reference, the regression was introduced in [this commit](https://github.com/mongodb/mongo/commit/eaaee39e2c4eaf9938c5a75bce30648435ae10cc#diff-7b9a463114d0718dd549d5a65df54e34c431f389cb7f579bc7e4bc9560bf2099). The regression was filed in [this JIRA ticket](https://jira.mongodb.org/browse/SERVER-79775), and it was ultimately fixed by [this commit](https://github.com/mongodb/mongo/commit/6a82e4262717c24bd088141369921a5ae0ec2d82).

### What was the regression?

The regression was located in the function `parseSubFields()` inside ***mongo/src/mongo/db/matcher/expression_parser.cpp***. This function is part of the query parsing code, so it is executed at the beginning of every query.

In the code, it calls `e.wrap()`, where `e` is a `BSONElement` object, which the `BSONElement` object to be copied:
```cpp
doc_validation_error::createAnnotation(
    expCtx,
    e.fieldNameStringData().toString(),
    BSON(name << e.wrap())
)
```

However, `createAnnotation()` was actually a no-op because the guard is false and the functions just returns `NULL`:
```cpp
createAnnotation() {
  if (expCtx->isParsingCollectionValidator) {  // this guard is false
    ...
  } else {
    return NULL;
  }
}
```

The fix is to check the guard `if (expCtx->isParsingCollectionValidator)` before calling `e.wrap()`, skipping the extraneous object copy.

### Building MongoDB

Generally speaking, for performance testing, you need to build the binaries from source rather than download pre-built binaries from a package manager. Each software project uses a different build system, so you'll need to consult their documentation. Word of warning: figuring out a new build system can be quite time-consuming, especially if you are building an old version of the codebase because modern compilers can throw all sorts of weird errors when compiling old code.

MongoDB's documentation can be found in [mongo/docs/building.md](https://github.com/mongodb/mongo/blob/r7.0.0/docs/building.md). All MongoDB versions have a corresponding git tag, so in particular, the tag "r7.0.0" corresponds to MongoDB v7.0.0.

To begin, let's create a directory for all our work and clone the MongoDB repository:
```bash
mkdir creep-tutorial
cd creep-tutorial

# for future reference, let's record the tutorial directory
export TUTORIAL_DIR=$(pwd)

# clone with depth of 1 to discard all git history, which speeds up the cloning process
git clone --branch r7.0.0 --depth 1 https://github.com/mongodb/mongo.git
```

MongoDB's build system uses Python, so let's create a virtual environment and install all the necessary dependencies.

However, we must use a version of Python <3.12 because MongoDB v7.0.0 uses an old version of `setuptools` that relies on a feature removed in Python 3.12. In the following command, I will use Python 3.11.

Also, we must edit `mongo/etc/pip/components/core.req` and change the line `PyYAML >= 3.0.0, <= 6.0.3` to `PyYAML >= 3.0.0`. Otherwise, it will try to install PyYAML v6.0.0, which is a broken release. Building old code is so much fun, isn't it? Now, we can continue:

```bash
python3.11 -m venv venv-3-11
source venv-3-11/bin/activate

python3 -m pip install -r mongo/etc/pip/compile-requirements.txt
```

Next, we have to monkey patch the source code to ensure it compiles without errors:
- Open `src/third_party/boost/boost/thread/future.hpp`, go to line 4672, and change `that_=x.that;` to `that_=x.that_;`. Believe it or not, this is a [known typo](https://github.com/boostorg/thread/issues/402) in older versions of the Boost library.
- Open `src/mongo/db/free_mon/free_mon_options.h` and add `#include <cstdint>` at the top of the file. Otherwise, the compiler will throw bespoke errors when it tries to compile `enum class EnableCloudStateEnum : std::int32_t` because `std::int32_t` is undefined.

Finally, let's create a directory to store the final MongoDB binary. We will be building MongoDB twice, once for the buggy version and once for the patched version:
```bash
mkdir install-mongo-bugggy
mkdir install-mongo-patched
```

Now, we can build the original, buggy version of MongoDB v7.0.0:
```bash
cd mongo

# Modern compilers are stricter and throw more warnings than old compilers
# we need --disable-warnings-as-errors to ignore these recently introduced warnings
python3 buildscripts/scons.py \
   DESTDIR=../install-mongo-buggy/ \
   install-mongod \
   --disable-warnings-as-errors
```

> [!WARNING]  
> Building MongoDB for the first time can take up to an hour!

Next, we will apply the git patch fixing the regression and rebuild MongoDB. You can download the patch, called `mongo-regression.patch`, from this repository.
```bash
git apply ../mongo-regression.patch

python3 buildscripts/scons.py \
   DESTDIR=../install-mongo-patched/ \
   install-mongod \
   --disable-warnings-as-errors
```

## Setting up YCSB

Now that we have our database built, we need a benchmarking tool to run queries against the database and measure its performance. We will be using [YCSB](https://github.com/brianfrankcooper/YCSB), which is a popular (but unmaintained) benchmarking tool for NoSQL databases. Different applications will need different benchmarking setups, so as always, you'll have to read through the documentation.



## Setting up your testing environment

## Traditional Performance Analysis

### Benchmarking tools

### Flamegraphs

### Function latency instrumentation

## pt-fuser Workflow

### Collecting, filtering, and merging traces

### Visualizing and comparing traces

## Takeaways
