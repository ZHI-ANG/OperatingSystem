# Assignment of Lab 1 - Bootstrap

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

**Important: Calling Chain**

Before going on, it's necessary to learn about the GCC x86 calling convention. 

| Example instruction | What it does                                  |
| ------------------- | --------------------------------------------- |
| pushl %eax          | subl $4, %esp <br />movl %eax, (%esp)         |
| popl %eax           | movl (%esp), %eax  <br />addl $4, %esp        |
| call 0x12345        | pushl %eip(\*)  <br />movl $0x12345, %eip(\*) |
| ret                 | popl %eip (\*)                                |

The stack of a callee and its caller looks like this:

``` assembly
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

**`eip` or `cs:eip`, always points to the next instruction to run.** Thus it is important to save `eip` at the right time and position. When enter the callee, `eip` must points to the callee's first instruction.

The old `%ebp` is pushed to stack when a new function enters and popped when the function returns explicitly:  

```assembly
Current Function:
	pushl %ebp
	movl %esp, %ebp
	
	#################################
    ##  Here is the function body  ##
    #################################
    
    mov %ebp, %esp
    popl %ebp
    ret
```

Thus `ebp` records the calling chain.

The disassembly of our code is as follows:

``` assembly
# ------------------------ func mon_backtrace --------------------------
(gdb) b kern/monitor.c:61
Breakpoint 1 at 0xf0100774: file kern/monitor.c, line 61.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100774 <mon_backtrace+6>:	push   $0x4

Breakpoint 1, mon_backtrace (argc=0, argv=0x0, tf=0x0) at kern/monitor.c:62
62	    cprintf("x %d, y %x, z %d\n", x, y, z);
(gdb) si
=> 0xf0100776 <mon_backtrace+8>:	push   $0x3
0xf0100776	62	    cprintf("x %d, y %x, z %d\n", x, y, z);
(gdb) si
=> 0xf0100778 <mon_backtrace+10>:	push   $0x1
0xf0100778	62	    cprintf("x %d, y %x, z %d\n", x, y, z);
(gdb) si
=> 0xf010077a <mon_backtrace+12>:	push   $0xf0101b6e # The address of fmt 1
0xf010077a	62	    cprintf("x %d, y %x, z %d\n", x, y, z);
(gdb) info registers
esp            0xf010ff04	0xf010ff04  # -------------> stack top
ebp            0xf010ff18	0xf010ff18  # -------------> stack bottom of mon_backtrace
eip            0xf010077a	0xf010077a <mon_backtrace+12> # ret %eip

