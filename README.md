# os_zero

Developing RiscV OS with a hex editor
That means that I dont have a compiler (thus no C or higher support), no assembler (so can't write in assembly), I litterly have to build it all myself. I only have 2 tools:
- hex editor, for creating the os binary, and writing this documentation.
- `qemu` for running the os. Line used: `qemu-system-riscv32 -M virt -m 2048 -serial mon:stdio -bios hellos`.

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



#### Globals / Syscalls added:
- `putc(c: char_a0) -> None` - `sys0` (uses `t0,t1`).
- `puts(s: pointer_a0) -> None` - `sys20`
- `boot_a0` - `g_FF4`.
- `boot_a1` - `g_FF8`.
- `boot_a2` - `g_FFC`.
- Rule: each `sys` is aligned to `0x10`, and each `global` is aligned to `0x4`.

Unused memory will always be marked as `'A'` bytes.

#### Fallbacks/Bugs:
- Wrote the first os in big endien.
 - I initialy failed to encode 5 out of 10 opcodes. Encoding by hand is HARD.
 - I got wrong the loading address of the bios binary. (thought 0x1000, was 0x80000000). I tried writing an elf format in the hex editor, and it took me trial and error + debugging to understand what is needed - a non-elf binary file (first opcode in offset 0), that's loaded into address 0x80000000. At first I was afraid to use debugger, as it felt like cheating, but it was to understand qemu's ABI.
