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
- The (v0)(v1), and the TemplateFunc, are to allow me to specify very similar functions, without duplicating the documentation here.
- All the tests have a generic 0x100 bytes stack frame already allocated for them (so they can use `0(sp)` up to `0x3FF(sp)`).

# Testing-Functions Tests (80002000-80002400):

#### Prep for these tests: (80002000-80002010)
```assembly
addi s1, zero, 1
sw zero, FE4(gp)
addi sp, sp, 0xC00  // Reserve 0x400 bytes generic stack frame for all tests.
nop  // OP 13000000
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
// Filled2[immidiate] {  // Note that on the case that LOW&0x800 is on, it decrements. I decided that for reading-simplicitly - I'll ignore that (meaning that you'll need to specify FFFFF|FFF for 0xFFFFFFFF, and I'll do the actual calculation and get to UPPER=00000,LOW=FFF in the encoding phase).
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



# Regular SYS Funtions Tests (80002400-80004000)

These tests can be skipped if bit0 in `testing_mask` is off.

#### Prep for these tests: (80002400-80002414)
```assembly
lw a0, 0xFEC(gp)
andi a0, a0, 1      // sys-tests flag
bne a0, zero, 12
lui x1, 0x80010000
jalr x0, 0(x1)
```

### Output tests (80002420-80002580)

These tests (beside avoid "TEST FAIL" print) should print "bog ABCF0129".

These tests can be skipped if bit9 in `testing_mask` is off.

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

#### Test `puts` actual string: (800024A0-800024C0)  (v0)
#### Test `puts` empty string:  (800024D0-800024F0)  (v1)
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

### Input tests (80002580-80002780)

These tests (beside avoid "TEST FAIL" print):
- Expect the next input "code\n" + "maxlen", and wait about 0.1s to start the first input.
- Prints "de\n" + "maxlen".

These tests can be skipped if bit10 in `testing_mask` is off.

#### Prep for these tests: (80002580-80002594)
```assembly
lw a0, 0xFEC(gp)
andi a0, a0, 0x400  // input tests flag
bne a0, zero, 12
lui x1, 0x80002
jalr x0, 0x780(x1)
```

#### Test `availc` false: (800025A0-800025C8)
```assembly
jalr x1, 1B0(gp)
jalr x1, 1C0(gp)
jalr x1, A0(gp)
jalr x1, 1C0(gp)
addi a1, zero, 0
addi a2, zero, 0
jalr x1, 1A0(gp)
lui a0, 0x00000
addi a0, a0, 0xBF0  // all but a0
jalr x1, 1F0(gp)
```

#### Test `availc` true: (800025D0-80002608)
```assembly
lui t0, 0x10000
lb t1, 5(t0)
andi t1, t1, 0x1
beq t1, x0, -8    // wait until there input available

jalr x1, 1B0(gp)
jalr x1, 1C0(gp)
jalr x1, A0(gp)
jalr x1, 1C0(gp)
addi a1, zero, 1
addi a2, zero, 0
jalr x1, 1A0(gp)
lui a0, 0x00000
addi a0, a0, 0xBF0  // all but a0
jalr x1, 1F0(gp)
```

#### Test `getc` char-ready:    (80002610-80002638)  (v0)
#### Test `getc` wait-for-char: (80002640-80002668)  (v1)
```assembly
// Assumes v1 happens right after v0, which happens right after Test-availc-true.
jalr x1, 1B0(gp)
jalr x1, 1C0(gp)
jalr x1, 80(gp)
jalr x1, 1C0(gp)
(v0) addi a1, zero, 0x63
(v1) addi a1, zero, 0x6F
addi a2, zero, 0
jalr x1, 1A0(gp)
lui a0, 0x00000
addi a0, a0, 0xB90  // all but t0,t1,a0
jalr x1, 1F0(gp)
```

#### Test `gets` stop at newline: (80002670-800026E4)  (v0)
#### Test `gets` stop at max-len: (800026F0-80002764)  (v1)
```assembly
// gp+0xFD0 is out buffer, and gp+0xFC0 is out expected result. FCC(gp) is fixed BBBB bytes.
lw a0, FCC(gp)
sw a0, FD0(gp)
sw a0, FD4(gp)
sw a0, FD8(gp)
sw a0, FDC(gp)

