Maintainer: Icenowy Zheng <icenowy@aosc.xyz>

Thanks to:

- PaX/Grsecurity
- Mempo project
- Debian GNU/Linux Community
- Shawn C[a.k.a "Citypw"]
- Linux From Scratch

Copyright (c) TYA infotech ltd http://tya.company/

# Reproducible builds for PaX/Grsecurity

These scripts are intended to do reproducible builds for Linux kernel with Grsecurity patch set.

## Dependencies

The kernel building script will need the standard kernel building dependencies to be install.

### Debian-derived distributions

Use the command below to install the dependencies needed.

```
sudo apt-get install build-essential bc flex bison gnupg
```

## Basic Usage

To build the kernel deterministically, a certain kernel build directory is necessary. Currently /kbuild is chosen to be the fixed directory. So you should at first create it and grant rwx permission of the directory for the UNIX user you used to build the kernel.

As a directory under /, root access will be needed to create this directory.

```
sudo mkdir /kbuild
# Assume we're using the kernelbuild user
sudo chown kernelbuild /kbuild
```

Most of the source tarballs downloaded by the script is signed by GnuPG. If you do not have the necessary GPG public key imported to verify the signature of GNU things and the Linux Kernel, you can run

```
./import-keys.sh
```

or you can just set VERIFY_GPG environment variable to 0, thus signature verifying will be disabled. (IT'S NOT RECOMMENDED!)

After preparing the directory and have the keys imported, you can place a kernel config file named "config" in this directory, and then just run:

```
./run.sh
```

Some kernel configs modified to enable PaX and being deterministic is placed under configs/, include:

- configs/paxed-allnoconfig: an all-no config with PaX and module support enabled, only for testing purpose.

- configs/paxed-defconfig: defconfig with PaX enabled, can be used as a basis to customify the config.

- configs/paxed-mint-config: a config file from Linux Mint 18 with PaX enabled, can be directly used on Debian-derived distributions without modification.

Then the output kernel (bzImage, vmlinux, modules, DPKG packages and build fingerprint) is located at out/

## Reproduce

To reproduce a kernel build, there's some ways shown below:

- Manually extract the fingerprint.sh and config from the build, and then use run.sh. (run.sh will check whether a fingerprint.sh or config exists)

- Use the debian package prefixed linux-image- as the parameter of run.sh. (The image should be one generated by the script, otherwise it won't do a reproduce)

- To just test build-kernel.sh, the script named "try-reproduce.sh" can be used. It will automatically run "build-kernel.sh" twice, and check the results. (NOTE: It requires the toolchain to be built at first.)

## DPKG packages notice

Currently, we cannot ensure DPKG packages to be reproducible. However, we can promise the content of all the packages are reproducible.

A shell script named "deb-diff.sh" is present to compare the content of two DPKG packages. It will simply extract files from the package, and then use "diff" command to check the difference of them.

## Config options to be noted

### CONFIG_MODULE_SIG (Module signature verification)

When this option is enabled, there will be a key embedded into the kernel, which is used to sign modules.

The key is either generated at build time (from /dev/random, which made the key not reproducible), or pre-generated (which made the signing system useless, as the key-pair will be provided to anyone who wants to verify the build).

Although support for pre-generated key-pair can be implemented, it's not implemented now.

So the option should be *DISABLED* now.

**TODO**: support for external pre-generated key-pair.

### CONFIG_PAX_LATENT_ENTROPY (Generate some entropy during boot and runtime)

When this option is enabled, the generated binary code will contain some random bits generated by GCC at build time, as entropy.

Enabling this option will lead to irreproducible builds.

So the option should be *DISABLED* now.

### CONFIG_GRKERNSEC_RANDSTRUCT (Randomize layout of sensitive kernel structures)

When this option is enabled, the generated binary will have sensitive kernel structures randomized.

It uses a seed from /dev/urandom at build time, however, currently the scripts have already hacked the seed generation process. Now the seed is part of the build fingerprint.

So the option is now *safe to ENABLE*.

## Out-of-tree kernel modules notes

As the build system used a "x86_64 to x86_64 cross-compiler", the modules cannot be built with the host compiler.

So if an out-of-tree module needs to be built, you should use the linux kernel source tree under /kbuild/linux-4.6.5, and add "CROSS_COMPILE=/kbuild/tools/bin/x86_64-kernelonly-linux-gnu-" argument to the make command.

For example, to build the acpi-call kernel module (which uses KDIR variable to indicate the kernel source tree):

```
acpi-call-1.1.0 # make KDIR=/kbuild/linux-4.6.5 CROSS_COMPILE=/kbuild/tools/bin/x86_64-kernelonly-linux-gnu-
```

**TODO**: add a framework for easily building of out-of-tree modules.

## Reference
- http://www.dwheeler.com/trusting-trust/
- https://github.com/mempo/mempo-kernel
- https://wiki.debian.org/Mempo
- https://wiki.debian.org/ReproducibleBuilds
