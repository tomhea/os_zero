# os_zero

Developing RiscV OS with a hex editor
That means that I dont have a compiler (thus no C or higher support), no assembler (so can't write in assembly), I litterly have to build it all myself. I only have 2 tools:
- hex editor, for creating the os binary, and writing this documentation.
- `qemu` for running the os. Line used: `qemu-system-riscv32 -M virt -m 2048 -serial mon:stdio -smp 4 -bios hellos`.
    - I plan to use the next qemu devices: `-device VGA -drive file=mydisk.img,format=raw,if=none,id=drive0 -device virtio-blk,drive=drive0 -device e1000 -device virtio-keyboard -device virtio-mouse -device virtio-sound` (append it to the qemu line).

The only things that separates me from the first days of computers, is that:
- I have today's knowledge (I can search the web, but not copy anything).
- RiscV is a newer architecture, and I use qemu so I don't touch real hardware.

Other than that - I want to replicate the feeling of a first day computer scientist. I won't even enjoy the benifits of git, and just use this repo as a way to share my progress.


# My goal

Create a stable and fast os, with graphics, network driver, build a basic browser, and be able to open youtube and watch a video (you can suggest which in the repo discussions).

## How to get there

My main idea is to write an assembler in opcode-bytes, run it, then assemble a compiler (written in assembly text) and run it, and then write the big os in that high level compiled language. Then program a file system, files, shell, simple cli-ide, graphics, real ide, and then write all the programs and drivers needed to achive the goal.




# My Journey

Here I'll write my OS versions, from first to current.


## Part 0 - hellos

`hellos` is Hello World OS. You'll see that I'll give a name to every OS that I'm going to develop, until reaching `os_zero` - The complete one with the youtube.

I managed to write a single character to UART (output), 'T', using a "putc" assembly function. Calling twice makes a "TT" print. That's awesome. I'll call this os `ttos`.

I added a `partial_os` folder to store working but not complete os-versions, which are a critical part of my way of getting to the actual next os-version).
That's my way of saving versions, as I can't use a git-versioning until I develop one. You can look at each file in this folder as a backup disk, used to store only the current os on it.

I decided to go 32bit as I think that'll be enough for our end-goal.

I decided about a rough "memory regions" partition (all hexadecimal numbers. `X:Y` means addresses from `X` up to `Y`, not including `Y`):
- Stack: `FFFFFFF8` and downwords. The Ram is `80000000:100000000`
- Fast syscalls: `80001000:80001800` (note that `gp=80001000`), so that's `gp:gp+800`. `putc` is at `gp+0`. I'll use the shortcut sysX for `gp+X`, so `putc` is `sys0`.
- Global data: `80000800:80001000`, so `gp-800:gp`. I'll use the shortcut `g_X` for `gp-1000+X`.
- Boot code: At `80000000`, jumps to the first code.
- First code: At `80002000`.

I decided I'm gonna allow myself to use the "Text" window on my hex editor, and not the 16-byte-wide window, for writing this README. It's still not a markdown editor, 
and I justified this Text window for being ok because I basically do the same writing on my notebook. The only benefit here is that it's backed up, and That's fine by me.

Currently my first code is still in the Boot code region. My goal for `hellos` is a First code at its region, that reads a string "Hello, World!\n\0" (from Global data maybe) and prints it (maybe as another `sys`).

I wrote a small Boot code, that sets `gp, sp`, saves `a0, a1, a2` as globals, and calls the First code. Has a stub that prints "\nFIN\n" if the First code returns. I called this os (with First code that just returns) - `finos`.

I added two syscalls, `puts` (prints a string), and `exit`, and called them both from the First code to print "Hello, World!\n" (stored as a global). And it worked!

`hellos` is done!