jalr x1, 1B0(gp)
addi a0, gp, 0xFD0
addi a1, zero, 7
jalr x1, 1C0(gp)
jalr x1, B0(gp)
jalr x1, 1C0(gp)
lui a0, 0x00000
addi a0, a0, 0xFF0  // all
jalr x1, 1F0(gp)

addi a2, gp, 0x8DC
(v0) addi a3, gp, FC0
(v1) addi a3, gp, FB0
addi a4, gp, FD0

lw a0, 0(a4)
lw a1, 0(a3)
jalr x1, 1A0(gp)
lw a0, 4(a4)
lw a1, 4(a3)
jalr x1, 1A0(gp)
lw a0, 8(a4)
lw a1, 8(a3)
jalr x1, 1A0(gp)
lw a0, C(a4)
lw a1, C(a3)
jalr x1, 1A0(gp)
```


#### Test `brk`: (80002780-800027F0)

Parametrized (8000E800-8000E880):
- `v0` - fail 1-out-of-bounds
- `v1` - fail 1-below-bounds
- `v2` - fail above-bounds
- `v3` - fail below-bounds
- `v4` - success middle address
- `v5` - success same address
- `v6` - success max address
- `v7` - success min address

Parametrization struct:
```C
u32 test_name_str;
u32 argument;
u32 expected_result;
u32 expected_new_heap_top;
```

Test:
```assembly
// Note that this test suite starts with top==start.
lw a0, FA8(gp)
sw a0, FA4(gp)

lui a0, 0x8000F
addi a0, a0, 0x800
sw a0, 0(sp)      // current param pointer
addi a0, a0, 0x80
sw a0, 4(sp)      // params end

// TEST_CASE_START:
jalr x1, 1B0(gp)
lw a0, 0(sp)
lw a0, 4(a0)
jalr x1, 1C0(gp)
jalr x1, 200(gp)
jalr x1, 1C0(gp)

