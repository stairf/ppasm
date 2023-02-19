PPASM -- The Preprocessor-Based Assembler
=========================================

**PPASM** is a multi-pass assembler for the 64-bit x86 architecture. It is
entirely written in the C/C++ preprocessor.

 - [Project State](#project-state)
 - [Installing PPASM](#installing-ppasm)
 - [Building PPASM](#building-ppasm)
 - [Examples](#example)
 - [Contributing](#contributing)


Project State
-------------

This is a proof of concept to demonstrate how powerful the preprocessor can be.
There may be missing or surprising functionality (aka bugs). Do not use PPASM
in serious projects.


Installing PPASM
----------------

The entire assembler is in a single header file, `ppasm.h`.


Building PPASM
--------------

There is no need to build PPASM. The header file is ready to use!


Examples
--------

```c
// file test1.c
#include "ppasm.h"
asmfunc(main,
	xor(eax, eax),
	ret()
)
```

The preprocessor translates the assembler instructions to bytes (formatted and annotated below):

```
$ cpp test1.c
__attribute__((section(".text.ppasm")))
const unsigned char main[] = {
	0x31, 0xc0, /* xor eax, eax */
	0xc3        /* ret          */
}
```

The remaining job of the C compiler is to copy this byte sequence into an ELF file:

```sh
$ cc -o test1 test1.c && ./test1
$ echo $?
0
```

Here is a hello world example:

```c
// file test2.c
#include "ppasm.h"
asmfunc(main,
	mov(eax, imm08zx32(1)),     // rax := 0x1 (__NR_write), operand has type imm32
	mov(edi, imm08zx32(1)),     // rdi := 0x1 (STDOUT_FILENO), operand has type imm32
	lea(rsi, label(0)),         // rsi points to label#0
	mov(edx, labeldiff32(1,0)), // rdx := label#1 - label#0
	syscall(80),                // syscall write(1, L0, L1-L0)
	xor(eax, eax),              // rax := 0
	ret(),                      // return
	label(0),
	byte(48),
	byte(65),
	byte(6c),
	byte(6c),
	byte(6f),
	byte(2c),
	byte(20),
	byte(42),
	byte(61),
	byte(72),
	byte(74),
	byte(21),
	byte(0a),
	label(1)
)
```

Again, this example can be compiled and executed:

```sh
$ cc -o test2 test2.c && ./test2
Hello world!
$ echo $?
0
```

Here is the same example, but without dependencies to the C runtime library. It
calls `exit()` (via system call) instead of returning from main. The `asm`
macro only generates the raw byte sequence, without the surrounding definition
of an `unsigned char[]`.

```c
// file test3.ppasm
#include "ppasm.h"
asm(
	mov(eax, imm08zx32(1)),     // rax := 0x1 (__NR_write), operand has type imm32
	mov(edi, imm08zx32(1)),     // rdi := 0x1 (STDOUT_FILENO), operand has type imm32
	lea(rsi, label(0)),         // rsi points to label#0
	mov(edx, labeldiff32(1,0)), // rdx := label#1 - label#0
	syscall(),                  // syscall write(1, L0, L1-L0)
	mov(eax, imm08zx32(3c)),    // rax := 0x3c (__NR_exit), operand has type imm32
	xor(edi, edi),              // rdi := 0x0
	syscall(),                  // syscall exit(0)
	label(0),
	byte(48),
	byte(65),
	byte(6c),
	byte(6c),
	byte(6f),
	byte(2c),
	byte(20),
	byte(42),
	byte(61),
	byte(72),
	byte(74),
	byte(21),
	byte(0a),
	label(1)
)
```

This version does not need a C compiler, only the preprocessor:

```sh
# preprocess and fix format for later
$ cpp test3.ppasm | grep -v '^#' | sed -e 's/,//g' >test3.raw

# convert hex to raw binary
$ xargs -n1 printf '\\\\%03o' <test3.raw | xargs -n1 printf >test3.bin

# convert raw binary to object file
$ objcopy -I binary test3.bin -O elf64-x86-64 test3.o --add-section .text=test3.bin --add-symbol _start=.text:0

# link object file to executable
$ ld -o test3 test3.o
```


Contributing
------------

If you want to contribute to PPASM, please follow the detailed instructions in
[Contributing.md](https://www.youtube.com/watch?v=dQw4w9WgXcQ).


