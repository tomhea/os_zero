# Binary Tests

In this notebook I'll document every test that gets done in the critical Binary phase - The phase which I'll encode every opcode to 0/1 by hand.

Tests here are crucial, so I'll be having them as fast as I can.

I decided that ALL the tests will be automatic tests, meaning that even the testing functions themselves will be checked for both expected "test success" and "test failure".
For this to work, I added the `test_print_on_failure` global, so that I could turn it off during the "testing functions tests", thus checking the global `tests_success` instead of manually looking at the output myself.

The `test_print_on_failure` global will initially be set to false, until, and when the "testing functions tests" phase will end it will be set to true.

Notes:
- The address ranges are "where are the test in memory", in hex, and they here won't include the last byte.
- The binary tests have a "First code" section reserved for them, meaning addresses 80002000-80004000.
    - The Testing-Functions tests are in 80002000-80002400, and the sysX tests are at 80002400-80004000.
- There is a code that prints wether all test passed, and it's located in `80002040`.
- I added some nops between tests so that there will be at least one nop between tests, and that each test will bw aligned to 0x10.

# Testing-Functions Tests (80002000-80002400):

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

#### Test sanity `store_all_regs` and `put_regs_test_values`: (80002040-80002080)
```assembly
jalr x1, 1B0(gp)
jalr x1, 1C0(gp)
addi a3, zero, 0x20
addi a2, zero, 0
addi a4, sp, 0    // memory pointer
addi a1, zero, 4  // register index
addi a5, zero, 9

// LOOP:
lw a0, 0(a4)
bne a1, a5, 8
addi a0, a0, 8  // make it 9 from 1
jalr x1, 0x1A0(gp)
addi a1, a1, 1
addi a4, a4, 4
bne a1, a3, FE8

jalr x1, 1D0(gp)
addi sp, sp, 0x70
```

#### Test failure `store_all_regs` and `put_regs_test_values`: (80002090-800020D4)
```assembly
jalr x1, 1B0(gp)
addi a3, zero, 0x20
jalr x1, 1C0(gp)
addi a2, zero, 0
addi a4, sp, 0    // memory pointer
addi a1, zero, 4  // register index
addi a5, zero, 9

// LOOP:
lw a0, 0(a4)
bne a1, a5, 8
addi a0, a0, 8  // make it 9 from 1
jalr x1, 0x1A0(gp)
addi a1, a1, 1
addi a4, a4, 4
bne a1, a3, FE8

jalr x1, 1D0(gp)
addi sp, sp, 0x70
```

#### TestTemplate "PSSV": `put, store, store, validate`:
```assembly
jalr x1, 0x1B0(gp)
jalr x1, 0x1C0(gp)
// Filled1[opcodes]
jalr x1, 0x1C0(gp)
// Filled2[immidiate] {
    lui a0, UPPER
    addi a0, a0, LOW
}
jalr x1, 0x1F0(gp)
// Filled3[Success/Failure]: jalr x1, 0x1D0(gp) / jalr x1, 0x1E0(gp)
```

#### Test PSSV sanity nothing changed (800020E0-800020FC):
```assembly
// Implements TestTemplate "PSSV".
Filled1 = {
    // Empty
}
Filled2 = FFFFF|FF0.
Filled3 = Success.
```

#### Test PSSV success mask_off_first (80002100-80002120):
```assembly
// Implements TestTemplate "PSSV".
Filled1 = {
    addi x4, zero, 0x123
}
Filled2 = FFFFF|FE0.
Filled3 = Success.
```

#### Test PSSV success mask_off_last (80002130-80002150):
```assembly
// Implements TestTemplate "PSSV".
Filled1 = {
    addi x31, zero, 0x123
}
Filled2 = 7FFFF|FF0.
Filled3 = Success.
```

#### Test PSSV failure mask_on_first (80002160-80002180):
```assembly
// Implements TestTemplate "PSSV".
Filled1 = {
    addi x4, zero, 0x123
}
Filled2 = 00000|010.
Filled3 = Failure.
```

#### Test PSSV failure mask_on_last (80002190-800021B0):
```assembly
// Implements TestTemplate "PSSV".
Filled1 = {
    addi x31, zero, 0x123
}
Filled2 = 80000|000.
Filled3 = Failure.
```

#### Test PSSV success mask_off_twice (800021C0-800021E0):
```assembly
// Implements TestTemplate "PSSV".
Filled1 = {
    addi x4, zero, 0x123
    addi x5, zero, 0x124
}
Filled2 = 00000|FC0.
Filled3 = Success.
```

#### Test PSSV failure mask_on_twice (800021F0-80002210):
```assembly
// Implements TestTemplate "PSSV".
Filled1 = {
    addi x4, zero, 0x123
    addi x5, zero, 0x124
}
Filled2 = 00000|030.
Filled3 = Failure.
```

