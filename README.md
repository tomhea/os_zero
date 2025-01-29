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

I decided that I need to be more organized, and that I'll allow myself writing multiple "notebooks" (yeah, I'm addressing markdowns as "Notebooks").
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



## Part 4 - dictos (**CURRENT**)

In this operation system I'm going to implement a dictionary data structure (256-bins hashmap) using hand-coded assembly, Thus - `dictos`.
It will be very helpful for writing an assembler in the future version, `asos`.
The dictionary will only have "Init, Get, Insert", and no delete. 

For this dynamic data structure I'm going to intrucduce a "heap" - for allocating 16 bytes of data.



## Part 5 - asos (**NEXT**)

`asos` is **as**sembly **os**.

This version will be an actual assembler, written byte by byte, to assemble a piece of memory as text, into assembly binary, and execute it.
It will allow me to continue much more easily after it's done, and very tested.
