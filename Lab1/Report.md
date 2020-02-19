# Report of Lab1 - Booting and GCC tracing

## Part 1: PC Bootstrap

### The PC's Physical Address Space

A PC's physical address space is hard-wired to have the following general layout:

``` none
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

When Intel finally "broke the one megabyte barrier" with the 80286 and 80386 processors, which supported 16MB and 4GB physical address spaces respectively, the PC architects nevertheless preserved the original layout for the low 1MB of physical address space in order to ensure backward compatibility with existing software. Modern PCs therefore have a "hole" in physical memory from 0x000A0000 to 0x00100000, dividing RAM into "low" or "conventional memory" (the first 640KB) and "extended memory" (everything else). In addition, some space at the very top of the PC's 32-bit physical address space, above all physical RAM, is now commonly reserved by the BIOS for use by 32-bit PCI devices.

### The ROM BIOS

The following line:

```gdb
[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b
```

is GDB's disassembly of the first instruction to be executed. From this output you can conclude a few things:

- The IBM PC starts executing at physical address 0x000ffff0, which is at the very top of the 64KB area reserved for the ROM BIOS.
- The PC starts executing with `CS = 0xf000` and `IP = 0xfff0`.
- The first instruction to be executed is a `jmp` instruction, which jumps to the segmented address `CS = 0xf000` and `IP = 0xe05b`.

## Part 2: The Boot Loader

#### 1. At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

When `CR0` register turns on. Before which, *gdt* is loaded, and the `cs`, `ds`, `ss` registers are set to specific values.

#### 2. What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?

According to the disassembly file `obj/boot/boot.asm`, the last instruction of the boot loader:

``` c
((void (*)(void)) (ELFHDR->e_entry))();
```

Set breakpoint at the instruction and print the memory at `0x10018`:

```assembly
Breakpoint 1, 0x00007d6b in ?? ()
(gdb) x/8x 0x10000
0x10000:	0x464c457f	0x00010101	0x00000000	0x00000000
0x10010:	0x00030002	0x00000001	0x0010000c	0x00000034
```

`0x0010000c` is the physical address of the entry of the kernel.

According to the disassembly file `obj/kern/kernel.asm`, the first instruction of the kernel:

``` assembly
f010000c <entry>:
movw $0x1234, 0x472
```

Since the last instruction in `boot/main.c` is

#### 3. *Where* is the first instruction of the kernel?

Loaded at the physical address of `0x0100000`(1M)

#### 4. How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

From the ELF *program header*, more details are as follows.

### Load the Kernel

The boot loader uses the ELF *program headers* to decide how to load the sections. The program headers specify which parts of the ELF object to load into memory and the destination address each should occupy.

```shell
objdump -h # print the ELF header
objdump -x # print the program header
```

After executing the command `objdump -x obj/kern/kernel`, the console's output:

``` shell
obj/kern/kernel:     file format elf32-i386
obj/kern/kernel
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c  ### entry of the program

Program Header:   
### The areas marked LOAD will be loaded into memory at the virtual address(vaddr) or physical address(paddr) with the size of memory(memsz) and file(filesz).
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x00007120 memsz 0x00007120 flags r-x
    LOAD off    0x00009000 vaddr 0xf0108000 paddr 0x00108000 align 2**12
         filesz 0x0000a948 memsz 0x0000a948 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx

Sections:
### These are the sections to be loaded by the ELF loader in boot/main.c
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001871  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       00000714  f0101880  00101880  00002880  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         000038d1  f0101f94  00101f94  00002f94  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000018bb  f0105865  00105865  00006865  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .bss          00000648  f0112300  00112300  00013300  2**5
                  CONTENTS, ALLOC, LOAD, DATA
  6 .comment      00000035  00000000  00000000  00013948  2**0
                  CONTENTS, READONLY

```

The minimal ELF loader in `boot/main.c` reads each section of the kernel from disk into memory at the section's load address and then jumps to the kernel's entry point.

``` c
// boot/main.c line 50
// load each program segment (ignores ph flags)
ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
eph = ph + ELFHDR->e_phnum;
for (; ph < eph; ph++)
	// p_pa is the load address of this segment (as well
	// as the physical address)
	readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

// call the entry point from the ELF header
// note: does not return!
((void (*)(void)) (ELFHDR->e_entry))();
```

## Part 3: The Kernel

### Memory Remapping

Up until `kern/entry.S` sets the `CR0_PG` flag, memory references are treated as physical addresses. Once `CR0_PG` is set, memory references are virtual addresses that get translated by the virtual memory hardware to physical addresses. `entry_pgdir` translates virtual addresses in the range 0xf0000000 through 0xf0400000 to physical addresses 0x00000000 through 0x00400000, as well as virtual addresses 0x00000000 through 0x00400000 to physical addresses 0x00000000 through 0x00400000.

**Exercise 7** Now trace into the JOS kernel and stop at the `movl %eax, %cr0`. 

``` shell
(gdb) b *0x0100025
Breakpoint 2 at 0x100025
(gdb) c
Continuing.
=> 0x100025:	mov    %eax,%cr0

Breakpoint 2, 0x00100025 in ?? ()
(gdb) x/4x 0x100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
(gdb) x/4x 0xf0100000
0xf0100000 <_start+4026531828>:	0x00000000	0x00000000	0x00000000	0x00000000
(gdb) si
=> 0x100028:	mov    $0xf010002f,%eax
0x00100028 in ?? ()
(gdb) x/4x 0x100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
(gdb) x/4x 0xf0100000
0xf0100000 <_start+4026531828>:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766

```

The output indicates that the the virtual address `0xf0100000` is remapped to the physical address `0x100000`.

**Notice: ** In `kern/kernel.c`, there is a comment:

``` assembly
###################################################################
# The kernel (this code) is linked at address ~(KERNBASE + 1 Meg), 
# but the bootloader loads it at address ~1 Meg.
#	
# RELOC(x) maps a symbol x from its link address to its actual
# location in physical memory (its load address).	 
###################################################################

```

The link and load addresses are at the top of `kern/kernel.ld`:

``` assembly
/* Link the kernel at this address: "." means the current address */
. = 0xF0100000;
```

Before `CR0_PG` flag is on, we need to remap the entry of the kernel by hand, as it is said in the comment above.

### Formatted Printing to the Console

Read `kern/printf.c`, `lib/printfmt.c`, `kern/console.c` and make sense of their **relationship**.

**Exercise 8**

> We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment.

This code fragment lies in `kern/printfmt.c line 207`. According to `case 'x'`, it's not hard to fill in the code block.

``` c
// kern/printfmt.c line 207
case 'o':
	putch('0', putdat);
	num = getuint(&ap, lflag);
	base = 8;
	goto number;
```

##### 2. Explain the following from console.c

```c
if (crt_pos >= CRT_SIZE) {
    int i;
    memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
    for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
        crt_buf[i] = 0x0700 | ' ';
        crt_pos -= CRT_COLS;
}
```

According to the context of the code, `crt_pos` is the curse position from the left top of the screen, `crt_buf` saves the character that should be put on the screen at `crt_pos`. `CRT_SIZE` is the size of the screen, equals to `CRT_COLS(80)*CRT_ROWS(25)`   

Thus, the code here means to flush the first line on the screen and display the line(2) ~ line(CRT_COLS+1) on the screen.  

One thing to notice is the `0x0700|' '` here, we know the characters in ASCII format should be 8 bits. However, the `crt_buf` is 16 bits. The reason is that the higher 8 bits stand for the property of the character, the lower 8 bits stands for the ASCII. `0x0700|' '` means that print out the blank with default property.   

##### 3. Trace the execution of the following code step-by-step:  

```
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```

- In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?  
- List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.  

JOS reserves a code segment for us to debug the code, in `kern\monitor.c`. Line 57.

``` c
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	// Your code here.
	int x = 1, y = 3, z = 4;
    cprintf("x %d, y %x, z %d\n", x, y, z);
	return 0;
}
```

Compile and run, set the breakpoint.

**Important: Calling stack**

Before going on, it's necessary to learn about the GCC x86 calling convention. The calling stack of a callee and its caller looks like this:

``` shell
		       +------------+   |
		       | arg 2      |   \
		       +------------+    >- previous function's stack frame
		       | arg 1      |   /
		       +------------+   |
		       | ret %eip   |   /  ---> eip will be used by ret when the current func ret
		       +============+      ---> Here is no skip on the stack
		       | saved %ebp |   \
		%ebp-> +------------+   |
		       |            |   |
		       |   local    |   \
		       | variables, |    >- current function's stack frame
		       |    etc.    |   /
		       |            |   |
		       |            |   |
		%esp-> +------------+   /  ---> esp is always pointing to the top of the stack
		
```

| Example instruction | What it does                                  |
| ------------------- | --------------------------------------------- |
| pushl %eax          | subl $4, %esp <br />movl %eax, (%esp)         |
| popl %eax           | movl (%esp), %eax  <br />addl $4, %esp        |
| call 0x12345        | pushl %eip(\*)  <br />movl $0x12345, %eip(\*) |
| ret                 | popl %eip (\*)                                |

`eip` ensures the calling stack to return at the right position.

Now we know 

##### 4. Run the following code.

```
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
```

What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise.

The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616`to a different value?

[Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](http://www.networksorcery.com/enp/ien/ien137.txt).



##### 5. In the following code, what is going to be printed after'y=?' (note: the answer is not a specific value.) Why does this happen?

```
    cprintf("x=%d y=%d", 3);
```



##### 6. Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?