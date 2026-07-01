# Why Do We Even Need a Cross Compiler?

If you already have GCC installed on your Linux box, it feels natural to ask: why not just use that? It already compiles C, it already spits out ELF files, so what's missing?

The answer comes down to one thing: your regular GCC was built assuming it's compiling programs *for* Linux, not writing the *next layer below* Linux.

## The core problem

The GCC sitting on your machine right now has a bunch of assumptions baked into it that you never think about because they're always true, until you start writing an OS:

- It assumes glibc exists and links against it automatically.
- It assumes there's already a kernel underneath handling memory, processes, and file access.
- It generates startup code (`crt0.o`, `crti.o`, etc.) that expects to call into `libc_start_main`, which in turn expects a running OS to hand control to.
- Its default header and library search paths point straight at your host system's files.

None of that exists yet in your kernel. Your kernel isn't a program running on an OS, it's supposed to become the OS. There's no libc, no syscalls in the Linux sense, no process model. Asking your host GCC to compile kernel code is like asking a contractor to install kitchen cabinets in a house that doesn't have walls yet. The tool keeps trying to attach to structure that isn't there.

## "But ELF is ELF, right?"

This is the part that actually confused most people the first time, myself included. ELF is just a file format, basically a container. Two ELF files can look structurally similar and still be built on completely different assumptions.

Your host GCC producing ELF doesn't mean it's producing *freestanding* ELF. Under the hood it can still:

- Quietly pull in Linux's startup routines.
- Emit calls to libc functions you never asked for (stack protector checks, some compiler builtins that expand into libc calls).
- Let you accidentally `#include` your host's real `stdio.h` or link against your host's real libc, giving you a binary that references OS features your kernel hasn't built yet.

So the container format matching doesn't save you. The compiler's internal assumptions about "what environment will this code run in" are the actual problem.

## What i686-elf actually means

This is a GCC target triple, and it decodes as: x86, 32-bit, ELF output, no target OS. That last part is the key. Building a cross compiler with this target gives you a GCC that:

- Has zero built-in dependency on libc or any OS.
- Won't silently reach into your host system's headers or libraries.
- Defaults to freestanding-safe code generation, which is exactly what a kernel needs (you still pass `-ffreestanding` explicitly to be sure, but the target already leans that direction).

It's not a different, weaker version of GCC. It's the same compiler, just stripped of the assumption that Linux (or any OS) is sitting underneath the code it produces.

## The bootstrapping intuition

Think of it like building a house with no electricity or plumbing yet. You can't use tools that assume power outlets and running water already exist, because they don't. You need tools that make zero assumptions about the environment, since right now there's nothing but bare metal.