#### Globals / Syscalls added:
- `putc(c: char_a0) -> None` - `sys0` (uses `t0,t1`, keeps `a0`).
- `puts(s: pointer_a0) -> None` - `sys20` (keep all regs).
- `exit() -> NO_RETURN` - `sys70` (infinite loop).
- `boot_a0` - `g_FF4`.
- `boot_a1` - `g_FF8`.
- `boot_a2` - `g_FFC`.
- Rule: each `sys` is aligned to `0x10`, and each `global` is aligned to `0x4`.

Unused memory will always be marked as `'A'` bytes.

#### Fallbacks/Bugs:
- Wrote the first os in big endien.
- I initialy failed to encode 5 out of 10 opcodes. Encoding by hand is HARD.
- I got wrong the loading address of the bios binary. (thought 0x1000, was 0x80000000). I tried writing an elf format in the hex editor, and it took me trial and error + debugging to understand what is needed - a non-elf binary file (first opcode in offset 0), that's loaded into address 0x80000000. At first I was afraid to use debugger, as it felt like cheating, but it was to understand qemu's ABI.



## Part 1 - multi_hellos

I decided to add the `-smp 4` (4 cores) support now, so the Boot code need to hande the other cores too.
Currently adding this flag prints the "Hello, World!\n" string four times, which is great. I want to change it so that only the first core will run, and the rest will "sleep".

I improved the current Boot code: It now sets different stack for each process, based on `a0` (initialized at boot with the hardware thread id). If it's the 0 thread then jump to the First code.
If it's another thread id, then jump to a `wfi` which afterwards jumps to the address found in `multicore_boot_address` (initialized to the `wfi` address).
It allows us to control when to "release" the other threads, and what code they should execute.

`multi_hellos` is done!

