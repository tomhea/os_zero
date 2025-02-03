# Binary Tests

In this notebook I'll document every test that gets done in the critical Binary phase - The phase which I'll encode every opcode to 0/1 by hand.

Tests here are crucial, so I'll be having them as fast as I can.

There are two kinds of tests:
- Manual tests (tests which I control the input manualy, but I don't run them on every input each time.
   - For example, to test that my 4 testing functions are working properly, I'll have manual tests for them, so that I could test that they report on failures, but I wouldn't want to see "Test failures" on expected results.
- Automatic tests:
   - Any other tests are preferred to be automatic, and all cases of them will run every time.

Notes:
- The address ranges are "where arre the text in memory", in hex, and they here won't include the last byte.
- The binary tests have a "First code" section reserved for them, meaning addresses 80002000-80004000.
- There is a code that prints wether all test passed, and it's located in `80002040`.

# Maual Tests

#### Test `assert_ret` (80002000-80002010)

#### Test `store_all_regs` and `put_regs_test_values` (80002010-80002040)



# Automatic Tests
