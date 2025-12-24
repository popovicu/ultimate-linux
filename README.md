# Ultimate Linux!!!

This is a fun tiny project for building a tiny Linux distribution in just JavaScript (and a tiny bit of C to enable mounting to get some fun results).

```
--- ULTIMATE LINUX SHELL ---
Commands: ls, cd, cat, mkdir, mount, exit
```

## Background context

I post a lot on X (Twitter) and if you don't follow me already, [please follow now](https://x.com/popovicu94)!

Lately I've been posting a lot about Unix, Linux, ideas of kernel syscall stability, etc.

In particular, I explored lately how Linux is more or less unique in the kernel/OS world for multiple reasons. First, it's a rare kernel that is shipped independently from the rest of the OS. BSDs, for example, ship the kernel in a coherent unit with the foundational userspace. Linux thus has a unique problem of defining its contract with software built on top of it. And Linux chose stable syscall ABI as this contract. This is in contrast with something like macOS, which is a Unix-certified OS, but which exposes only its system library as the public contract. Apple doesn't guarantee binary backwards compatibility.

Then I explored how pure Go binaries can interestingly target the kernel itself directly via syscalls for its static binaries, and not depend on the system libraries, at least on Linux. There were some explorations around `u-root` project, etc.

Every once in a while I get comments about how wrong I am when talking about C, Go, Rust, you name it. Comments like Go sucks because it does what it does, I'm wrong when I say "Linux is a kernel, not a complete OS", I don't understand Unix, POSIX, whatever you can think of.

So this time I'm doing something to get all their love. I'm creating a libc-less micro Linux distribution in... JavaScript! A standalone JavaScript binary no less! Of course, there's a transpilation step through C, but who cares -- this is the Ultimate Linux! 💪🐧

Anyway, putting the jokes aside, if you want to really understand what is going on here and you want to understand the fundamentals of how the Linux kernel interfaces with user software, please check out [this article](https://popovicu.com/posts/making-a-micro-linux-distro/) that I have previously written. It's about making these "micro Linux distros" and it should give you fundamental understanding of what Linux distros really are.

## Build instructions

Download `quickjs` source code:

```
wget https://bellard.org/quickjs/quickjs-2025-09-13-2.tar.xz
```

Unpack it:

```
tar -xf quickjs-2025-09-13-2.tar.xz
```

Go inside the source directory, run `make` and go back up.

Now go ahead and install `musl` libc on your system: https://musl.libc.org/

**Do not worry**, `musl` installation is _polite_ by default, meaning it will install itself into `/usr/lib/local`, it will not clash with your host's libc. The reason why we install `musl` is because it provides `gcc` and `clang` wrapper scripts for linking against `musl` instead of your system library. You can then use

```
/usr/local/musl/bin/musl-gcc
```

instead of your system's GCC to link against the freshly built `musl` instead of your host system. That's what we do here and we link statically against `musl` to make a standalone ELF file which doesn't depend on the running system's libc.

We're now ready to transpile the JavaScript code to C, link it together with some system operations and produce the final ULTIMATE SHELL!

```
./quickjs-2025-09-13/qjsc -M sys_ops,js_init_module_sys_ops -e -o ultimate_shell.c ultimate_shell.js && /usr/local/musl/bin/musl-gcc -static -o ultimate_shell ultimate_shell.c sys_ops.c -I ./quickjs-2025-09-13 ./quickjs-2025-09-13/libquickjs.a -lm -ldl -lpthread
```

You can run `./ultimate_shell` on your build machine as well, it should be fully portable.

However, let's run it on a VM! First, let's build `initramfs`.

```
echo "ultimate_shell" | cpio -o -H newc > image.cpio
```

Now let's run the VM with the Ultimate Shell as the PID 1!

```
qemu-system-x86_64 -m 4G -kernel /tmp/linux/linux-6.17.12/arch/x86/boot/bzImage -initrd ./image.cpio -nographic --enable-kvm -smp 8 -append "console=ttyS0 rdinit=/ultimate_shell"
```

After a long blob of text from QEMU, you should get the shell prompt and you can play around a bit:

```
...
[    0.805878] x86/mm: Checked W+X mappings: passed, no W+X pages found.
[    0.807049] x86/mm: Checking user space page tables
[    0.839182] x86/mm: Checked W+X mappings: passed, no W+X pages found.
[    0.840185] Run /ultimate_shell as init process
--- ULTIMATE LINUX SHELL ---
Commands: ls, cd, cat, mkdir, mount, exit
[/] # ls
.  ..  ultimate_shell  root  dev
[/] # ls /dev
.  ..  console
[/] # mkdir proc
[/] # ls
.  ..  proc  ultimate_shell  root  dev
[/] # mount proc /proc proc
Mount proc -> /proc: Success
[/] # cat /proc/cmdline
console=ttyS0 rdinit=/ultimate_shell
[/] # cat /proc/1/environ
HOME=/
[/] # cat /proc/1/cmdline
/ultimate_shell
[/] #
```