#### Globals / Syscalls added:
- `multicore_boot_address - `g_FF0`.



## Part 2 - echos

This version will be able to get input. echos will get string as input and will echo it.

I decided that I need to be more organized, and that I'll allow myself writing multiple "notebooks" (yeah, I'm addressing these ".md" files as "Notebooks").
I created the [bare_asm.md](bare_asm.md) for the actual assembly I coded in memory, and the memory regions.

I implemented `getc` that inputs a byte, and `availc` that returns wether an input is available to be read.

I wrote a small program that inputs a char, increments it and prints it (`charos`).

I added a First code that prints "Enter your name: ", calls gets(buf, 16), and then prints "Hello {buf}!\n".

While encoding the `gets` function, I saw that I misencoded the `addi sp, sp, 0x14` at the end of `putc`. Fixed.

I created a working echos! It prints "Hello {buf}!", but it doesn't print any of the characters live (like, it doesn't show anything until you press "Enter". I call it `ghost_echos`.

I added a print to every char (and replaced \r with a \n print). 

`echos` is done!

#### Globals / Syscalls added:
- `getc() -> char_a0` - `sys80` (uses `t0,t1`).
- `availc() -> bool_a0` - `sysA0` (keep all regs) - **NOT CHECKED**.
- `gets(out_s: pointer_a0, max_len: a1) -> None` - `sysB0` (keep all regs).

#### Fallbacks/Bugs:
- I implemented `gets` to stop only of '\n', but my keybord actually only generated '\r', so I needed to add another check.



## Part 3 - hexos

I want to be able to print registers, numbers. 
This os version version will be dedicated to print 4-byte hexadecimal values - thus `hexos`.

Of course, I managed to find a bug in `gets` from the previous version - `a1` was stored on the `a0` stack spot.

I implemented the hex print, and it works perfectly.

`hexos` is done!

#### Globals / Syscalls added:
- `putx(hex: a0) -> None` - `sys130` (keep all regs).



## Part 4 - testos

I think that before going further, I must have some testing support, and "testing routine". Thus - `testos`.

In this operation system version I'm going to implement multiple testing functions (That will print errors on test failures),
and test every existing `sysX` function.

Adding testing capabilities this early, will hopefully lead me to keep my entire operation system tested at every phase of the process. That's exciting! So I have to do a good job in this part.

My 0x800 space after `$gp` is running out, so I'm gonna start using it as a "jumpt to" table, in the next way:
sysXX0 will now only take 12 bytes (+4 more reserved): `sw x1, FFC(x2)`, `lui x1 0x?????`, `jalr x0, 0x???(x1)`. The jumped address will be responsible for doing the `addi sp, sp, FFC` part.
Note that the current 7 sys funcs already used `0x1A0` bytes. 
Making this change now will let me use the next `0x660` bytes as 102 more functions. That will probably be enough for the assembler functionality, and if not, we could find a future solution for that.

Ok, so let's create the first function - `assert_ret`.
I need to find a place for the function's body. I'll designate the next section for this part: 0x80004000-0x8000FFFF.

I implemented the `assert_ret`!
I also a `tests_failure` global, that initialized with 1 but set to 0 on any test failed (by the testing functions). It gets printed at the end of the First code.

It took time to encode this function. Hopefully the next will be easier.

I proceeded to implement both `put_regs_test_values()` and `store_all_regs()`, and tested them together.

First code is now:
- 80002000-80002010: Check `assert_ret`.
- 80002010-8000203C: Check `put_regs_test_values, store_all_regs`.
- 8000203C-80002058: Print the `tests_success` boolean, and finish.

I'll have to have some kind of documentation + order of the "First code" section, as now the data there might get lost. 
I want this `testos` to also define exactly how the "First code" shouls look like.

My first idea is to have it "First code" to lie in addresses 80002000-80004000, and it will be responsible for setting up the tests and running them, and afterwords to jump to the assembler code in 0x80010000.
Each test in the first code will be documented in a separate document "binary_tests.md".

I decided to have all of my tests automatic - means that I won't have to manually change the testing-opcode-bytes each time to make sure all cases work.
It actually only matters for testing the testing functions. I decided of having two "atomic" testing functions, which are as simple as they can be:
- `assert_test_success` - For making sure the global `tests_success` in on. Prints if not, and anyway sets it on again.
- `assert_test_failure` - For making sure the global `tests_success` in off. Prints if not, and anyway sets it on again.

I really see myself digging too deep here, as why didn't I called the previous testing functions "atomic"? Well, here I took extra measures. Not sure if it was really needed, but it's better checked in that way.

My goal now is to write a documented First code that tests all 4 testing functions (of course, to implement the 4th too). This os version will be called `tested_test_funcsos`.

I completed testing the `assert_ret`, and the `put+store` combination, for both sanity and failure, with the new atomic functions!

I completed implementing the `validate_all_regs_unchanged`, and testing it, with 8 different tests!
I also decided and modified `put_regs_test_values` to store `s1=1` for my ease of not setting it again every time I call sys1D0 or sys1E0.

Now all of my testing functions are implemented, documented and tested. 
The other sysXXX functions aren't tested, but I still stop for a second to celebrate, and call the current os: The `tested_test_funcsos`.

I continue with jumping to 0x80002400, the start of the sys tests. I decided that when this part ends, it will jump to 0x80010000, the start of the actual code to run (the assembler).

I implemented the output tests, and the input tests!

All of my binary functions are now fully tested (except the three ATOMIC functions).
Also, Each piece of binary-encoded assembly is documented in the `bare_asm.md` or `binary_tests.md` notebook.

This part was the longest to implement yet. I'm so glad to declare:

`testos` is done!

#### Globals / Syscalls added:
- `assert_test_success() -> None` - `sys1D0` JUMPER (modifies a0, relies on `s1==1`).
- `assert_test_failure() -> None` - `sys1E0` JUMPER (modifies a0, relies on `s1==1`).
- `assert_ret(ret_val: a0, expected: a1, string_testname: a2) -> None` - `sys1A0` JUMPER (keep all regs).
- `put_regs_test_values() -> None` - `sys1B0` JUMPER (modifies all regs).
- `store_all_regs() -> None` - `sys1C0` JUMPER (keep all regs except sp).
- `validate_all_regs_unchanged(regs_mask: a0) -> None` - `sys1F0` JUMPER (modifies the a-regs, and sp).
- `testing_mask` - `g_FEC`
- `tests_success` - `g_FE8`
- `test_print_on_failure` - `g_FE4`
- `testing_tests_success` - `g_FE0`.
- `gets_tests_buffers` - `g_FB0` (0x30 bytes).



## Part 5 - allocos

`allocos` is dynamic **alloc**ation **os**.

In this os I'm gonna implement a very simple dynamic allocation functionality: `brk`, `sbrk`. Of course, they'll be tested.
The allocation will have a specific memory chunk to allocate from it: 0x81000000 -> 0x81FFFFFF.
This dynamic heap will be called binary_heap, as it's in the binary section (as I'm still coding in binary).

To test these functions I'll use some test data. 
I see that if I'll keep using the globals spot for tests data, I might finish it very fast.
So I need another spot that's reserved just for that:
- I'll reserve the 8000E800-8000F800 page just for big test-data (buffers, parametrized arrays, etc.).
- I'll reserve 0x400 stack frame ready to use by any test, without having to mess with modifying sp.

I coded `brk` and manually tested the function.
I designed my first "parametrized" test, the `brk` test (described in the `binary_tests.md` notebook).
After a few pushbacks, it works as a charm! I'll call this half-version, of `brk` implemented and tested - `brkos`.

After that, implementing `sbrk` was easy, and for its parametrized tests I basically copy pasted the "brk parametrized test" and just modified the parameters. That's awesome.

`allocos` is done!

#### Globals / Syscalls added:
- `brk() -> None` - `sys200` JUMPER.
- `sbrk() -> None` - `sys210` JUMPER.
- `binary_heap_segment_end` - `g_FAC`.
- `binary_heap_segment_start` - `g_FA8`.
- `binary_heap_top` - `g_FA4` - Initialized with "segment_start", and represent the first unallocated byte in the binary_heap.

#### Fallbacks/Bugs:
- I `btl` instead of `bltu`.
- Gave `assert_ret` `a2-param` the first-word of a string, instead of the pointer to the string.



## Part 6 - dictos (**CURRENT**)

In this operation system I'm going to implement a dictionary data structure (256-bins hashmap) using hand-coded assembly, Thus - `dictos`.
It will be very helpful for writing an assembler in the future version, `asos`.
The dictionary will only have "Init, Get, Insert", and no delete. 
For it's dynamic nature, it could use the `sbrk()` as node allocations for the bins. When we finish with the dict, we can "free" the memory with `brk(before_dict)`, assuming no other allocations.

#### Globals / Syscalls added:
- `strncmp(str1: a0, str2: a1, n: a2) -> a0` - `sys220` JUMPER - **NOT IMPLEMENTED, NOT TESTED**.
- `DICT_initialize(buffer: a0) -> None` - `sys230` JUMPER - **NOT IMPLEMENTED, NOT TESTED**.
- `DICT_calculate_key(str: a0, n: a1) -> a0` - `sys240` JUMPER - Calculate the key - **NOT IMPLEMENTED, NOT TESTED**.
- `DICT_get_by_bin(bin: a0, str: a1, n: a2) -> a0` - `sys250` JUMPER - Returns pointer to the found node or NULL if not found - **NOT IMPLEMENTED, NOT TESTED**.
- `DICT_get(dict: a0, str: a1, n: a2) -> a0, a1` - `sys260` JUMPER - a0 same as `DICT_get_by_bin`, a1 the bin address - **NOT IMPLEMENTED, NOT TESTED**.
- `DICT_insert(dict: a0, str: a1, n: a2, value: a3) -> None` - `sys270` JUMPER - **NOT IMPLEMENTED, NOT TESTED**.




## Part 7 - asos (**NEXT**)

`asos` is **as**sembly **os**.

This version will be an actual assembler, written byte by byte, to assemble a piece of memory as text, into assembly binary, and execute it.
It will allow me to continue much more easily after it's done, and very tested.