# ------------------------- func cprintf -----------------------------
(gdb) si
=> 0xf010077f <mon_backtrace+17>:	call   0xf010090b <cprintf>
0xf010077f	62	    cprintf("x %d, y %x, z %d\n", x, y, z);
(gdb) si
=> 0xf010090b <cprintf>:	push   %ebp # -------------> save the ebp of mon_backtrace
cprintf (fmt=0xf0101b6e "x %d, y %x, z %d\n") at kern/printf.c:27
27	{
(gdb) si
=> 0xf010090c <cprintf+1>:	mov    %esp,%ebp # ----------> new ebp of cprintf
0xf010090c	27	{
(gdb) si
=> 0xf010090e <cprintf+3>:	sub    $0x10,%esp # ---------> reserved for local variables
0xf010090e	27	{
(gdb) si
=> 0xf0100911 <cprintf+6>:	lea    0xc(%ebp),%eax # -----> the address of ap
31		va_start(ap, fmt);
(gdb) si
=> 0xf0100914 <cprintf+9>:	push   %eax # ---------------> address of ap in stack
32		cnt = vcprintf(fmt, ap);
(gdb) si
=> 0xf0100915 <cprintf+10>:	pushl  0x8(%ebp) # ----------> address of fmt in stack
0xf0100915	32		cnt = vcprintf(fmt, ap);

############################################################
## to verify fmt address is transmit to eax, dive into the
## stack and see if %ebp+0x8 == 0xf0101b6e.
## the following result confirms.
############################################################
(gdb) info registers
esp            0xf010fee4	0xf010fee4
ebp            0xf010fef8	0xf010fef8 # ---------------> the current ebp
eip            0xf0100915	0xf0100915 <cprintf+10>
(gdb) x/8x 0xf010fef8
0xf010fef8:	0xf010ff18	0xf0100784	0xf0101b6e	0x00000001
										^			^ 0xf010ff04 (ap = 1), the 2nd arg
									 	^ 0xf010fef8+0x8 (fmt = 0xf0101b6e)
0xf010ff08:	0x00000003	0x00000004	0xf01008d2	0xf010ff2c
				^ 3 (arg 3)  ^ 4 (arg 4)
```

`fmt` points to `0xf0101b6e`, which is the first argument.  `ap` points to `0xf010ff04`, which is the second argument.

According to the ASCII table, the address of `fmt` could be decode as the following strings (Notice: x86 is little-endian):

``` assembly
(gdb) x/6x 0xf0101b6e
0xf0101b6e:	0x64 25 20 78	0x20 79 20 2c	0x20 2c 78 25	0x64 25 20 7a
			  d  %  _  x 	  _  y  _  , 	  _  ,  x  % 	  d  %  _  z
0xf0101b7e:	0x3e 4b 00 0a	0x0d 09 00 20
					\0 \n
```

Now it is time to tackle the most confusing but interesting function `printfmt`. This function could accept arbitrary number of arguments and print to the console. What is the principle behind? 

``` c
// lib/printfmt.c line 67
static long long
getint(va_list *ap, int lflag)
{
	if (lflag >= 2)
		return va_arg(*ap, long long);
	else if (lflag)
		return va_arg(*ap, long);
	else
		return va_arg(*ap, int);
}

// inc/stdarg.h line 10
#define va_arg(ap, type) __builtin_va_arg(ap, type)
```

The macro`va_arg` first locates at the `ap` location in the memory and take `len(type)`bytes out of the memory. That is the argument we passed in when calling function `cprintf()`.

We could verify our assumption by debuging `lib/printfmt.c`:

``` assembly
(gdb) b lib/printfmt.c:75
Breakpoint 2 at 0xf0100fca: file lib/printfmt.c, line 75.
(gdb) c
Continuing.
=> 0xf0100fca <vprintfmt+705>:	mov    0x14(%ebp),%eax

Breakpoint 2, vprintfmt (putch=0xf01008d2 <putch>, putdat=0xf010fecc, 
    fmt=0xf0101b6e "x %d, y %x, z %d\n", ap=0xf010ff04 "\001")
    at lib/printfmt.c:75
75			return va_arg(*ap, int);
(gdb) c
Continuing.
=> 0xf0100fca <vprintfmt+705>:	mov    0x14(%ebp),%eax

Breakpoint 2, vprintfmt (putch=0xf01008d2 <putch>, putdat=0xf010fecc, 
    fmt=0xf0101b6e "x %d, y %x, z %d\n", ap=0xf010ff0c "\004")
    at lib/printfmt.c:75
75			return va_arg(*ap, int);
(gdb) c
Continuing.
=> 0xf0100fca <vprintfmt+705>:	mov    0x14(%ebp),%eax

Breakpoint 2, vprintfmt (putch=0xf01008d2 <putch>, putdat=0xf010feec, 
    fmt=0xf010185c "leaving test_backtrace %d\n", ap=0xf010ff24 "")
    at lib/printfmt.c:75
75			return va_arg(*ap, int);
(gdb) 

```

Actually, the function never try to remember the number of arguments we passed in. It only accesses the memory the same times as the number of '%' in the format string.

##### 4. Run the following code.

```
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
```

What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise.

The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616`to a different value?

[Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](http://www.networksorcery.com/enp/ien/ien137.txt).

The output is "Hello World". 

##### 5. In the following code, what is going to be printed after'y=?' (note: the answer is not a specific value.) Why does this happen?

```c
    cprintf("x=%d y=%d", 3);
```

The memory address of the second argument is ought to be `%ebp+8`. However, this address is actually occupied by the local variable of the outer function(caller).

##### 6. Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?

First parse the whole formatted string, then figure out each type of the argument in a reverse order. Then things happen as above. However, there is a severe problem of this solution, if the last argument is lost or has a wrong type, all of the front arguments would not be correctly printed out.

### The Stack

**Exercise 9.** Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

In `kern\entry.S`:

``` assembly
relocated:

	# Clear the frame pointer register (EBP)
	# so that once we get into debugging C code,
	# stack backtraces will be terminated properly.
	movl	$0x0,%ebp			# nuke frame pointer

	# Set the stack pointer
	movl	$(bootstacktop),%esp
	
##############################################################
# boot stack
##############################################################
	.p2align	PGSHIFT		# force page alignment
	.globl		bootstack
bootstack:
	.space		KSTKSIZE
	.globl		bootstacktop   
bootstacktop:
```

This code segmentation set the initial `%ebp` to be `0x0`, and `%esp` to be `$0xf0110000`:

``` assembly
(gdb) b kern/entry.S:77
Breakpoint 1 at 0xf0100034: file kern/entry.S, line 77.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100034 <relocated+5>:	mov    $0xf0110000,%esp

```

So the stack top is now `$0xf0110000`, the reserved space for stack is 32kb:

``` c
// inc/memlayout.h
#define KSTACKTOP	KERNBASE
#define KSTKSIZE	(8*PGSIZE)   		// size of a kernel stack
#define KSTKGAP		(8*PGSIZE)   		// size of a kernel stack guard

// inc/mmu.h
#define PGSIZE 4096
```

In `inc/memlayout.h` , the kernel stack in RAM is described as:

``` c
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
```

**Exercise 10.** To become familiar with the C calling conventions on the x86, find the address of the `test_backtrace` function in `obj/kern/kernel.asm`, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of `test_backtrace` push on the stack, and what are those words?

The definition of the function `test_backtrace` is:

``` c 
// Test the stack backtrace function (lab 1 only)
void
test_backtrace(int x)
{
	cprintf("entering test_backtrace %d\n", x);
	if (x > 0)
		test_backtrace(x-1);
	else
		mon_backtrace(0, 0, 0);
	cprintf("leaving test_backtrace %d\n", x);
}
```

Set breakpoint and debug:

``` assembly
(gdb) b *0xf0100040
Breakpoint 1 at 0xf0100040: file kern/init.c, line 13.

(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0100040 <test_backtrace>:	push   %ebp

Breakpoint 1, test_backtrace (x=5) at kern/init.c:13
13	{
(gdb) p $ebp
$3 = (void *) 0xf010fff8
(gdb) p $esp
$4 = (void *) 0xf010ffdc
(gdb) c
Continuing.
=> 0xf0100040 <test_backtrace>:	push   %ebp

Breakpoint 1, test_backtrace (x=4) at kern/init.c:13
13	{
(gdb) p $ebp
$5 = (void *) 0xf010ffd8
(gdb) p $esp
$6 = (void *) 0xf010ffbc
(gdb) c
Continuing.
=> 0xf0100040 <test_backtrace>:	push   %ebp

Breakpoint 1, test_backtrace (x=3) at kern/init.c:13
13	{
(gdb) p $ebp
$7 = (void *) 0xf010ffb8
(gdb) p $esp
$8 = (void *) 0xf010ff9c
(gdb) c
Continuing.
=> 0xf0100040 <test_backtrace>:	push   %ebp

Breakpoint 1, test_backtrace (x=2) at kern/init.c:13
13	{
(gdb) p $ebp
$9 = (void *) 0xf010ff98
(gdb) p $esp
$10 = (void *) 0xf010ff7c
(gdb) c
Continuing.
=> 0xf0100040 <test_backtrace>:	push   %ebp

Breakpoint 1, test_backtrace (x=1) at kern/init.c:13
13	{
(gdb) p $ebp
$11 = (void *) 0xf010ff78
(gdb) p $esp
$12 = (void *) 0xf010ff5c
(gdb) c
Continuing.
=> 0xf0100040 <test_backtrace>:	push   %ebp

Breakpoint 1, test_backtrace (x=0) at kern/init.c:13
13	{
(gdb) p $ebp
$13 = (void *) 0xf010ff58
(gdb) p $esp
$14 = (void *) 0xf010ff3c
```

From the debugging log above, we could see clearly, each of the recursive procedure means  `0x20(32)` bytes' stack grow, which is  8-bit words. Now take a look at what these stack space contains:

``` assembly
(gdb) x/52x $ebp
0xf010ff38:	0xf010ff58*	0xf0100068	0x00000000^	0x00000001
0xf010ff48:	0xf010ff78	0x00000000	0xf01008d2	0x00000002
0xf010ff58:	0xf010ff78*	0xf0100068	0x00000001^	0x00000002
0xf010ff68:	0xf010ff98	0x00000000	0xf01008d2	0x00000003
0xf010ff78:	0xf010ff98*	0xf0100068	0x00000002^	0x00000003
0xf010ff88:	0xf010ffb8	0x00000000	0xf01008d2	0x00000004
0xf010ff98:	0xf010ffb8*	0xf0100068	0x00000003^	0x00000004
0xf010ffa8:	0x00000000	0x00000000	0x00000000	0x00000005
0xf010ffb8:	0xf010ffd8*	0xf0100068	0x00000004^	0x00000005
0xf010ffc8:	0x00000000	0x00010094	0x00010094	0x00010094
0xf010ffd8:	0xf010fff8*	0xf01000d4	0x00000005^	0x00001aac
0xf010ffe8:	0x00000640	0x00000000	0x00000000	0x00000000
0xf010fff8:	0x00000000	0xf010003e	0x00111021	0x00000000
```

The `*`means the saved `ebp`, and `^` means the parameter of the function. The value between them is the `eip` of the caller(return address). Other spaces in the stack is the reserved space for local variables.

**Exercise 11.** Implement the backtrace function as specified above. Use the same format as in the example, since otherwise the grading script will be confused. When you think you have it working right, run make grade to see if its output conforms to what our grading script expects, and fix it if it doesn't. *After* you have handed in your Lab 1 code, you are welcome to change the output format of the backtrace function any way you like.

``` c
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	// Your code here.
	uint32_t ebp;
	ebp = read_ebp();
	uint32_t *ptr_ebp;
	
	cprintf("Stack backtrace:\n");
	
	while (ebp!=0){
		ptr_ebp = (uint32_t *)ebp;
		cprintf("\tebp %x  eip %x  args %x %x %x %x %x\n",
				ebp, ptr_ebp[1], ptr_ebp[2], ptr_ebp[3], ptr_ebp[4], ptr_ebp[5], ptr_ebp[6]);
		ebp = *ptr_ebp;
	}

	return 0;
}
```

**Exercise 12.** Complete the implementation of `debuginfo_eip` by inserting the call to stab_binsearch to find the line number for an address. Add a backtrace command to the kernel monitor, and extend your implementation of `mon_backtrace` to call `debuginfo_eip` and print a line for each stack frame of the form:

```swift
K> backtrace
Stack backtrace:
  ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
         kern/monitor.c:143: monitor+106
  ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
         kern/init.c:49: i386_init+59
  ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
         kern/entry.S:70: <unknown>+0
K>
```

