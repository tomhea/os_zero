# Bare Assembly

In this notebook I'll write about the updated memory bits and bytes, at least until I could code in actual assembly.

In this notebook you could find the TEXTUAL ASSEMBLY (WOW) that I personally encoded by hand into the binary blob.


## Memory Regions
- Boot code: At `80000000`.
- Fast syscalls: `80001000:80001800`, so that's `gp:gp+800`. `putc` is at `gp+0`. I'll use the shortcut sysX for `gp+X`, so `putc` is `sys0`.
- Global data: `80000800:80001000`, so `gp-800:gp`. I'll use the shortcut `g_X` for `gp-1000+X`.
- First code: At `80002000` (Used for sys-funcs tests). When finished jumps to "Assembler code".
- Syscalls implementations: `80004000:80010000`. Some functions in the `Fast syscalls` sections are "jumpers" into this section.
- Assembler code: At `80010000` and onwards.
- Stack: `FFFFFFF8` and downwords.

Notes:
- The Ram is `80000000:100000000`.
- All are hexadecimal numbers. 
- `X:Y` means addresses from `X` up to `Y`, not including `Y`.
- `gp=80001000` until there is a real os.


## Boot code:

The boot code is the following code:
```assembly
lui gp, 0x80001
sw a0, 0xff4(gp)
sw a1, 0xff8(gp)
sw a2, 0xffc(gp)

slli sp, a0, 0x14
sub sp, zero, sp
addi sp, sp, -8
bne a0, zero, 12

lui t0, 0x80002
jalr zero, 0(t0)   // hart0 jumps to First code

auipc t0, 0
addi t0, t0, 0xc
sw t0, 0xff0(gp)

wfi                // hart1+ idle loop
lw t0, 0xff0(gp)
jalr zero, 0(t0)
```


## Syscalls:
All the implemented syscalls, in order:

`putc(c: char_a0) -> None` - `sys0` (uses `t0,t1`, keeps `a0`)
```assembly
// Prints the given byte.
lui t0, 0x10000
lb t1, 5(t0)
andi t1, t1, 0x20
beq t1, x0, -8
sb a0, 0(t0)
ret
```

`puts(s: pointer_a0) -> None` - `sys20` (keep all regs)
```assembly
// Prints a string.
addi sp, sp, -0x14
sw x1, 0(sp)
sw x5, 4(sp)
sw x6, 8(sp)
sw x7, 0xc(sp)
sw x10, 0x10(sp)

addi x1, gp, 40  // return to loop:
addi t2, a0, 0
//loop:
lbu a0, 0(t2)
addi t2, t2, 1
bne a0, x0, -0x48  // Offset to sys0

lw x1, 0(sp)
lw x5, 4(sp)
lw x6, 8(sp)
lw x7, 0xc(sp)
lw x10, 0x10(sp)
addi sp, sp, 0x14
ret
```

`exit() -> NO_RETURN` - `sys70` (infinite loop) (**ATOMIC**)
```assembly
// Self loop.
beq zero, zero, 0
```

`getc() -> char_a0` - `sys80` (uses `t0,t1`)
```assembly
// Get a char from stdin (blocking).
lui t0, 0x10000
lb t1, 5(t0)
andi t1, t1, 0x1
beq t1, x0, -8
lbu a0, 0(t0)
ret
```

`availc() -> bool_a0` - `sysA0` (keep all regs)
```assembly
// Checks if there is an input byte ready to be read.
lui a0, 0x10000
lb a0, 5(a0)
andi a0, a0, 0x1
ret
```

`gets(out_s: pointer_a0, max_len: a1) -> None` - `sysB0` (keep all regs)
```assembly
// Reads at most max_len-1 bytes from input, and writes them as a string to the given buffer.
// Also stops at a newline. Ends the string with a null-char (and removes the ending newline).
// Note: prepare a buffer of size max_len chars.
// Note prints every char it gets.
addi sp, sp, -0x18
sw x1, 0(sp)
sw x5, 4(sp)
sw x6, 8(sp)
sw x7, 0xc(sp)
sw x10, 0x10(sp)
sw x11, 0x14(sp)

addi t2, a0, -1
add a1, a1, t2
//LOOP:
addi t2, t2, 1
beq t2, a1, put_null_char_finish (+0x2C)
jalr x1, 0x80(gp)
addi t0, zero, 0xA
beq a0, t0, print_newline_and_finish (+0x18)
addi t0, zero, 0xD
beq a0, t0, print_newline_and_finish (+0x10)
sb a0, 0(t2)
jalr x1, 0(gp)
beq zero, zero, LOOP (-0x24)

//print_newline_and_finish:
addi a0, zero, 0xA
jalr x1, 0(gp)
//put_null_char_finish:
sb zero, 0(t2)

lw x1, 0(sp)
lw x5, 4(sp)
lw x6, 8(sp)
lw x7, 0xc(sp)
lw x10, 0x10(sp)
lw x11, 0x14(sp)
addi sp, sp, 0x18
ret
```

