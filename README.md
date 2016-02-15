# randkit
 Random number rootkit for the Linux kernel

The goals are to see:
* how a kernel module can access and modify the `/dev/urandom` and `/dev/random` structures.
* what happens when we replace a cryptographically secure PRNG with a predictable and reversible PRNG (here, this simple PRNG is `xor128`).

Contents
=

zero/
-

A simple module that replaces `/dev/(u)random` with `/dev/zero`

```bash
# output 16 random bytes from /dev/urandom
$ head -c16 /dev/urandom | hexdump
0000000 55a4 98ea 7179 4179 d634 fd92 89f2 898e
0000010
# load the zero rootkit
$ sudo insmod randkit_zero.ko
# output 16 zero bytes from /dev/urandom
$ head -c16 /dev/urandom | hexdump
0000000 0000 0000 0000 0000 0000 0000 0000 0000
0000010
# remove the rootkit
$ sudo rmmod randkit_zero
# the random numbers are random again!
$ head -c16 /dev/urandom | hexdump
0000000 194b 2c9f 5054 113b f6bc 5ab8 3ed9 dee2
0000010
```

xor128/
-

A module that replaces `/dev/(u)random` with a simple predictable and reversible PRNG.

This PRNG is called *xor128* and is part of the very fast Xorshift PRNGs: https://en.wikipedia.org/wiki/Xorshift

One of the *xor128* features is that it passes the *diehard* random number tests: https://en.wikipedia.org/wiki/Diehard_tests (which is a rather complete randomness test suite).

Let's do a quick randomness test with another randomness test suite. We will use the `ent` tool available at https://www.fourmilab.ch/random/

For the real `/dev/urandom` device:

```bash
# generate 5 MB of random data from the real /dev/urandom
$ dd if=/dev/urandom of=random5MB.urandom bs=1M count=5
5+0 records in
5+0 records out
5242880 bytes (5.2 MB) copied, 0.406672 s, 12.9 MB/s
# test the randomness of the data using the ent tool
$ ent random5MB.urandom
Entropy = 7.999968 bits per byte.

Optimum compression would reduce the size
of this 5242880 byte file by 0 percent.

Chi square distribution for 5242880 samples is 233.65, and randomly
would exceed this value 75.00 percent of the times.

Arithmetic mean value of data bytes is 127.4776 (127.5 = random).
Monte Carlo value for Pi is 3.143720682 (error 0.07 percent).
Serial correlation coefficient is -0.000062 (totally uncorrelated = 0.0).
```

For the backdoored `/dev/urandom` device:

```bash
# load the rootkit
$ sudo insmod randkit_xor128.ko
# generate 5 MB of random data from the backdoored /dev/urandom
$ dd if=/dev/urandom of=random5MB.xor128 bs=1M count=5
5+0 records in
5+0 records out
5242880 bytes (5.2 MB) copied, 0.0266334 s, 197 MB/s
# test the randomness of the data using the ent tool
$ ent random5MB.xor128 
Entropy = 7.999964 bits per byte.

Optimum compression would reduce the size
of this 5242880 byte file by 0 percent.

Chi square distribution for 5242880 samples is 261.73, and randomly
would exceed this value 50.00 percent of the times.

Arithmetic mean value of data bytes is 127.4943 (127.5 = random).
Monte Carlo value for Pi is 3.140932900 (error 0.02 percent).
Serial correlation coefficient is 0.000238 (totally uncorrelated = 0.0).
```

Both the files appear to be random. Therefore it is difficult to distinguish a predictable PRNG like *xor128* with a **really** random CSPRNG like `/dev/urandom`.

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

Go inside the `fops/`, `zero/` or `xor128/` directories and type `make`.

You need the linux headers for your kernel.

Test
=

Go to the `examples/` directory and type `make`.

This will encrypt data using a key generated by the backdoored PRNG. Then it will delete the key and try to decrypt the data by retrieving the PRNG previous values.

```bash
$ cd examples
$ make
./example.sh
[*] reloading the module
[*] cleaning files from previous run
[*] generating 5KB of random numbers
1+0 records in
1+0 records out
5120 bytes (5.1 kB) copied, 6.9455e-05 s, 73.7 MB/s
[*] generating the GPG symmetric key
[*] encrypting data
[*] deleting the key
[*] generating 5KB of random numbers again
1+0 records in
1+0 records out
5120 bytes (5.1 kB) copied, 6.6115e-05 s, 77.4 MB/s
[*] decrypt the data by reversing the PRNG to retrieve the key
[*] this should take approx. 1280 iterations
[*] iteration: 1
gpg: decryption failed: bad key
[*] bad key. Retrying with previous state...
[*] iteration: 2
gpg: decryption failed: bad key
[*] bad key. Retrying with previous state...
[*] iteration: 3
gpg: decryption failed: bad key
[*] bad key. Retrying with previous state...
...
[*] iteration: 1287
gpg: decryption failed: bad key
[*] bad key. Retrying with previous state...
[*] iteration: 1288
gpg: decryption failed: bad key
[*] bad key. Retrying with previous state...
[*] iteration: 1289
[*] key found!
[*] comparing the original and decrypted files:
[*] Success. Files are equal!
```