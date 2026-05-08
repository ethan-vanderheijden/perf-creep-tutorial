# Tutorial: investigating a tiny performance regression with [`pt-fuser`](https://github.com/ethan-vanderheijden/pt-fuser)

This interactive tutorial will walk you through setting up your testing environment and investigating a real-world bug in MongoDB v7.0.0 with the help of our tool, [`pt-fuser`](https://github.com/ethan-vanderheijden/pt-fuser). You are encouraged to run all these commands on your own machine, and if you encounter any issues, please open an issue or a pull request!

### Table of Contents

<ol>
    <li><a href="#building-mongodb-v700">Building MongoDB v7.0.0</a></li>
    <li><a href="#setting-up-ycsb-mongodb">Setting up YCSB / MongoDB</a></li>
    <li><a href="#setting-up-your-testing-environment">Setting up your testing environment</a></li>
    <li>
        <a href="#traditional-performance-analysis">Traditional Performance Analysis</a>
        <ul>
        <li><a href="#benchmark-metrics">Benchmark metrics</a></li>
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
mkdir install-mongo-buggy
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

Next, we will apply the git patch fixing the regression and rebuild MongoDB. You can download the patch ([resources/mongo-regression.patch](resources/mongo-regression.patch)) from this repository.
```bash
git apply ../mongo-regression.patch

python3 buildscripts/scons.py \
   DESTDIR=../install-mongo-patched/ \
   install-mongod \
   --disable-warnings-as-errors
```

## Setting up YCSB / MongoDB

Now that we have our database built, we need a benchmarking tool to run queries against the database and measure its performance. We will be using [go-ycsb](https://github.com/pingcap/go-ycsb), a fork of the popular YCSB written in Go, which is designed for benchmarking NoSQL databases. Different applications will need different benchmarking setups, so as always, you'll have to read through the documentation.

Let's clone and build it:
```bash
cd $TUTORIAL_DIR

git clone https://github.com/pingcap/go-ycsb.git
cd go-ycsb
make
cd ..
```

Next, let's initialize the MongoDB database and load some test data. First, we'll create a data storage directory, and then, we'll start the MongoDB server using our configuration file. You can download the configuration file ([resources/mongo-config.yaml](resources/mongo-config.yaml)) from this repository.
```bash
mkdir mongo-data
install-mongo-buggy/bin/mongod --dbpath mongo-data --config mongo-config.yaml
```

> [!TIP]  
> When configuring your application, try to disable as much logging as possible! Logging means writing to disk, which can introduce performance noise as I/O performance is very variable.

Now, download the YCSB workload file ([resources/ycsb-workload](resources/ycsb-workload)) from this repository and run the following command to load test data into MongoDB. This workload file creates 10,000,000 rows, which takes up ~12 GB. When running benchmarks, the entire database must be loaded into memory, so if your machine doesn't have this much memory, you can edit the workload file and reduce `recordcount`.
```bash
go-ycsb/bin/go-ycsb load mongodb -P resources/ycsb-workload --threads $(nproc)
```

## Setting up your testing environment

Before we start running benchmarks, it's important to set up a testing environment that controls for performance variation as much as possible.

### Configuring boot parameters

1. Disabling Intel P-states

Modern CPUs have dynamic frequency scaling, which means they can run at high frequencies when extra performance is needed and run at slower frequencies to save power when the system is idle. However, this can introduce performance variation if the frequency fluctuates while benchmarking. The exact frequency is typically auto-magically determined by the CPU. Our ultimate goal is to lock the CPU at the highest frequency, so our first step is to disable hardware-managed frequency scaling:

> Add the boot option `intel_pstate=disable` and reboot your machine.
> 
> To verify, run `cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_driver` and ensure it prints `acpi-cpufreq`.

2. Disabling the Scheduler Tick

The kernel programs a periodic timer interrupt, called the scheduler tick, to perform various housekeeping tasks, such as tracking the CPU time of a process, handling RCU callbacks, etc. If you are curious, you can check the exact scheduler tick frequency by reading `/proc/config.gz` (with `zless`) and searching for `CONFIG_HZ`, which is usually set to 1000 hz.

Unfortunately, every time the scheduler tick fires, it context switches into the kernel, interrupting the application and trashing the L1/L2 cache. In modern kernels, we can disable the [scheduler tick](https://docs.kernel.org/timers/no_hz.html):

> Add the boot option `nohz_full=X-Y`, where `X-Y` is the range of CPU cores that should be tickless (e.g., `nohz_full=1-13`), and reboot your machine.

> [!IMPORTANT]
> At the time of writing (with Linux v7.0), there is a kernel bug where tickless cores will enter very deep c-states whenever idle, which powers off the L1/L2 cache and severely degrades performance after it wakes up. I documented the bug [here](https://docs.google.com/document/d/1zwAjKgkpWc3fUe1_qqesdu5Z3KvY6PQBXmC4-XeiPN4/edit?tab=t.yfe18cnkn0qy). Long story short, we should disable every c-state except the most shallow one to sidestep this issue:
>
> Add the boot parameter `intel_idle.max_cstate=1` and reboot your machine.

### Configuring runtime parameters

> [!IMPORTANT]
> These steps need to be performed every time you boot your machine, so I recommend creating a script to automate this.

1. Disable simultaneous multithreading (SMT) 

Simultaneous multithreading (which Intel calls Hyperthreading) is a technique where two or more instruction streams are multiplexed on a single physical core, producing multiple logical cores. However, these logical cores share many physical resources, such as L1/L2 cache and execution units. If you run your application on a hyperthread, and another application runs on its sibling hyperthread, your application might experience performance fluctuations as it contends with its sibling.

The simplest solution is to disable SMT through Linux:
```bash
echo off > /sys/devices/system/cpu/smt/control
```

Alternatively, you can sometimes disable SMT through your machine's BIOS setting.

1. Lock CPU frequency to the maximum

```bash
for f in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
  echo performance > $f
done
```

The "performance" CPU governor will simply set the CPU frequency to the maximum and keep it there.

3. Disabling turbo boost

With dynamic overclocking (which Intel calls Turbo Boost), the CPU can temporarily raise its frequency above the normal maximum to boost performance. However, this causes wild spikes in our benchmarking performance, so let's disable it:
```bash
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
```

4. Isolating CPU cores with cgroups

The last thing we want is for the scheduler to place other applications on the same CPU core as MongoDB/YCSB, which will cause performance variation as they compete for CPU time. We will use a Linux kernel feature called cgroups to isolate a set of CPU cores for MongoDB and a set for YCSB. I highly recommend reading the [cgroup documentation](https://docs.kernel.org/admin-guide/cgroup-v2.html). In a nutshell, you'll need to do something like this:

```bash
cd /sys/fs/cgroup

# create a new cgroup called "testing_env"
mkdir testing_env
# we need to enable the "cpuset" controller for this cgroup (read the docs for details)
echo +cpuset > testing_env/cgroup.subtree_control
# configure the cgroup to reserve exclusive access to CPU cores 1-13
echo root > testing_env/cpuset.cpus.partition
echo 1-13 > testing_env/cpuset.cpus.exclusive

cd testing_env

# reserve CPU cores 6-13 for the "mongo" cgroup
mkdir mongo
echo root > mongo/cpuset.cpus.partition
echo 6-13 > mongo/cpuset.cpus.exclusive

# reserve CPU cores 1-5 for the "ycsb" cgroup
mkdir ycsb
echo root > ycsb/cpuset.cpus.partition
echo 1-5 > ycsb/cpuset.cpus.exclusive
```

> [!IMPORTANT]
> On my computer, CPU cores 0-5 are P-cores and 6-13 are E-cores. The exact configuration is unimportant, but if you are reserving a range of cores, make sure that those cores are homogenous (e.g., don't mixing P-cores and E-cores in the same cgroup).

> [!TIP]
> When you want to start benchmarking, create two terminals and add their PIDs to the corresponding cgroups:
> ```bash
> echo <TERM1_PID> > testing_env/mongo/cgroup.procs
> echo <TERM2_PID> > testing_env/ycsb/cgroup.procs
> ```
> That way, any process launched from those terminals, like MongoDB / YCSB, is automatically added to the corresponding cgroup.

## Traditional Performance Analysis

### Benchmark metrics

### Flamegraphs

### Function latency instrumentation

## pt-fuser Workflow

### Collecting, filtering, and merging traces

### Visualizing and comparing traces

## Takeaways