`putx(hex: a0) -> None` - `sys130` (keep all regs)
```assembly
// This function prints a0 as an 8-digit hexadecimal number (first char is the most-significant hex digit).
addi sp, sp, -0x1C
sw x1, 0(sp)
sw x5, 4(sp)
sw x6, 8(sp)
sw x7, 0xc(sp)
sw x10, 0x10(sp)
sw x11, 0x14(sp)
sw x12, 0x18(sp)

addi a1, a0, 0
addi x7, x0, 0x20
addi x12, zero, 0x3A

//loop:
addi x7, x7, 0xFFC
srl a0, a1, x7
andi a0, a0, 0xF
addi a0, a0, 0x30
blt a0, a2, 8
addi a0, a0, 0x7
jalr x1, 0(gp)
bne x7, x0, -0x1C

lw x1, 0(sp)
lw x5, 4(sp)
lw x6, 8(sp)
lw x7, 0xc(sp)
lw x10, 0x10(sp)
lw x11, 0x14(sp)
lw x12, 0x18(sp)
addi sp, sp, 0x1C
ret
```

`assert_ret(ret_val: a0, expected: a1, string_testname: a2) -> None` - `sys1A0` JUMPER_4000 (keep all regs)
```assembly
// This function checks if a0 equals a1, and if not - prints this test failure (a0, a1, x1 values, and a2 message if not null).
// If test_print_on_failure global is off - Don't print anything (also ignore a2), just update the tests_success global if needed.
bne a0, a1, 0xC
lw x1, FFC(x2)
jalr x0, 0(x1)

sw zero, FE8(gp)
lw x1, FE4(gp)
beq x1, zero, FF0

addi sp, sp, 0xFF8
sw a0, 0(sp)

addi a0, gp, 0x800
jalr x1, 0x20(gp)
beq a2, zero, 0xC
addi a0, a2, 0
jalr x1, 0x20(gp)

addi a0, gp, 0x810
jalr x1, 0x20(gp)
lw a0, 4(sp)
jalr x1, 0x130(gp)

addi a0, gp, 0x818
jalr x1, 0x20(gp)
addi a0, a1, 0
jalr x1, 0x130(gp)

addi a0, gp, 0x828
jalr x1, 0x20(gp)
lw a0, 0(sp)
jalr x1, 0x130(gp)

addi a0, gp, 0x834
jalr x1, 0x20(gp)

lw a0, 0(sp)
lw x1, 4(sp)
addi sp, sp, 0x8
jalr x0, 0(x1)
```

`put_regs_test_values() -> None` - `sys1B0` JUMPER_4080 (modifies all regs)
```assembly
// Loads the immediate N for each register xN (for N>=4, N<=31), but s1==1.
addi x4, zero, 4
addi x5, zero, 5
...
addi x31, zero, 31

lw x1, FFC(sp)
jalr x0, 0(x1)
```

`store_all_regs() -> None` - `sys1C0` JUMPER_4100 (keep all regs except sp)
```assembly
lw x1, FFC(sp)
addi sp, sp, 0xF90
sw x4, 0(sp)
sw x5, 4(sp)
...
sw x31, 0x6C
jalr x0, 0(x1)
```

`assert_test_success() -> None` - `sys1D0` JUMPER_4180 (modifies a0, relies on `s1==1`) (**ATOMIC**)
```assembly
// Prints error if global `tests_success` in off, and anyway sets it on again.
lw x1, FFC(x2)
lw a0, FE8(gp)
beq a0, s1, 0x14

sw zero, FE0(gp)
sw s1, FE8(gp)
addi a0, gp, 0x850
jalr x0, 0x20(gp)

jalr x0, 0(x1)
```

