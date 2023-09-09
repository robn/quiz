# quiz - the [q]em[u] [i]nteractive [z]fs environment

(Look, names are hard, and this one I found at least mildly amusing. Just go with it.)

**quiz** is a program to support fast edit-compile-test cycles for Linux kernel development, with a focus on OpenZFS. At its heart its a qemu-based microvm, stripped down to the essentials to setup and boot a VM and start a test program in under a second.

## Motivation

My preferred programming style is highly interactive and exploratory, which requires a fast edit-compile-test cycle. But then I started working I work on [OpenZFS](https://openzfs.org), which is a big chunk of kernel code. Kernel code means developing on either real hardware or a VM, which tend to be awkward to get real code onto, and then, being kernel code, its pretty easy to crash, wedge or otherwise damage the kernel, requiring a reboot.

I wanted a way to work on OpenZFS just like I’d work on any other program - make a change, compile it, run it and see what happens. I happened to know that microVMs were getting to the point where booting in a fraction of a second was possible, so I decided to see if I could use one for this. **quiz** is what I came up with.

## Note

This is still pretty rough; you need to be willing to get your hands dirty and customise things, and its still firmly in the experimental stage: I change how things work pretty often, as I understand better what I need.

That said, I'm using this to spin up tens and sometimes hundreds of VMs every day, doing Real Programming and furiously creating value for the shareholders. It does work and is safe to play with!

Just don't treat anything here as more than "highly experimental".

## Demo

Simple boot to prompt:

[![quiz boot](demo/boot.gif)](https://asciinema.org/a/gQAwKSBnaDiZ6ZiO6ocaARd61)

Creating a ZFS pool over memory-backed devices:

[![quiz ZFS pool create](demo/zfs.gif)](https://asciinema.org/a/ZtZvX6MzK5y7HYsuEE7STaEbn)

Running (part of) the ZFS test suite:

[![quiz ZFS test suite run](demo/zfs-test.gif)](https://asciinema.org/a/kZ5TeTaqH7aU2xuJHSdDDu4at)

## Setup

### Install prerequisites

On a Debian 12 (bookworm) system you’ll need at least:

* `qemu-system-x86`
* `mmdebstrap`
* kernel build deps

### Build the kernel

Run `quiz-prepare-kernel`. This will grab kernel source, compile it with a very minimal config to support only what’s needed for the microvm, and install it into the system dir.

### Build the root image

Run `quiz-prepare-root`. This will generate a barely-there Debian root image.

### Build the work dir

Run `quiz-prepare-work`. This sets up a chunk of host filesystem that will be mapped into the VM, and is where the OpenZFS modules and related development programs can be stored.

### Build OpenZFS

The configure and install commands are quite involved to get it to properly target the work dir. The `quiz-build-zfs` wrapper is there to help with this.

From an OpenZFS checkout:

```
$ ./autogen.sh
$ ~/quiz/quiz-build-zfs configure --enable-debug --enable-debuginfo # your configure options here
$ make -j5
$ ~/quiz/quiz-build-zfs make install
```

### Run it

Simple boot to a shell:

```
$ ./quiz
[quiz] 20230909-23:35:00 starting microvm
[    0.318686] brd: module loaded
[    0.445527] loop: module loaded
[    0.451687] virtio_blk virtio2: [vda] 1608310 512-byte logical blocks (823 MB/785 MiB)
...
[    0.520927] Run /sbin/init as init process
[    0.640503] Using default interface naming scheme 'v252'.
[    0.851721] quiz: starting user program
[INFO  tini (1)] Spawned child process '/bin/bash' with pid '511'
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
root@quiz:/#
```

or run a program directly:

```
$ ./quiz uname -a
[quiz] 20230909-23:35:37 creating run script
[quiz] 20230909-23:35:37 starting microvm
[    0.317245] brd: module loaded
[    0.456128] loop: module loaded
[    0.463744] virtio_blk virtio2: [vda] 1608310 512-byte logical blocks (823 MB/785 MiB)
...
[    0.523497] Run /sbin/init as init process
[    0.714328] Using default interface naming scheme 'v252'.
[    0.913468] quiz: starting user program
[INFO  tini (1)] Spawned child process '/bin/sh' with pid '511'
+ uname -a
Linux quiz 5.10.170 #3 SMP Wed Jun 21 15:52:50 AEST 2023 x86_64 GNU/Linux
[INFO  tini (1)] Main child exited normally (with status '0')
```

Include some profiles:

```
$ ./quiz -p zfs zpool status
$ ./quiz -p zfs zpool status
[quiz] 20230909-23:41:05 including profile: zfs
[quiz] 20230909-23:41:05 creating run script
[quiz] 20230909-23:41:05 starting microvm
[    0.318037] brd: module loaded
[    0.444730] loop: module loaded
[    0.452162] virtio_blk virtio2: [vda] 1608310 512-byte logical blocks (823 MB/785 MiB)
...
[    0.886639] quiz: starting profile init: zfs
[    1.075146] spl: loading out-of-tree module taints kernel.
[    2.223443] zfs: module license 'CDDL' taints kernel.
[    2.223489] Disabling lock debugging due to kernel taint
[    3.145029] ZFS: Loaded module v2.2.99-62_g28fdf9f64 (DEBUG mode), ZFS pool version 5000, ZFS filesystem version 5
[    3.146129] quiz: starting user program
[INFO  tini (1)] Spawned child process '/bin/sh' with pid '526'
+ zpool status
no pools available
[INFO  tini (1)] Main child exited normally (with status '0')
```

## Config

Some amount of config can be done by creating a `quiz.cfg`. Defaults are set in `quiz-config` (which loads `quiz.cfg` to override them). The core scripts and profiles are configured this way.

## Profiles

Profiles are quiz's extension mechanism. They introduce additional setup commands, config files or even whole programs into the VM as its being generated. The idea is that you can just enable the bits you need as you need them, to keep the boot time as small as possible for any given task.

Profiles are enabled with the `-p` switch to `quiz`, or via the `QUIZ_PROFILE` config variable.

The following profiles exist:

* `blockdev`: creates some block devices, backed by files on the host and exposed to the VM through `virtio-blk`. They will appear as `/dev/quizbX`.
* `memdev`: creates some block devices, backed by files on `tmpfs` inside the guest. They will appear as `/dev/quizm0`.
* `zfs`: runs `depmod` to ensure the latest modules in the work dir have been, and then loads them into the VM kernel as part of the boot sequence.
* `ztest`: adds a couple of hacks into the VM to allow the ZFS test suite to run properly.

Profiles are in the `profile/` dir, each in their own named dir. There's three main hooks available:

* If `setup` exists, it will be sourced by the `quiz` program before the VM is started (that is, it runs on the host)
* If `init` exists, it will be sourced by the `init2` second-stage boot inside the VM, before control is handed to the shell or user program
* Any other files will be copied into the work dir at the given paths, and so be available inside the VM after it boots

Note that profiles currently have no way to influence the kernel config or the root filesystem, which is a problem if a profile needs something from it. I haven't decided what, if anything, to do about this yet.

## Todo

* more profiles
  * persistent blockdevs
  * mdadm stuff (fake error environments)
  * pool create/import
* a good way to get stuff back to the host (logs, test artifacts, etc)
* multiple terminal support
  * right now there's only thing; even finding a way to run tmux inside would probably do
* multi-instance
  * this should really be something you use inside your OpenZFS worktree, so that they don't trample on each other
* cleaner lines of integration with host
  * there's the place to install to, and then profile-generated stuff, and the internal .quiz/ work area, and then there's stuff the user wants to persist across runs, and further stuff that the user wants to bring into every VM (eg test programs). its not clear what should go where yet
* just one program
  * it should just be `quiz foo`, and be on your path
* more architectures
  * anything qemu supports should be possible. in particular, I would like a big-endian architecture
* FreeBSD host support (ie bhyve instead of kvm)
  * qemu may not have support, so either make that happen (!) or find a different monitor package
* FreeBSD guest support
  * wanted for OpenZFS dev on FreeBSD. [recent work on boot times in microvms](https://www.usenix.org/publications/loginonline/freebsd-firecracker) make this both plausible and attractive
