---
layout: post
title: Installing DINAMITE natively. A Linux example.
excerpt_separator: <!--more-->
---

In this blog post we describe how to install DINAMITE natively. We
describe the installation steps we used on a Fedora Linux system.
Similar steps work on OS X.
<!--more-->

## Overview

DINAMITE depends on LLVM 3.5, its sources and some of its tools.

The easiest way to get both LLVM and DINAMITE installed is to use
Docker with a container that can be found here:
[here](https://github.com/dinamite-toolkit/dinamite-compiler-docker.git).

If you don't like Docker we describe how to install LLVM 3.5 and
DINAMITE from scratch.

We describe the steps that we followed on a RedHat Fedora Linux system
([Amazon LinuxAMI](https://aws.amazon.com/amazon-linux-ami/2016.09-release-notes/).

(We also sucessfully performed manual installation on OS X (Darwin). See a
MacOS note at the end of the article.)

## Installation

Follow the steps below or just paste them into a text file and run
them as a script!

We assume that your DINAMITE installation lives in the
`~/Work/DINAMITE/LLVM` directory.

```shell
# Create a root directory for your DINAMITE installation
mkdir ~/Work/DINAMITE/LLVM
cd ~/Work/DINAMITE/LLVM

# Download LLVM sources
wget http://releases.llvm.org/3.5.0/llvm-3.5.0.src.tar.xz;
wget http://releases.llvm.org/3.5.0/compiler-rt-3.5.0.src.tar.xz;
wget http://releases.llvm.org/3.5.0/cfe-3.5.0.src.tar.xz;
wget http://releases.llvm.org/3.5.0/clang-tools-extra-3.5.0.src.tar.xz;

# Unpack everything, put it in the right place
# Get rid of unneeded files
tar xf llvm-3.5.0.src.tar.xz;
rm llvm-3.5.0.src.tar.xz;

tar xf cfe-3.5.0.src.tar.xz;
rm cfe-3.5.0.src.tar.xz;
mv cfe-3.5.0.src llvm-3.5.0.src/tools/clang;

tar xf clang-tools-extra-3.5.0.src.tar.xz;
rm clang-tools-extra-3.5.0.src.tar.xz;
mv clang-tools-extra-3.5.0.src llvm-3.5.0.src/tools/clang/tools/extra;

tar xf compiler-rt-3.5.0.src.tar.xz;
rm compiler-rt-3.5.0.src.tar.xz;
mv compiler-rt-3.5.0.src llvm-3.5.0.src/projects/compiler-rt;

# Apply a patch
cd llvm-3.5.0.src;
wget https://raw.githubusercontent.com/dinamite-toolkit/dinamite-compiler-docker/master/dinamite_clang_gcc6.patch;
patch -p0 < dinamite_clang_gcc6.patch;

# Build the LLVM
cd ~/Work/DINAMITE/LLVM;
mkdir build;
cd build;
cmake -G "Unix Makefiles" ../llvm-3.5.0.src;
make -j 4;

# Run configure to set up the right variables and paths.
cd ~/Work/DINAMITE/LLVM/llvm-3.5.0.src;

./configure;

# Set the environment variables
echo "export PATH=$HOME/Work/DINAMITE/LLVM/build/bin:$PATH" >> ~/.bashrc;
echo "export LLVM_SOURCE=$HOME/Work/DINAMITE/LLVM/llvm-3.5.0.src" >> ~/.bashrc;
echo "export LLVM_BUILD=$HOME/Work/DINAMITE/LLVM/build" >> ~/.bashrc;
echo "export DIN_ROOT=$HOME/Work/DINAMITE/LLVM/llvm-3.5.0.src/projects/dinamite" >> ~/.bashrc;
echo "export INST_LIB=$HOME/Work/DINAMITE/LLVM/llvm-3.5.0.src/projects/dinamite/library" >> ~/.bashrc;
echo "export DIN_CC=\"clang -g -Xclang -load -Xclang $HOME/Work/DINAMITE/LLVM/llvm-3.5.0.src/Release+Asserts/lib/AccessInstrument.so\"" >> ~/.bashrc;
echo "export DIN_CXX=\"clang++ -g -Xclang -load -Xclang $HOME/Work/DINAMITE/LLVM/llvm-3.5.0.src/Release+Asserts/lib/AccessInstrument.so\"" >> ~/.bashrc;
echo "export DIN_LDFLAGS=\"-L/$HOME/Work/DINAMITE/LLVM/llvm-3.5.0.src/projects/dinamite/library -linstrumentation -lpthread\"" >> ~/.bashrc;
echo "export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/Work/DINAMITE/LLVM/llvm-3.5.0.src/projects/dinamite/library" >> ~/.bashrc;

# And make them active
. ~/.bashrc;

# Make sure you are using the clang you just installed by default
# The output should be:
# ~/Work/DINAMITE/LLVM/build/bin/clang
which clang;

# Clone the DINAMITE repository and build
cd ~/Work/DINAMITE/LLVM/llvm-3.5.0.src/projects;
git clone https://bitbucket.org/datamancers/dinamite.git;

cd dinamite/library;
make binary;
cd ..;
make

# Test that you can build and run instrumented executables
./cbuild.sh
./crun.sh
```

## Common errors:

**Various linker errors**

By far the most common cause of errors is caused by compiling with a
wrong version of `clang`. You should be compiling with the `clang`
that you just installed. Check that this is so:

   ```shell
   $ which clang
   ```
The output should be:

   ```shell
   ~/Work/DINAMITE/LLVM/build/bin/clang
   ```

If it is not, put the directory
`$HOME/Work/DINAMITE/LLVM/build/bin/clang` at the beginning of your
`$PATH` variable. See "Set the environment variables step" above.

If you do not want the newly installed clang to be default, then you'd
need to specify a full path for the clang you installed whenever you
compile with DINAMITE.

**Your compiler complains that it cannot find gcc installation files**

For example:

   ```shell
   /usr/bin/ld: cannot find crtbegin.o: No such file or directory
   ```

This is a [known issue](http://stackoverflow.com/questions/4160262/clang-linker-problem) with LLVM 3.5.

To work around it, you need to tell clang the path to your compiler
installation. You can find where your gcc is installed by searching
for the location of the missing file, e.g., `crtbegin.o`. For example
if that location happened to be
`/usr/lib/gcc/x86_64-amazon-linux/4.8.3`, open `cbuild.sh` (or your
Makefile) and add the following directive to you compilation command
(prior to "clang"):

   ```shell
   COMPILER_PATH=/usr/lib/gcc/x86_64-amazon-linux/4.8.3
   ```
**The compilation complains that it cannot find gcc libraries**

For example:

   ```shell
   /usr/bin/ld: cannot find -lgcc
   /usr/bin/ld: cannot find -lgcc_s
   ```

Add the location of your gcc libraries as the library path in  `cbuild.sh` (or your Makefile). For instance, if your compiler path is `/usr/lib/gcc/x86_64-amazon-linux/4.8.3`, add the following to the compilation command:

   ```shell
   -L/usr/lib/gcc/x86_64-amazon-linux/4.8.3
   ```
An example with these changes can be found in
`~/Work/DINAMITE/LLVM/llvm-3.5.0.src/projects/dinamite/cbuild-gccnotfound-workaround.sh`.


## MacOS note:

We are investigating a strange anomaly on OS X, where the main
compiler pass would only build with the built-in system clang or g++,
while the instrumented programs would only build with the clang
included in the LLVM distribution. To work around it, do the
following:

Modify the LLVM build steps shown above as follows:

```shell
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=/usr/local ../llvm-3.5.0.src;
make -j 4;
cd ~/Work/DINAMITE/LLVM/llvm-3.5.0.src;
CXX=g++ ./configure
```


This way, the instrumentation pass itself will be built using g++, but
   the instrumented programs will be built using the clang that you
   just installed, which would be placed in `/usr/local`.