`assert_test_failure() -> None` - `sys1E0` JUMPER_41A0 (modifies a0, relies on `s1==1`) (**ATOMIC**)
```assembly
// Prints error if global `tests_success` in on, and anyway sets it on again.
lw x1, FFC(x2)
lw a0, FE8(gp)
beq a0, zero, 0x10

sw zero, FE0(gp)
addi a0, gp, 0x884
jalr x0, 0x20(gp)

sw s1, FE8(gp)
jalr x0, 0(x1)
```

`validate_all_regs_unchanged(regs_mask: a0) -> None` - `sys1F0` JUMPER_41C0 (modifies the a-registers, and sp)
```assembly
// This function compares RegsAfter and RegsBefore, and only compares according to regs_mask (if the Nth bit is on, then compares xN with its Before&After values). Print if compared a register and it was changed.
// RegsAfter (x4-x31) are at the latest 0x70 bytes of stack, and RegsBefore (x4-x31) are at the next 0x70 bytes of the stack.
// This function is responsible to increment sp such that both RegsBefore and RegsAfter will be removed.

addi a1, sp, 0      // current_reg in RegsAfter (+0x70 will get to current RegBefore)
addi a2, sp, 0x70   // End of RegsAfter
addi a3, zero, 4    // index of current register (used for printing)
lw a4, FFC(sp)   // store x1
srli a5, a0, 4   // shifted flags

//LOOP:
andi a6, a0, 1
beq a6, zero, NextIter (+0x74)
lw a7, 0(a1)
lw a6, 0x70(a1)
beq a6, a7, NextIter (+0x68)

sw zero, FE8(gp)
lw x1, FE4(gp)
beq x1, zero, NextIter (+0x5C)

addi a0, gp, 0x800
jalr x1, 0x20(gp)

addi a0, gp, 0x8B8
jalr x1, 0x20(gp)
addi a0, a3, 0
jalr x1, 0x130(gp)
addi a0, gp, 0x8C4
jalr x1, 0x20(gp)

addi a0, gp, 0x810
jalr x1, 0x20(gp)
addi a0, a4, 0
jalr x1, 0x130(gp)

addi a0, gp, 0x818
jalr x1, 0x20(gp)
addi a0, a6, 0
jalr x1, 0x130(gp)

addi a0, gp, 0x828
jalr x1, 0x20(gp)
addi a0, a7, 0
jalr x1, 0x130(gp)

addi a0, gp, 0x834
jalr x1, 0x20(gp)

//NextIter:
srli a5, a5, 1
addi a3, a3, 1
addi a1, a1, 4
bne a1, a2, LOOP (-0x84, F7C)

addi sp, sp, 0xE0
jalr x0, 0(a4)
```

`brk(pointer: a0) -> a0` - `sys200` JUMPER_4270 (keep all regs)
```assembly
// Sets binary_heap_top to a0, return 0 on success, or -1 on failure (argument not inside the binary-heap segment).
lw x1, FA8(gp)
bgeu a0, x1, 0xC
addi a0, zero, 0xFFF
beq zero, zero, 0x14

lw x1, FAC(gp)
bgeu a0, x1, -0xC

sw a0, FA4(gp)
addi a0, zero, 0

lw x1, 0xFFC(sp)
ret
```

`sbrk(incr: a0) -> pointer_a0` - `sys210` JUMPER_42A0 (keep all regs)
```assembly
// Increments binary_heap_top by a0, and return the previous top on success, or -1 on failure (if next top won't reside inside the binary-heap segment).
addi sp, sp, -0x8
sw x11, 0x0(sp)

lw a1, FA4(gp)
add a0, a0, a1

jalr x1, 200(gp)
bne a0, zero, 8
addi a0, a1, 0

lw x11, 0x0(sp)
lw x1, 0x4(sp)
addi sp, sp, 0x8
ret
```

