# RISC-V LLVM on HiFive1 RevB

Source: [oficial riscv-tools](https://github.com/riscv/riscv-tools) and [sifive riscv-llvm](https://github.com/sifive/riscv-llvm)

This repository adds support for various aspects of the RISC-V instruction set to the Clang C/C++ compiler band LLVM back end.

You will need the RISC-V GCC toolchain. If you don't have RISC-V hardware then you will want to have QEMU to run your programs.

The following bash commands have been tested on Debian 10

By default, clang is generating rv32imac/rv64imac code but you can override this, and in particular if you have floating point hardware you can add "-march=rv32gc" (or rv64gc) to the clang command line to get FPU instructions with a soft float ABI.

### Requirements

You need install programs before use the tools

```bash
sudo apt-get update
sudo apt-get install binutils build-essential libtool texinfo gzip zip unzip patchutils curl git make cmake ninja-build automake bison flex gperf grep sed gawk python bc zlib1g-dev libexpat1-dev libmpc-dev libglib2.0-dev libfdt-dev libpixman-1-dev 
```

### Prebuilt RISC‑V GCC Toolchain

Save time by using one of the prebuilt toolchains that contain all the tools necessary to compile and debug programs on SiFive products.

For Debian distributions family:

```bash
wget https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-8.2.0-2019.05.3-x86_64-linux-ubuntu14.tar.gz
wget https://static.dev.sifive.com/dev-tools/riscv-openocd-0.10.0-2019.05.1-x86_64-linux-ubuntu14.tar.gz
```

For macOS, Windows or CentOS visit: https://www.sifive.com/boards/

### LLVM SiFive version

Change FolderLocation to the folder location of the toolchain.

```bash
git clone https://github.com/llvm/llvm-project.git riscv-llvm
cd riscv-llvm
ln -s ../../clang llvm/tools || true
mkdir build && cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE="Release" \
  -DBUILD_SHARED_LIBS=True -DLLVM_USE_SPLIT_DWARF=True \
  -DCMAKE_INSTALL_PREFIX="Toolchain prebuilt Folder Location" \
  -DLLVM_OPTIMIZED_TABLEGEN=True -DLLVM_BUILD_TESTS=False \
  -DDEFAULT_SYSROOT="Toolchain prebuilt Folder Location/riscv64-unknown-elf" \
  -DLLVM_DEFAULT_TARGET_TRIPLE="riscv64-unknown-elf" \
  -DLLVM_TARGETS_TO_BUILD="RISCV" \
  ../llvm
cmake --build . --target install
```

Sanity test your new RISC-V LLVM

```bash
echo -e '#include <stdio.h>\n int main(void) { printf("Hello world!\\n"); return 0; }' > hello.c

# 32 bit
clang -O -c hello.c --target=riscv32
riscv64-unknown-elf-gcc hello.o -o hello -march=rv32imac -mabi=ilp32

# 64 bit
clang -O -c hello.c
riscv64-unknown-elf-gcc hello.o -o hello -march=rv64imac -mabi=lp64
```

If you want can test the versions of Clang and LLVM with `` clang --version`` and ``llc --version``  

![](/home/fgranados/Documents/git/LLVM-RISCV/images/llc.png) ![](/home/fgranados/Documents/git/LLVM-RISCV/images/clang.png)



### Using the Tools 

For use the tools based on freedom-e-sdk you need export  RISC‑V GCC Toolchain, OpenOCD and clang path :

```bash
export RISCV_PATH=/dir/riscv64-unknown-elf-gcc
export RISCV_OPENOCD_PATH=/dir/openocd
export CLANG_PATH=/dir/clang 
```

#### Building an Example

By default the tool uses a "hello-world!" program, all programs are in the software folder. If you want to add another program only need to be created a new folder into the software directory with Makefile similar to the other programs and the code in C.

First is generated an LLVM IR file:

```bash
make clang
```

You should apply the necessary changes in the .ll file, next need generated an object file to compile into a HiFive1 board.

```bash
make llc
```

If everything is right:

```bash
make software
```

And finally upload to board:

```bash
make upload
```

#### Building a program

To build a different program to default hello, you need to add the instruction PROGRAM with the name of the program, for example:

```bash
make [PROGRAM=hello] clang
```

As the before steps:

```bash
make [PROGRAM=hello] llc
make [PROGRAM=hello] software
make [PROGRAM=hello] upload
```



## LLVM IR 

Using the toolchain riscv64-unknown-elf-gcc (SiFive GCC 8.2.0-2019.05.3) 

On example a simple loop:

```c
#include <stdio.h>
int main(){
    int sum = 0;
    for(int i = 0; i < 100; i++){
        sum += i;
        }
    printf("sum: %d\n", sum);
    return 1;
}
```



| LLVM IR generated                                            |
| :----------------------------------------------------------- |
| ; Function Attrs: nounwind <br/>define i32 @main() #0 {<br/>  %1 = alloca i32, align 4<br/>  %2 = alloca i32, align 4<br/>  %3 = alloca i32, align 4<br/>  store i32 0, i32* %1, align 4<br/>  store i32 0, i32* %2, align 4<br/>  store i32 0, i32* %3, align 4<br/>  br label %4 |
| ; <label>:4:           ; preds = %11, %0<br/> %5 = load i32, i32* %3, align 4<br/> %6 = icmp slt i32 %5, 100<br/> // In this case, variable% 6 corresponds to the stop value, it is the condition that is observed in the code: i <100 <br/> // If some techniques are applied such as Loop Perforation this value is reduced by an error defined percentage.<br/>br i1 %6, label %7, label %14 |
| ; <label>:7:              ; preds = %4<br/>  %8 = load i32, i32* %3, align 4<br/>  %9 = load i32, i32* %2, align 4<br/>  %10 = add nsw i32 %9, %8<br/>  store i32 %10, i32* %2, align 4<br/>  br label %11 |
| ; <label>:11:              ; preds = %7<br/>  %12 = load i32, i32* %3, align 4<br/>  %13 = add nsw i32 %12, 1<br/>  store i32 %13, i32* %3, align 4<br/>  br label %4 |
| ; <label>:14:                 ; preds = %4<br/> %15 = load i32, i32* %2, align 4<br/> %16 = call signext i32 (i8*, ...) i32 signext %15<br/> ret i32 1<br/>} |

