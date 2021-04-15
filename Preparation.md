# Preparation

Before starting doing any lab, we should first set up `xv6` on our own computers. [Here](https://pdos.csail.mit.edu/6.S081/2020/tools.html) is the instruction of installation. Yet I stuck on some of the steps. Fortunately the problems have already solved and my solutions are written below.\
\
P.S. I was using Ubuntu at that time so I followed the "Installing via APT (Debian/Ubuntu)" part of instructions.

## 1) riscv64-linux-gnu-gcc: error: unrecognized command line option ‘-mno-relax’; did you mean ‘-Wno-vla’?

After installing dependencies, I ran `make qemu` and the error message above came out halfway. This problem has been [reported on Github already](https://github.com/mit-pdos/xv6-riscv/issues/7) and it can be solved by switching to version 8 of `riscv64-linux-gnu-gcc`:

```bash
sudo apt install gcc-8-riscv64-linux-gnu
sudo apt remove gcc-riscv64-linux-gnu
```

Then replace `CC = $(TOOLPREFIX)gcc` with `CC = $(TOOLPREFIX)gcc-8` in the `Makefile`.

## 2) make: qemu-system-riscv64: Command not found

I posted this problem [on Stackoverflow](https://stackoverflow.com/questions/66718225/qemu-system-riscv64-is-not-found-in-package-qemu-system-misc/66721466#66721466) too. For a quick reference: **UPGRADE UBUNTU**.\
\
Again on the way of `make qemu`, another error came up:

```bash
# lots of outputs...
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
make: qemu-system-riscv64: Command not found
```

and I found that there is no `qemu-system-riscv64` under `/usr/bin` after installing `qemu-system-misc`(version 1:2.11+dfsg-1ubuntu7.36):

```bash
$ ls /usr/bin | grep qemu
qemu-img
qemu-io
qemu-nbd
qemu-system-alpha
qemu-system-cris
qemu-system-lm32
qemu-system-m68k
qemu-system-microblaze
qemu-system-microblazeel
qemu-system-moxie
qemu-system-nios2
qemu-system-or1k
qemu-system-sh4
qemu-system-sh4eb
qemu-system-tricore
qemu-system-unicore32
qemu-system-xtensa
qemu-system-xtensaeb
```

I had tried to install an older version of `qemu-system-misc` which is mentioned in the instruction:

> At this moment in time, it seems that the package qemu-system-misc has received an update that breaks its compatibility with our kernel. If you run make qemu and the script appears to hang after
>```bash
>qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk device,drive=x0,bus=virtio-mmio-bus.0
>```
>you'll need to uninstall that package and install an older version:
>```bash
>$ sudo apt-get remove qemu-system-misc
>$ sudo apt-get install qemu-system-misc=1:4.2-3ubuntu6
>```

yet `bash` said that version 1:4.2-3ubuntu6 was not found.\
\
After a long long time, I suddenly brought to mind that when I checked my debian version, the output was `buster/sid`, which is too old for using QEMU(and actually it has been mentioned in the instruction...). Therefore, everything went right after I upgraded Ubuntu to version 20.04.2 lol (I had used Ubuntu 18.04.5 originally).