lw s0, 0xE0(sp)
lw a1, 8(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

lw a0, FA4(gp)
lw a1, C(s0)
jalr x1, 1A0(gp)

addi a0, zero, 0xBF0  // all but a0
jalr x1, 1F0(gp)

addi s0, s0, 0x10
sw s0, 0(sp)
lw a0, 4(sp)
bltu s0, a0, TEST_CASE_START (-0x48)

lw a0, FA8(gp)
sw a0, FA4(gp)
```


#### Test `sbrk`: (80002800-80002870)

Parametrized (8000E8A0-8000E960):
- `v0` - fail 1-below-bounds on initial address
- `v1` - fail 1-out-of-bounds on initial address
- `v2` - fail below-bounds on initial address
- `v3` - fail above-bounds on initial address

- `v4` - success incr=0 on initial address
- `v5` - success incr=0x10 on initial address
- `v6` - success incr=0x10 on middle address
- `v7` - success incr=0 on middle address (after +)
- `v8` - success incr=-0x8 on middle address
- `v9` - success incr=0 on middle address (after -)

- `vA` - fail below-bounds on middle address
- `vB` - fail after-bounds on middle address

Parametrization struct:
```C
u32 test_name_str;
u32 argument;
u32 expected_result;
u32 expected_new_heap_top;
```

Test:
```assembly
// Note that these tests rely on their order.
lw a0, FA8(gp)
sw a0, FA4(gp)

lui a0, 0x8000F
addi a0, a0, 0x8A0
sw a0, 0(sp)      // current param pointer
addi a0, a0, 0xC0
sw a0, 4(sp)      // params end

// TEST_CASE_START:
jalr x1, 1B0(gp)
lw a0, 0(sp)
lw a0, 4(a0)
jalr x1, 1C0(gp)
jalr x1, 210(gp)
jalr x1, 1C0(gp)

lw s0, 0xE0(sp)
lw a1, 8(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

lw a0, FA4(gp)
lw a1, C(s0)
jalr x1, 1A0(gp)

addi a0, zero, 0xBF0  // all but a0
jalr x1, 1F0(gp)

addi s0, s0, 0x10
sw s0, 0(sp)
lw a0, 4(sp)
bltu s0, a0, TEST_CASE_START (-0x48)

lw a0, FA8(gp)
sw a0, FA4(gp)
```


#### Test `strncmp`: (80002880-800028E0)

Parametrized (8000E980-8000EAC0):
- `v0` - same strings, n < length
- `v1` - same strings, n == length
- `v2` - same strings, n > length

- `v3` - same length, string lexi-smaller, big n.
- `v4` - same length, string lexi-bigger, big n.
- `v5` - same length, string lexi-bigger, n is the first different index.
- `v6` - same length, string lexi-bigger, n is the 1 before the first different index.

- `v7` - different string from the first char, n==1.
- `v8` - different string from the first char, n==0.
- `v9` - the second is a prefix of the first, big n.

Parametrization struct:
```C
u32 test_name_str;
u32 length_argument;
u32 expected_result;
char[8] string1;
char[8] string2;
```

Test:
```assembly
lui a0, 0x8000F
addi a0, a0, 0x980
sw a0, 0(sp)      // current param pointer
addi a0, a0, 0x140
sw a0, 4(sp)      // params end

// TEST_CASE_START:
jalr x1, 1B0(gp)
lw a0, 0(sp)
lw a2, 4(a0)
addi a1, a0, 0x14
nop
addi a0, a0, 0xC
jalr x1, 1C0(gp)
jalr x1, 220(gp)
jalr x1, 1C0(gp)

lw s0, 0xE0(sp)
lw a1, 8(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

addi a0, zero, 0xBF0  // all but a0
jalr x1, 1F0(gp)

addi s0, s0, 0x20
sw s0, 0(sp)
lw a0, 4(sp)
bltu s0, a0, TEST_CASE_START (-0x48)
```


#### Test `DICT_initialize`: (800028F0-80002948)

Uses test memory 8000EAE0-8000EF00 (0x400 bytes buffer with "guards").

```assembly
jalr x1, 1B0(gp)
lui a0, 0x8000F
addi a0, a0, 0xAF0
jalr x1, 1C0(gp)
jalr x1, 230(gp)
jalr x1, 1C0(gp)

addi a0, zero, 0xFF0  // all
jalr x1, 1F0(gp)

lui a3, 0x8000F
addi a3, a3, 0xAF0
addi a4, a3, 0x400
addi a1, zero, FFF
addi a2, zero, 0

lw a0, FFC(a3)
jalr x1, 1A0(gp)
lw a0, 0(a4)
jalr x1, 1A0(gp)

addi a1, zero, 0

lw a0, 0(a3)
jalr x1, 1A0(gp)
addi a3, a3, 4
bltu a3, a4, 0xFF4
```


#### Test `DICT_calculate_key`: (80002950-800029B0)

Parametrized (8000EF00-8000EF60):
- `v0` - "A",      n=0, expected=0x00
- `v1` - "\x50",   n=1, expected=0x00
- `v2` - "AB",     n=1, expected=0xD0
- `v3` - "AB",     n=2, expected=0x5A
- `v4` - "BA",     n=2, expected=0xA4
- `v5` - "STRING", n=6, expected=0xB6

Parametrization struct:
```C
u32 test_name_str;
u8  length;
u8  expected_result;
char[10] string;
```

Test:
```assembly
lui a0, 0x8000F
addi a0, a0, 0xF00
sw a0, 0(sp)      // current param pointer
addi a0, a0, 0x60
sw a0, 4(sp)      // params end

// TEST_CASE_START:
jalr x1, 1B0(gp)
lw a0, 0(sp)
lbu a1, 4(a0)
nop
nop
addi a0, a0, 0x6
jalr x1, 1C0(gp)
jalr x1, 240(gp)
jalr x1, 1C0(gp)

lw s0, 0xE0(sp)
lbu a1, 5(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

addi a0, zero, 0xBF0  // all but a0
jalr x1, 1F0(gp)

addi s0, s0, 0x10
sw s0, 0(sp)
lw a0, 4(sp)
bltu s0, a0, TEST_CASE_START (-0x48)
```

#### Test `DICT_get_by_bin`: (800029C0-80002A28)

Parametrized (8000EF80-8000F020) (Also uses upto 8000F140 for the linked lists):
- All the tests search the "string", n=6 in the given linked-list.
- `v0` - Empty list.
- `v1` - List with 1 correct node.
- `v2` - List with 1 bad node (both str and n).
- `v3` - List with 1 node, same n not same str.
- `v4` - List with 1 node, same str but smaller n.
- `v5` - List with 1 node, same str but bigger n.
- `v6` - List with 3 nodes, none are correct.
- `v7` - List with 4 nodes, first is correct.
- `v8` - List with 6 nodes, middle (third) is correct.
- `v9` - List with 4 nodes, last is correct.

Parametrization struct:
```C
u32 test_name_str;
u32 bin;
u32 expected_result;
u32 padding;
```

Test:
```assembly
lui a0, 0x8000F
addi a0, a0, 0xF80
sw a0, 0(sp)      // current param pointer
addi a0, a0, 0xA0
sw a0, 4(sp)      // params end
addi a0, a0, 0x20
sw a0, 8(sp)

// TEST_CASE_START:
jalr x1, 1B0(gp)
lw a0, 0(sp)
lw a1, 8(sp)
addi a2, zero, 0x6
nop
addi a0, a0, 0x4
jalr x1, 1C0(gp)
jalr x1, 250(gp)
jalr x1, 1C0(gp)

lw s0, 0xE0(sp)
lw a1, 8(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

addi a0, zero, 0xBF0  // all but a0
jalr x1, 1F0(gp)

addi s0, s0, 0x10
sw s0, 0(sp)
lw a0, 4(sp)
bltu s0, a0, TEST_CASE_START (-0x48)
```


#### Test `DICT_get`: (80002A30-80002AAC)

Parametrized (8000F180-8000F220) (Also uses upto 8000F700 for the linked lists and the dict):
- `v0` - nothing in the bin.
- `v1` - List of 1 in the bin, but other word.
- `v2` - List of 1 in the bin, exactly the word.
- `v3` - List of 2 in the bin, Not the word.
- `v4` - List of 3 in the bin, The word is last.

Parametrization struct:
```C
u32 test_name_str;
u32 str_len;
u32 expected_a0; // node
u32 expected_a1; // bin
char[16] str;
```

Test:
```assembly
lui a0, 0x8000F
addi a0, a0, 0x180
sw a0, 0(sp)      // current param pointer
addi a0, a0, 0xA0
sw a0, 4(sp)      // params end
addi a0, a0, 0xE0
sw a0, 8(sp)

// TEST_CASE_START:
jalr x1, 1B0(gp)
lw a0, 0(sp)
addi a1, a0, 0x10
lw a2, 4(a0)
lw a0, 8(sp)
nop
jalr x1, 1C0(gp)
jalr x1, 260(gp)
jalr x1, 1C0(gp)

lw s0, 0xE0(sp)
lw a1, 8(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

lw a0, 0x1C(sp)
lw a1, 0xC(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

lui a0, 0xFFFFF
addi a0, a0, 0x3F0  // all but a0,a1
jalr x1, 1F0(gp)

addi s0, s0, 0x20
sw s0, 0(sp)
lw a0, 4(sp)
bltu s0, a0, TEST_CASE_START (-0x5C)
```


#### Test `DICT_insert`: (80002AC0-80002???)

Parametrized (8000F700-8000F7F0) (Also uses upto 8000FC80 for the dict):
- `v0` - Success add "STR",len=3,val=1
- `v1` - Failure (already exist) add "STR",len=3,val=1
- `v2` - Failure (already exist) add "STR",len=3,val=2
- `v3` - Success add "AB",len=2,val=3 (its bin was empty before)
- `v4` - Success add "STR]",len=4,val=4 (its bin wasn't empty before)
- `v5` - Failure ("STRI" isn't in dict, but no memory for sbrk)

Parametrization struct:
```C
u32 test_name_str;
u32 str_len;
u32 value;
u32 sbrk_by;

char[8] str;
u32 expected_result;
u32 expected_next;

u32 expected_top;
u32 get_s1_res;
u32 get_s2_res;
u32 get_s3_res;
```

Test:
```assembly
lui a0, 0x8000F
addi a0, a0, 0x180    ##TODO 0x700
sw a0, 0(sp)      // current param pointer
addi a0, a0, 0xA0     ##TODO 0x120
sw a0, 4(sp)      // params end
addi a0, a0, 0xE0     ##TODO 0x60
sw a0, 8(sp)

##TODO insert 2:
lw a0, FA8(gp)
sw a0, FA4(gp)

// TEST_CASE_START:
jalr x1, 1B0(gp)
lw a0, 0(sp)
addi a1, a0, 0x10     ##TODO override these 4 with:   addi a1, a0, 0x10 ; lw a2, 4(a0) ; lw a3, 8(a0) ; lw a0, 8(s0)
lw a2, 4(a0)
lw a0, 8(sp)
nop
jalr x1, 1C0(gp)
jalr x1, 260(gp)      ##TODO 270(gp)
jalr x1, 1C0(gp)

lw s0, 0xE0(sp)
lw a1, 8(s0)          ##TODO 0x18(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

##TODO INSERT NEXT BUNCH (7 times 0x1A0):
// Verify all new-node fields are correct.
addi t0, a0, 0
lw a0, 0(t0)
lw a1, 0x1C(s0)
jalr x1, 1A0(gp)

lw a0, 4(t0)
lw a1, 8(s0)
jalr x1, 1A0(gp)

lw a0, 8(t0)
lw a1, 4(s0)
jalr x1, 1A0(gp)

lw a0, C(t0)
addi a1, s0, 0x10
jalr x1, 1A0(gp)

// Verify if the 3 strings expected to exist.
lw a0, E8(sp)
addi a1, gp, 0x8F8
addi a2, zero, 3
jalr x1, 260(gp)
lw a1, 0x24(t0)
addi a2, s0, 0
jalr x1, 1A0(gp)

lw a0, E8(sp)
addi a1, gp, 0x8FC
addi a2, zero, 2
jalr x1, 260(gp)
lw a1, 0x28(t0)
addi a2, s0, 0
jalr x1, 1A0(gp)

lw a0, E8(sp)
addi a1, gp, 0x8F8
addi a2, zero, 4
jalr x1, 260(gp)
lw a1, 0x2C(t0)
addi a2, s0, 0
jalr x1, 1A0(gp)



lw a0, 0x1C(sp)         ##TODO remove this 4
lw a1, 0xC(s0)
addi a2, s0, 0
jalr x1, 1A0(gp)

lui a0, 0xFFFFF         ##TODO replace 2 ops with just 1: addi BF0
addi a0, a0, 0x3F0  // all but a0,a1
jalr x1, 1F0(gp)

addi s0, s0, 0x20       ##TODO 0x30
sw s0, 0(sp)
lw a0, 4(sp)
bltu s0, a0, TEST_CASE_START (-D0)    ##TODO Update op new offset

##TODO insert 2:
lw a0, FA8(gp)
sw a0, FA4(gp)
```


### Finishing code for the sys tests: (80002AC0-80002AC8)
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
