# Assembly Cheat-Sheet
Here I'll write encodings that appear a lot of times:

Stack:
```assembly
sw R, off(sp):        00 R1 2{off/2} 23
- For R>=16, 00 becomes 01.
- For off>0x20, it also flows into the 00 (7 msb). off%2==0.

addi sp, sp, off:     of f1 01 13

lw R, off(sp):        of f1 2{R/2} {8/0}3
- Choose 83 for R%2==1, or 03 for R%2==0.
```

Loading values:
```assembly
addi R, zero, imm:    im m0 0{R/2} {9/1}3
lw R, off(gp)         of f1 A{R/2} {8/0}3
addi R, gp, off:      of f1 8{R/2} {9/1}3
```

Call functions:
```
jalr x1, 0xXY0(gp):   E78001XY (correct order)
ret:                  67800000 (correct order)
```

Parametrized tests:
```assembly
lw R, off(s0):        of f4 2{R/2} {8/0}3
```

Examining Values:
```assembly
addi a0, R, 0         00 0{R/2} {8/0}5 13
```
