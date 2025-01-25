# os_zero
Developing RiscV OS with a hex editor
That means that I dont have a compiler (thus no C or higher support), no assembler (so can't write in assembly), I litterly have to build it all myself. I only have 2 tools:
- hex editor, for creating the os binary, and writing this documentation.
- `qemu` for running the os. Line used: `qemu-system-riscv32 -M v
irt -m 2048 -serial mon:stdio -bios hellos`.

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
I managed to write a single character to UART (output), 'T', using a "putc" assembly function. Calling twice makes a "TT" print. That's awesome.
I decided to go 32bit as I think that'll be enough for our end-goal.

#### Fallbacks/Bugs:
- Wrote the first os in big endien.
 - I initialy failed to encode 5 out of 10 opcodes. Encoding by hand is HARD.
 - I got wrong the loading address of the bios binary. (thought 0x1000, was 0x80000000). I tried writing an elf format in the hex editor, and it took me trial and error + debugging to understand what is needed - a non-elf binary file (first opcode in offset 0), that's loaded into address 0x80000000. At first I was afraid to use debugger, as it felt like cheating, but it was to understand qemu's ABI.