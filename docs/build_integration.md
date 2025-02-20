---
layout: default
parent: ClusterFuzzLite
title: >
  Step 1: Build Integration
has_children: false
nav_order: 3
permalink: /build-integration/
---
# Build integration
{: .no_toc}

- TOC
{:toc}
---

This page explains how to integrate your project with ClusterFuzzLite's build
system so that ClusterFuzzLite can build your project's fuzz targets with
sanitizers.
ClusterFuzzLite is intimately tied to sanitizers and libFuzzer. By integrating
with our build system, ClusterFuzzLite will be able to use the most recent
versions of these tools to secure your code.

By the end of the document you will be able to build and run your fuzz targets
with libFuzzer and a variety of sanitizers.

## Prerequisites

ClusterFuzzLite supports [libFuzzer targets] built with Clang on Linux.

ClusterFuzzLite reuses the [OSS-Fuzz] toolchain to make building easier. This
means that ClusterFuzzLite will build your project in a docker container.
If you are familiar with OSS-Fuzz, most of the concepts here are exactly the
same, with one key difference. Rather than checking out the source code in the
[`Dockerfile`](#dockerfile) using `git clone`, the `Dockerfile` copies in the
source code directly during `docker build`. Another minor difference is that
ClusterFuzzLite only supports libFuzzer and not other fuzzing engines.
If you are not familiar with OSS-Fuzz, have no fear! This document is written
with you in mind and assumes no knowledge of OSS-Fuzz.

Before you can start setting up your new project for fuzzing, you must do the
following to use the ClusterFuzzLite toolchain:
- Integrate [fuzz targets] with your codebase. See
  [this page](https://github.com/google/fuzzing/blob/master/docs/good-fuzz-target.md)
  for more details.

- [Install Docker](https://docs.docker.com/engine/installation)

  If you want to run `docker` without `sudo`, you can
  [create a docker group](https://docs.docker.com/engine/installation/linux/ubuntulinux/#/create-a-docker-group).

  **Note:** Docker images can consume significant disk space. Run
  [docker-cleanup](https://gist.github.com/mikea/d23a839cba68778d94e0302e8a2c200f)
  periodically to garbage-collect unused images.

- Clone the OSS-Fuzz repo: `git clone https://github.com/google/oss-fuzz.git`

## Generating an empty build integration

Next you need to configure your project to build fuzzers on ClusterFuzzLite.
To do this, your project needs three configuration files in the
`.clusterfuzzlite` directory in your project's root:
* [.clusterfuzzlite/project.yaml](#projectyaml) - provides metadata about the project.
* [.clusterfuzzlite/Dockerfile](#dockerfile) - defines the container environment with information
on dependencies needed to build the project and its [fuzz targets].
* [.clusterfuzzlite/build.sh](#buildsh) - defines the build script that executes
inside the Docker container and builds your project and its fuzz targets.

You can generate empty versions of these files with the following command:

```bash
$ cd /path/to/oss-fuzz
$ export PATH_TO_PROJECT=<path_to_your_project>
$ python infra/helper.py generate --external --language=c++ $PATH_TO_PROJECT
```
Note that you may need to change the `--language` argument to another value
if your project is written in another language.
This is discussed more in the [language section](#language).

Once the configuration files are generated, you should modify them to fit your
project. Let's look at each file one-by-one and explain what you should add to
them.

## project.yaml {#projectyaml}

This configuration file stores project metadata. Currently it is only used by
`helper.py` to build your project.
The only field you must fill out in this file is:

### language

Programming language the project is written in. Values you can specify include:

* `c`
* `c++`
* [`go`]({{ site.baseurl }}//build-integration/go-lang/)
* [`rust`]({{ site.baseurl }}//build-integration/rust-lang/)
* [`python`]({{ site.baseurl }}//build-integration/python-lang/)
* [`jvm`]({{ site.baseurl }}//build-integration/jvm-lang/)
  * This should be used for Java, Kotlin, Scala and other JVM-based languages.
* [`swift`]({{ site.baseurl }}//build-integration/swift-lang/)

Most of this guide applies directly to C/C++ projects. Please see the relevant
subguides for how to build fuzzers for that language.
Note that `c` and `c++` are the same to ClusterFuzzLite.

## Dockerfile {#dockerfile}

This integration file defines the Docker image for building your project.
Your [build.sh](#buildsh) script will be executed inside the image this file
defines.
For most projects, the Dockerfile is simple:
```docker
FROM gcr.io/oss-fuzz-base/base-builder:v1       # Base image with clang toolchain
RUN apt-get update && apt-get install -y ...    # Install required packages to build your project.
COPY . $SRC/<project_name>                      # Copy your project's source code.
WORKDIR $SRC/<project_name>                     # Working directory for build.sh.
COPY ./clusterfuzzlite/build.sh $SRC/           # Copy build.sh into $SRC dir.
```

See
[here](https://github.com/oliverchang/curl/blob/master/.clusterfuzzlite/Dockerfile)
for an example.

## build.sh {#buildsh}

This script must build binaries for [fuzz targets] in your project.
The script is executed within the image built from your [Dockerfile](#dockerfile).

In general, this script should do the following:

- Build the project using your build system with ClusterFuzzLite's compiler.
- Provide ClusterFuzzLite's compiler flags (defined as [environment variables](#compilation-env)) to the build system.
- Build your [fuzz targets]
  and link them with the `$LIB_FUZZING_ENGINE` (libFuzzer) environment variable.
- Place any fuzz target binaries in the directory defined by the environment variable `$OUT`.

Make sure that the binary names for your [fuzz targets]
contain only alphanumeric characters, underscore (`_`) or dash (`-`). They
should not contain periods (`.`) or have file extensions. Otherwise, they won't
run.
Your build.sh should not delete any source code. Source code is needed for code
coverage reports.

The `$WORK` environment variable defines a directory where build.sh can store
intermediate files.

Here's an example build.sh from Expat:

```bash
#!/bin/bash -eu

./buildconf.sh
# configure scripts usually use correct environment variables.
./configure

make clean
make -j$(nproc) all

$CXX $CXXFLAGS -std=c++11 -Ilib/ \
    $SRC/parse_fuzzer.cc -o $OUT/parse_fuzzer \
    $LIB_FUZZING_ENGINE .libs/libexpat.a

# Optional: Copy dictionaries and options files.
cp $SRC/*.dict $SRC/*.options $OUT/
```

### build.sh environment variables for compilation {#compilation-env}

You *must* use ClusterFuzzLite's compilers and compiler flags to build your fuzz
targets.
These are provided in the following environment variables:

| Env Variable           | Description
| -------------          | --------
| `$CC`, `$CXX`, `$CCC`  | C and C++ compilers.
| `$CFLAGS`, `$CXXFLAGS` | C and C++ compiler flags.
| `$LIB_FUZZING_ENGINE`  | C++ compiler argument to link fuzz target against libFuzzer.

These compiler flags are needed to properly instrument your fuzzers with
sanitizers and coverage instrumentation.

Note that even if your project is written in pure C you *must* use `$CXX` to
link your fuzz target binaries.

Many build tools will automatically use these environment variables (with the
exception of `$LIB_FUZZING_ENGINE`). If not, pass them manually to the build
tool.

You can also do the final linking step with `$LIB_FUZZING_ENGINE` in your
`build.sh`.

See the [Provided Environment Variables](https://github.com/google/oss-fuzz/blob/master/infra/base-images/base-builder/README.md#provided-environment-variables)
page in OSS-Fuzz's `base-builder` image documentation for more details on
environment variables that are available to `build.sh`.

## Fuzzer execution environment

You should not make any assumptions on the availability of dependent packages
in the execution environment and the built fuzzers should have dependencies
statically linked.

## Testing locally

When you have completed writing the build.sh and Dockerfile, you should test
that they work.
This includes running your fuzz targets, which we strongly recommend.
The helper.py script you used to generate your config files offers a few different ways of doing this:
1. Build your docker image and [fuzz targets]:

    ```bash
    $ python infra/helper.py build_image --external $PATH_TO_PROJECT
    $ python infra/helper.py build_fuzzers --external $PATH_TO_PROJECT --sanitizer <address/undefined/memory>
    ```
    The built binaries appear in the `/path/to/oss-fuzz/build/out/$PROJECT_NAME`
    directory on your host machine (and `$OUT` in the container). Note that
    `$PROJECT_NAME` is the name of the root directory of your project (e.g. if
    `$PATH_TO_PROJECT` is `/path/to/systemd`, `$PROJECT_NAME` is `systemd`).

2. Find common build issues to fix by running the `check_build` command:

    ```bash
    $ python infra/helper.py check_build --external $PATH_TO_PROJECT --sanitizer <address/undefined/memory>
    ```
    This checks that your fuzz targets are compiled with the right sanitizer and don't crash after fuzzing for a few seconds.

3. To run a particular fuzz target, use `run_fuzzer`:

    ```bash
    $ python infra/helper.py run_fuzzer --external --corpus-dir=<path-to-temp-corpus-dir> $PATH_TO_PROJECT <fuzz_target>
    ```
4. If you are going to use the code coverage report feature of ClusterFuzzLite
it is a good idea to test that coverage report generation works. This would use
the corpus generated from the previous `run_fuzzer` step in your local corpus
directory.

    ```bash
    $ python infra/helper.py build_fuzzers --external --sanitizer coverage $PATH_TO_PROJECT
    $ python infra/helper.py coverage --external $PATH_TO_PROJECT --fuzz-target=<fuzz_target> --corpus-dir=<path-to-temp-corpus-dir>
    ```

You may need to run `python infra/helper.py pull_images` to use the latest coverage tools.

<b>Make sure to test each
of the sanitizers with `build_fuzzers`, `check_build`, and `run_fuzzer`.</b>


If everything works locally, it should also work on ClusterFuzzLite.
If you experience failures running fuzzers on ClusterFuzzLite, review your
[dependencies](https://google.github.io/oss-fuzz/further-reading/fuzzer-environment/).

## Debugging problems

If you run into problems, the
[Debugging page](https://google.github.io/oss-fuzz/advanced-topics/debugging/)
lists ways to debug your build scripts and
[fuzz targets].

## Efficient fuzzing

To improve your fuzz target ability to find bugs faster, please read
[this section](https://google.github.io/oss-fuzz/getting-started/new-project-guide/#efficient-fuzzing).

## Running ClusterFuzzLite

Next: [Step 2: Running ClusterFuzzLite] for directions on setting up ClusterFuzzLite to run on your CI.

[fuzz targets]: https://github.com/google/fuzzing/blob/master/docs/glossary.md#fuzz-target
[libFuzzer targets]: https://github.com/google/fuzzing/blob/master/docs/glossary.md#fuzz-target
[OSS-Fuzz]: https://github.com/google/oss-fuzz
[`Dockerfile`]: #dockerfile
[Step 2: Running ClusterFuzzLite]: {{ site.baseurl}}/running-clusterfuzzlite/