The cross compiler is that "assume nothing" toolchain. It's not extra ceremony for the sake of tradition, it's the removal of a dependency (your host OS's runtime assumptions) that would otherwise sneak into every object file you compile without you noticing until something breaks in a confusing way.

Later, once your OS matures to the point where it can host its own compiler, you'd build a new GCC that targets *your* OS specifically instead of "no OS." That's the self-hosting step, and it's a much later milestone, not something to worry about on day one.

## cc from source 

first install all the dependencies needed   
`sudo dnf install gcc make bison flex gmp-devel mpfr-devel libmpc-devel texinfo nasm qemu libgcc.i686`

### now building the cross compiler (i686-elf)   

cc runs on your x86_64 machine but creates code for a diff target (i686 architecture) which is a standard practice for os dev   
refering to bare bones tutorial on https://wiki.osdev.org/Bare_Bones   

so now, setup directories and download sources:   
(will provide the command later)   
<img width="1920" height="860" alt="image" src="https://github.com/user-attachments/assets/f265b027-b692-42de-97d4-6efa14996057" />    

okay i will give next steps afterwards .. its been around 4-5 hours and i am still not able to make a cross compiler .. i will update this once i done with it 


## building the cross compiler (i686-elf)

before starting, i highly recommend reading the official osdev wiki page once. i am following the same guide here, but there is one important difference that i'll explain later.

https://wiki.osdev.org/GCC_Cross-Compiler

### 1. install the required dependencies

on fedora:

```bash
sudo dnf install gcc gcc-c++ make bison flex gmp-devel mpfr-devel libmpc-devel texinfo nasm qemu libgcc.i686
```

### 2. choose a location for the toolchain

i kept everything inside `~/src` and installed the final compiler into `~/opt/cross`.

```bash
mkdir -p ~/src
mkdir -p ~/opt/cross
```

set the required environment variables.

```bash
export PREFIX="$HOME/opt/cross"
export TARGET=i686-elf
export PATH="$PREFIX/bin:$PATH"
```

---

## building binutils

first download binutils.

```bash
cd ~/src

wget https://ftp.gnu.org/gnu/binutils/binutils-2.39.tar.gz

tar -xzf binutils-2.39.tar.gz
```

create a separate build directory.

```bash
mkdir build-binutils
cd build-binutils
```

configure it.

```bash
../binutils-2.39/configure \
    --target=$TARGET \
    --prefix="$PREFIX" \
    --with-sysroot \
    --disable-nls \
    --disable-werror
```

compile and install.

```bash
make -j$(nproc)

make install
```

verify everything.

```bash
ls ~/opt/cross/bin
```

you should see tools like

```text
i686-elf-as
i686-elf-ld
i686-elf-ar
i686-elf-ranlib
...
```

if those exist, binutils is done.

---

# do not use gcc 12, 13 or 14 if you're on fedora 44

this is the part where i wasted around 4 to 5 hours.

almost every tutorial i found used gcc 12 or 13. i followed them exactly, and every single version failed while compiling `libcody`.

the error looked something like

```text
error: no matching function for call to 'S2C(const char8_t...)'
```

this is because fedora 44 ships with gcc 16 as the host compiler, while gcc 12, 13 and 14 were never really tested against a host this new.

i tried:

- gcc 14
- gcc 13
- gcc 12
- patching source files
- replacing `u8""` strings
- editing libcody
- rebuilding everything multiple times

none of it worked.

if you're on fedora 44, just save yourself the headache and build gcc 16 instead.

---

# building gcc 16

download it.

```bash
cd ~/src

wget https://ftp.gnu.org/gnu/gcc/gcc-16.1.0/gcc-16.1.0.tar.xz

tar -xf gcc-16.1.0.tar.xz
```

download gcc's prerequisites.

```bash
cd gcc-16.1.0

./contrib/download_prerequisites

cd ..
```

this downloads compatible versions of gmp, mpfr, mpc and isl directly into the source tree.

---

create a fresh build directory.

```bash
mkdir build-gcc

cd build-gcc
```

configure gcc.

```bash
../gcc-16.1.0/configure \
    --target=$TARGET \
    --prefix="$PREFIX" \
    --disable-nls \
    --enable-languages=c \
    --without-headers
```

don't worry if configure prints warnings about unsupported target libraries.

for a bare metal target like `i686-elf`, that's completely normal.

---

build the compiler.

```bash
make -j$(nproc) all-gcc
```

this step took around **40 to 45 minutes** on my ryzen 3 3200u.

don't panic if it keeps printing compiler output for a long time.

---

once that's done:

```bash
make -j$(nproc) all-target-libgcc
```

then install everything.

```bash
make install-gcc

make install-target-libgcc
```

---

verify the installation.

```bash
~/opt/cross/bin/i686-elf-gcc --version
```

you should see something similar to

```text
i686-elf-gcc (gcc) 16.1.0
```

congratulations 🎉

you now have a working cross compiler.

---

## add it to your path permanently

if you're using zsh:

```bash
echo 'export PATH="$HOME/opt/cross/bin:$PATH"' >> ~/.zshrc

source ~/.zshrc
```

for bash:

```bash
echo 'export PATH="$HOME/opt/cross/bin:$PATH"' >> ~/.bashrc

source ~/.bashrc
```

after that you can simply run

```bash
i686-elf-gcc --version
```

from anywhere.

---

## some mistakes i made so you don't have to

- don't build gcc inside its source directory. always create a separate `build-gcc` folder.
- don't skip `./contrib/download_prerequisites`.
- don't edit gcc source files unless you actually know the root cause.
- if you're on fedora 44, don't waste time trying gcc 12, 13 or 14. build gcc 16 instead.
- don't forget `make install-gcc` and `make install-target-libgcc`. i finished building everything and then wondered why `i686-elf-gcc` didn't exist 😭

---

## finally

this honestly ended up being the hardest part of the bare bones tutorial for me.

writing the kernel was supposed to be the scary part, but getting a compiler that could actually compile the kernel turned out to be the real boss fight.

hopefully this saves someone else from spending half a day wondering why every tutorial on the internet keeps throwing `char8_t` errors on a modern fedora installation.


