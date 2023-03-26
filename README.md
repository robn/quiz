# quiz - the [q]em[u] [i]nteractive [z]fs environment

(Look, names are hard, and this one I found at least mildly amusing. Just go with it.)

**quiz** is a program to support fast edit-compile-test cycles for Linux kernel development, with a focus on OpenZFS. At its heart its a qemu-based microvm, stripped down to the essentials to setup and boot a VM and start a test program in under a second.

## Motivation

My preferred programming style is highly interactive and exploratory, which requires a fast edit-compile-test cycle. But then I started working I work on [OpenZFS](https://openzfs.org), which is a big chunk of kernel code. Kernel code means developing on either real hardware or a VM, which tend to be awkward to get real code onto, and then, being kernel code, its pretty easy to crash, wedge or otherwise damage the kernel, requiring a reboot.

I wanted a way to work on OpenZFS just like I’d work on any other program - make a change, compile it, run it and see what happens. I happened to know that microVMs were getting to the point where booting in a fraction of a second was possible, so I decided to see if I could use one for this. **quiz** is what I came up with.

## Note

This is early. I’ve got some ideas, but I’m not quite sure exactly where I’m headed yet. This is all safe to try though; its not going to damage anything.

## Setup

### Install prerequisites

On a Debian 11 system you’ll need at least:

* `qemu-system-x86` from backports (new enough to support the `microvm` profile)
* `mmdebstrap`
* kernel build deps

### Build the kernel

Run `quiz-prepare-kernel`. This will grab kernel source. compile it with a very minimal config to support only what’s needed for the microvm, and install it into the system dir.

### Build the root image

Run `quiz-prepare-root`. This will generate a barely-there Debian root image.

### Build the work dir

Run `quiz-prepare-work`. This sets up a chunk of host filesystem that will be mapped into the VM, and is where the OpenZFS modules and related development programs can be stored.

### Build OpenZFS

You’ll need to compile OpenZFS targetting the work dir:

```
./autogen.sh
./configure --with-linux=/path/to/quiz/build/kernel/linux-5.10.170 --with-linux-obj=/path/to/quiz/system/work/lib/modules/5.10.170/build --prefix=/zfs --disable-sysvinit --disable-systemd --disable-pam
make -j5
make install DESTDIR=/path/to/quiz/system/work
```

This is your compile step for now. It’ll get better!

### Run it

Simple boot to a shell:

```
$ ./quiz
[quiz] 20230326-12:10:54 running qemu
...
[    0.500224] Run /sbin/init as init process
[    0.606734] quiz: starting user program
[INFO  tini (1)] Spawned child process '/work/.quiz.run' with pid '439'
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
root@quiz:/# 
```

or run a program directly:

```
$ ./quiz uname -a
[quiz] 20230326-12:31:30 creating run script
[quiz] 20230326-12:31:30 running qemu
...
[    0.519514] Run /sbin/init as init process
[    0.629666] quiz: starting user program
[INFO  tini (1)] Spawned child process '/work/.quiz/run' with pid '439'
Linux quiz 5.10.170 #1 SMP Sat Mar 25 22:04:29 AEDT 2023 x86_64 GNU/Linux
[INFO  tini (1)] Main child exited normally (with status '0')
```

Include some profiles:

```
$ ./quiz -p zfs zpool status
[quiz] 20230326-12:31:46 including profile: zfs
[quiz] 20230326-12:31:46 creating run script
[quiz] 20230326-12:31:46 running qemu
...
[    0.775779] quiz: starting profile init
[    0.806856] spl: loading out-of-tree module taints kernel.
[    0.917986] zfs: module license 'CDDL' taints kernel.
[    0.918034] Disabling lock debugging due to kernel taint
[    1.972254] ZFS: Loaded module v2.1.99-1, ZFS pool version 5000, ZFS filesystem version 5
[    1.973699] quiz: starting user program
[INFO  tini (1)] Spawned child process '/work/.quiz/run' with pid '453'
no pools available
[INFO  tini (1)] Main child exited normally (with status '0')
```

## Todo

* a config file, be way less hardcodey
* more profiles
  * temp blockdevs
  * persistent blockdevs
  * mdadm stuff (fake error environments)
  * pool create/import
  * oh maybe even init could be one
  * oh crap package setup could be one too hmmm
* zfs build support (`quiz-build-zfs` or something)