#### Test validation don't modify non-a regs (80002220-80002248):
```assembly
jalr x1, 0x1B0(gp)
jalr x1, 0x1C0(gp)
jalr x1, 0x1C0(gp)
jalr x1, 0x1C0(gp)
jalr x1, 0x1F0(gp)
jalr x1, 0x1C0(gp)
lui a0, 0xFFFC0
addi a0, a0, 0x3F0
jalr x1, 0x1F0(gp)
jalr x1, 0x1D0(gp)
```

#### Test sanity demi-function complete validation (80002250-8000227C):
```assembly
jalr x1, 0x1B0(gp)
jalr x1, 0x1C0(gp)
addi a0, zero, 0x123
jalr x1, 0x1C0(gp)
addi a1, zero, 0x123
addi a2, zero, 0
jalr x1, 0x1A0(gp)
lui a0, 0x00000
addi a0, a0, 0xBF0
jalr x1, 0x1F0(gp)
jalr x1, 0x1D0(gp)  // Real implementations won't need it, as 
```

#### Finishing code for these tests: (80002280-80002294)
```assembly
lw x1, FE0(gp)
sw x1, FE8(gp)
sw s1, FE4(gp)
lui x1, 0x80002
jalr x0, 0x400(x1)
```



# Regular Funtions Tests (80002400-80004000)

#### Prep for these tests: (80002400-80002414)
```assembly
lw a0, 0xFEC(gp)
andi a0, a0, 1      // sys-tests flag
bne a0, zero, 12
lui x1, 0x80010000
jalr x0, 0(x1)
```

### Output tests (80002420-80002580)

#### Prep for these tests: (80002420-80002434)
```assembly
lw a0, 0xFEC(gp)
andi a0, a0, 0x200  // output tests flag
bne a0, zero, 12
lui x1, 0x80002
jalr x0, 0x580(x1)
```

#### Test `putc` actual char: (80002440-80002460)
```assembly
jalr x1, 1B0(gp)
addi a0, zero, 0x62
jalr x1, 1C0(gp)
jalr x1, 0(gp)
jalr x1, 1C0(gp)
lui a0, 0x00000
addi a0, a0, 0xF90  // all but t0,t1
jalr x1, 1F0(gp)
```

#### Test `putc` null byte: (80002470-80002490)
```assembly
jalr x1, 1B0(gp)
addi a0, zero, 0x0
jalr x1, 1C0(gp)
jalr x1, 0(gp)
jalr x1, 1C0(gp)
lui a0, 0x00000
addi a0, a0, 0xF90  // all but t0,t1
jalr x1, 1F0(gp)
```

#### Test `puts` actual string: (800024A0-800024C0) (v0)
#### Test `puts` empty string: (800024D0-800024F0)  (v1)
```assembly
jalr x1, 1B0(gp)
(v0) addi a0, gp, 0x8D4
(v1) addi a0, gp, 0x8D8
jalr x1, 1C0(gp)
jalr x1, 20(gp)
jalr x1, 1C0(gp)
lui a0, 0x00000
addi a0, a0, 0xFF0  // all
jalr x1, 1F0(gp)
```

#### Test `putx` sanity: (80002500-80002524)
```assembly
jalr x1, 1B0(gp)
lui a0, 0xABCF0
addi a0, a0, 0x129
jalr x1, 1C0(gp)
jalr x1, 130(gp)
jalr x1, 1C0(gp)
lui a0, 0x00000
addi a0, a0, 0xFF0  // all
jalr x1, 1F0(gp)
```

### Input tests (80002580-???)

#### Prep for these tests: (80002580-80002594)
```assembly
lw a0, 0xFEC(gp)
andi a0, a0, 0x400  // input tests flag
bne a0, zero, 12
lui x1, 0x80010  // TODO replace with "After input tests fnished" address.
jalr x0, 0(x1)
```

#### Test `availc` false: (800025A0-???) - NOT IMPLEMENTED, NOT PROGRAMMED.
```assembly

```

#### Test `availc` true: (???-???) - NOT IMPLEMENTED, NOT PROGRAMMED.
```assembly

```


#### Test `putc` actual char: (???-???) - NOT IMPLEMENTED, NOT PROGRAMMED.
```assembly
jalr x1, 1B0(gp)
addi a0, zero, 0x62
jalr x1, 1C0(gp)
jalr x1, 0(gp)
jalr x1, 1C0(gp)
lui a0, 0x00000
addi a0, a0, 0xF90  // all but t0,t1
jalr x1, 1F0(gp)
```



### Finishing code for the sys tests: (???-???)
```assembly
lui x1, 0x80010
jalr x0, 0(x1)
```

### Main starting code: (80010000-8001001C)
```assembly
// Print `tests_success` and exit.
addi a0, gp, 0x840
jalr x1, 20(gp)
lw a0, FE8(gp)
jalr x1, 130(gp)
addi a0, gp, 0x834
jalr x1, 20(gp)
jalr x1, 70(gp)
```
