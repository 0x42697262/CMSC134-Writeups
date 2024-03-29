# Machine problem 1: Buffer overflow to exit

Consider the following code for a nonterminating program:

```c
// vuln.c
#include <stdio.h>

void vuln() {
    char buffer[8];
    gets(buffer);
}

int main() {
    vuln();
    while (1) {
    }
}
```

This program will accept an unbounded number of non-null byte
characters from standard input until you hit \<Enter> and then proceed
to perform an infinite loop.

The goal is to construct "shellcode" to cause the program to terminate
with a desired exit code of `1` using a stack smash attack.

Note that crashes due to malformed "shellcode" will not result in an
exit code of `1` and therefore will not count.

## Compiling

Compile the `vuln.c` source code with the following incantation.

```sh
$ gcc -m32 -fno-stack-protector -mpreferred-stack-boundary=2
    -fno-pie -ggdb -z execstack vuln.c -o vuln
```

Otherwise, your exploit will not work due to memory safety mitigations.

## Finding addresses

Using `gdb` we can find addresses that we're interested in.

Attach `gdb` to our `vuln` program.

```sh
$ gdb vuln
```

Try to set a breakpoint at line 1.

```
(gdb) break 1
Breakpoint 1 at 0x1193: file vuln.c, line 5.
(gdb) run
Breakpoint 1, vuln () at vuln.c:5
5               gets(buffer);
```

Print addresses.

```
(gdb) print &buffer
$1 = (char (*)[8]) 0xffffd6c8
```

Show registers.

```
(gdb) info registers
eax            0x565561a2          1448436130
ecx            0x90761f74          -1871306892
edx            0xffffd700          -10496
ebx            0xf7fa9e34          -134570444
...
```

Show stack frame info.

```
(gdb) info frame
Stack level 0, frame at 0xffffd6d8:
eip = 0x56556193 in vuln (vuln.c:5); saved eip = 0x565561aa
called by frame at 0xffffd6e0
source language c.
...
```

## Obtaining machine code

Various ways to obtain machine code.

One way is to write assembly in C and disassemble it.

```c
// asm.c

int main() {
    __asm__("xor %eax, %eax;"
            "inc %eax;"
            "mov %eab, %eax;"
            "leave;"
            "ret;"
    );
}
```

Compile using the following incantation:

```sh
$ gcc -m32 -fno-stack-protector -fno-pie -masm=intel asm.c -o asm
```

And obtain machine code with the `objdump` tool:

```sh
$ objdump -d asm > asmdump
```

Machine code will be found in the file called `asmdump`.

```asm
...
0000117d <main>:
    117d:    55                       push   %ebp
    117e:    89 e5                    mov    %esp,%ebp
    1180:    e8 13 00 00 00           call   1198 <__x86.get_pc_thunk.ax>
    1185:    05 6f 2e 00 00           add    $0x2e6f,%eax
    118a:    31 c0                    xor    %eax,%eax
    118c:    40                       inc    %eax
    118d:    89 c3                    mov    %eax,%ebx
    118f:    cd 80                    int    $0x80
    1191:    b8 00 00 00 00           mov    $0x0,%eax
    1196:    5d                       pop    %ebp
    1197:    c3                       ret
...
```

Here, we see that `xor %eax, %eax` is `0x31 0xc0` (look at line
`118a`) in machine code.

## Writing down the shellcode

Suppose we want to put a `1` in the `%eax` register, the assembly for that is:

```asm
xor %eax, %eax // set %eax to 0
inc %eax // increment %eax by 1
```

The machine code from the disassembly shows these two instructions
are: `0x31 0xc0` and `0x40`.

We can write down actual bytes to a file called `egg` using `echo`:

```sh
$ echo -ne "\x31\xc0\x40" > egg
```

## Running the program in `gdb`

In `gdb`, we can run `vuln` and send in "shellcode" from `egg` as the input:

```
(gdb) run < egg
```

When the program asks for input, the contents of `egg` will be passed
in automatically.

## Collaboration

I expect you all to collaborate for this exercise, but the
deliverables are still by group.

If you do not have access to a Linux machine, please collaborate and
share outputs.

## Consultation

Feel free to ask me about getting to a solution for this machine problem.

## Deliverables

Use the same grouping as your writeup group.

- A short writeup.
- Copy of your machine code, `egg`.

Submit it on the UVEC.
