# Binary Tests

In this notebook I'll document every test that gets done in the critical Binary phase - The phase which I'll encode every opcode to 0/1 by hand.

Tests here are crucial, so I'll be having them as fast as I can.

I decided that ALL the tests will be automatic tests, meaning that even the testing functions themselves will be checked for both expected "test success" and "test failure".
For this to work, I added the `test_print_on_failure` global, so that I could turn it off during the "testing functions tests", thus checking the global `tests_success` instead of manually looking at the output myself.

The `test_print_on_failure` global will initially be set to false, until, and when the "testing functions tests" phase will end it will be set to true.

Notes:
- The address ranges are "where are the test in memory", in hex, and they here won't include the last byte.
- The binary tests have a "First code" section reserved for them, meaning addresses 80002000-80004000.
- There is a code that prints wether all test passed, and it's located in `80002040`.

# Testing-Functions Tests:

#### Prep for these tests: (80002000-80002010)
```assembly
addi s1, zero, 1
sw zero, FE4(gp)
nop  // OP 13000000
nop
```

#### Test `assert_ret`: (80002010-80002030)
```assembly
addi a0, zero, 0x123
addi a1, zero, 0x123
jalr x1, 1A0(gp)
jalr x1, 1D0(gp)

addi a0, zero, 0x122
addi a1, zero, 0x123
jalr x1, 1A0(gp)
jalr x1, 1E0(gp)
```

#### Test sanity `store_all_regs` and `put_regs_test_values`: (80002030-80002068)
```assembly
jalr x1, 1B0(gp)
jalr x1, 1C0(gp)
addi a3, zero, 0x20
addi a2, zero, 0
addi a4, sp, 0    // memory pointer
addi a1, zero, 4  // register index

// LOOP:
lw a0, 0(a4)
jalr x1, 0x1A0(gp)
addi a1, a1, 1
addi a4, a4, 4
bne a1, a3, FF0  // LOOP

addi s1, zero, 1
jalr x1, 1D0(gp)
addi sp, sp, 0x70
```

#### Test failure `store_all_regs` and `put_regs_test_values`: (800020A0)
```assembly
jalr x1, 1B0(gp)
addi s1, zero, 1
jalr x1, 1C0(gp)
addi a3, zero, 0x20
addi a2, zero, 0
addi a4, sp, 0    // memory pointer
addi a1, zero, 4  // register index

// LOOP:
lw a0, 0(a4)
jalr x1, 0x1A0(gp)
addi a1, a1, 1
addi a4, a4, 4
bne a1, a3, FF0  // LOOP

jalr x1, 1D0(gp)
addi sp, sp, 0x70
```



#### Finishing code for these tests: (???-???)
```assembly
lw x1, FE0(gp)
sw x1, FE8(gp)
sw s1, FE4(gp)

// Print `tests_success` and exit.
addi a0, gp, 0x840
jalr x1, 20(gp)
lw a0, FE8(gp)
jalr x1, 130(gp)
addi a0, gp, 0x834
jalr x1, 20(gp)
jalr x1, 70(gp)
```

#### Test `put, store, store, validate` (800020A0-???):
```assembly

```

#### Test sanity demi-function complete validation (???-???):
```assembly

```



# Regular Funtions Tests