`strncmp(str1: a0, str2: a1, n: a2) -> a0` - `sys220` JUMPER_42D0
```assembly
// Compares Lexicographically the first n bytes of str1 and str2. 
// Returns negative if str1 is before str2, zero if they are the same, and positive is str1 is after str2.
// Stops on '\0'.
addi sp, sp, FEC
sw t0, 0x0(sp)
sw t1, 0x4(sp)
sw a1, 0x8(sp)
sw a2, 0xC(sp)

add a2, a0, a2
addi t1, zero, 1
addi t0, zero, 0  // length 0 should succeed

//LOOP:
beq t1, zero, END (+0x20)
beq a0, a2, END (+0x1C)
lb t0, 0(a0)
lb t1, 0(a1)
sub t0, t0, t1
addi a0, a0, 1
addi a1, a1, 1
beq t0, zero, LOOP (-0x1C)

//END:
addi a0, t0, 0

lw t0, 0x0(sp)
lw t1, 0x4(sp)
lw a1, 0x8(sp)
lw a2, 0xC(sp)
lw x1, 0x10(sp)
addi sp, sp, 0x14
ret
```

`DICT_initialize(buffer: a0) -> None` - `sys230` JUMPER_4340
```assembly
// Expects a 0x400 bytes buffer, and initializes it as a new dictionary.
addi x1, a0, 0x3FC

sw zero, 0(x1)
addi x1, x1, FFC
bgeu x1, a0, FF8

lw x1, FFC(sp)
ret
```

`DICT_calculate_key(str: a0, n: a1) -> a0` - `sys240` JUMPER_4360 - Calculate the key
```assembly
// Returns the hash of the given string.
// Hash algorithm: sum (modulo 256) of hashes of chars, which is: (str[i] ^ (0x53*(i+1))). 
addi sp, sp, FEC
sw t0, 0x0(sp)
sw t1, 0x4(sp)
sw t2, 0x8(sp)
sw a1, 0xC(sp)

addi t0, zero, 0x53  // The MAGIC
add a1, a0, a1       // str_end
addi t1, zero, 0     // current result

beq a0, a1, END (+0x1C)
//LOOP:
lbu t2, 0(a0)
xor t2, t2, t0
add t1, t1, t2
addi t0, t0, 0x53    // The MAGIC
addi a0, a0, 1
bltu a0, a1, LOOP (-0x14)

//END:
andi a0, t1, 0xFF
lw t0, 0x0(sp)
lw t1, 0x4(sp)
lw t2, 0x8(sp)
lw a1, 0xC(sp)
lw x1, 0x10(sp)
addi sp, sp, 0x14
ret
```

Notes:
1. JUMPER: The next function is called from a jumper code, meaning a code that saved `x1` to the stack without decrementing the stack pointer, and then jumped right here. The opcodes that will be written here won't contain the jumping code itself.
   - JUMPER_XXXX means that the implementation is stored at 0x8000XXXX.
2. ATOMIC: Doesn't have automatic tests.



## globals
- `boot_a0` - `g_FF4` - holds the initial a0, hart-id (hardware thread id).
- `boot_a1` - `g_FF8` - holds the initial a1, the device tree.
- `boot_a2` - `g_FFC` - holds the initial a2, the `struct fw_dynamic_info`.
- `multicore_boot_address - `g_FF0` - holds the address that the non-0 harts will jump to on an interrupt.
- `testing_mask` - `g_FEC` - holds the "which tests to run" flags. 0x7FF means to run all tests.
    - bit `0` - Run the `sys` test suite.
    - bit `1` - Run the binary assembler full tests.
    - bit `2` - Run the assembly assembler full tests.
    - bit `3` - Run the assembly compiler tests.
    - bit `4` - Run the C compiler tests.
    - bits `5-8` - Reserved.
    - bit `9` - Run tests that output things.
    - bit `10` - Run tests that require specific input.
- `tests_success` - `g_FE8` - initialized with 1, and any test failure set it to 0.
- `test_print_on_failure` - `g_FE4` - if flase (0), the testing functions doesn't print on test failures, just update the `tests_success` boolean (and on 1 they both print & update).
- `testing_tests_success` - `g_FE0` - if true (1), the testing of the testing functions was successful (0 otherwise).
- `gets_tests_buffers` - g_FB0 (0x30 bytes).
    - `g_FB0` (0x10 bytes) - expected buffer for gets() max-len test.
    - `g_FC0` (0x10 bytes) - expected buffer for gets() newline test.
    - `g_FD0` (0x10 bytes) - working buffer for both gets() tests.
- `binary_heap_segment_end` - `g_FAC` - Constant 0x82000000.
- `binary_heap_segment_start` - `g_FA8` - Constant 0x81000000.
- `binary_heap_top` - `g_FA4` - Initialized with "segment_start", and represent the first unallocated byte in the binary_heap.
