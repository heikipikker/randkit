# randkit
 Random number rootkit for the Linux kernel

The goal is to see how a kernel module can access and modify the /dev/urandom and /dev/random structures.

Contents
=

zero/
-

A simple module that replaces /dev/(u)random with /dev/zero

xor128/
-

A module that replaces /dev/(u)random with a simple predictable and reversible PRNG.

This PRNG is called *xor128* and is part of the very fast Xorshift PRNGs: https://en.wikipedia.org/wiki/Xorshift

One of the *xor128* features is that it passes the *diehard* random number tests: https://en.wikipedia.org/wiki/Diehard_tests

Therefore it is difficult to distinguish a predictable PRNG like *xor128* with a **really** random CSPRNG like `/dev/urandom`.

fops/
-

A module that checks that the `struct file_operations` pointer from `/dev/urandom` is accessible using different techniques

tests/
-

Programs to test that the modules work correctly (e.g. read random numbers from the file descriptor, syscall, ...).

examples/
-

Shows how to retrieve past random numbers generated by the `randkit_xor128` module.

Build
=

Go inside the `zero/` or `xor128/` directories and type `make`.

You need the linux headers for your kernel.
