# Bare Assembly

In this notebook I'll write about the updated memory bits and bytes, at least until I could code in actual assembly.

In this notebook you could find the TEXTUAL ASSEMBLY (WOW) that I personally encoded by hand into the binary blob.


## Memory Regions
- Boot code: At `80000000`.
- Fast syscalls: `80001000:80001800`, so that's `gp:gp+800`. `putc` is at `gp+0`. I'll use the shortcut sysX for `gp+X`, so `putc` is `sys0`.
- Global data: `80000800:80001000`, so `gp-800:gp`. I'll use the shortcut `g_X` for `gp-1000+X`.
- First code: At `80002000`.
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

`putc(c: char_a0) -> None` - `sys0` (uses `t0,t1`, keeps `a0`).
```assembly
// Prints the given byte.
lui t0, 0x10000
lb t1, 5(t0)
andi t1, t1, 0x20
beq t1, x0, -8
sb a0, 0(t0)
ret
```

`puts(s: pointer_a0) -> None` - `sys20` (keep all regs).
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

`exit() -> NO_RETURN` - `sys70` (infinite loop).
```assembly
// Self loop.
beq zero, zero, 0
```

`getc() -> char_a0` - `sys80` (uses `t0,t1`).
```assembly
// Get a char from stdin (blocking).
lui t0, 0x10000
lb t1, 5(t0)
andi t1, t1, 0x1
beq t1, x0, -8
lbu a0, 0(t0)
ret
```

`availc() -> bool_a0` - `sysA0` (keep all regs) - **NOT CHECKED**.
```assembly
// Checks if there is an input byte ready to be read.
lui a0, 0x10000
lb a0, 5(a0)
andi a0, a0, 0x1
ret
```

`gets(out_s: pointer_a0, max_len: a1) -> None` - `sysB0` (keep all regs).
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

`putx(hex: a0) -> None` - `sys130` (keep all regs).
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



## globals
- `boot_a0` - `g_FF4` - holds the initial a0, hart-id (hardware thread id).
- `boot_a1` - `g_FF8` - holds the initial a1, the device tree.
- `boot_a2` - `g_FFC` - holds the initial a2, the `struct fw_dynamic_info`.
- `multicore_boot_address - `g_FF0` - holds the address that the non-0 harts will jump to on an interrupt.
- `testing_mask` - `g_FEC` - holds the "which tests to run" flags. 0xFFFFFFFF means to run all tests.
    - bit `0` - Run the `sys` test suite.
    - bit `30` - Run tests that output things.
    - bit `31` - Run tests that require specific input.
