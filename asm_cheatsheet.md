# Assembly Cheat-Sheet

Here I'll write encodings that appear a lot of times:

Stack:
```
sw R, off(sp):        00 R1 2{off/2} 23
- For R>=16, 00 becomes 01.
- For off>0x20, it also flows into the 00 (7 msb). off%2==0.

addi sp, sp, off:     of f1 01 13

lw R, off(sp):        of f1 2{R/2} {8/0}3
- Choose 83 for R%2==1, or 03 for R%2==0.
```

Loading values:
```
addi R, zero, imm:    im m0 0{R/2} {9/1}3
lw R, off(gp)         of f1 A{R/2} {8/0}3
addi R, gp, off:      of f1 8{R/2} {9/1}3
```

Parametrized tests:
```
lw s0, 0(sp):         00 81    20    23
lw R, off(s0):        00 R4 2{off/2} 23
```
