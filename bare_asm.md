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

`gets(out_s: pointer_a0, max_len: a1) -> None` - `sysB0` (keep all regs) - **NOT FINISHED**.
```assembly
// Writes a string from input to the given buffer, finishes when reached a newline / max_len-1. 
// Ends the string with a null-char (and removes the ending newline).
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

## globals
- `boot_a0` - `g_FF4` - holds the initial a0, hart-id (hardware thread id).
- `boot_a1` - `g_FF8` - holds the initial a1, the device tree.
- `boot_a2` - `g_FFC` - holds the initial a2, the `struct fw_dynamic_info`.
- `multicore_boot_address - `g_FF0` - holds the address that the non-0 harts will jump to on an interrupt.
