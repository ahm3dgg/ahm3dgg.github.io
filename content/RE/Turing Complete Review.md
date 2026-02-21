![[Pasted image 20260221211146.png]]

Turing Complete is a Game and a Circuit Simulator, where your goal is to build a complete computer from basic primitives (i.e. NAND gates), in fact you will be building two computers and the game and also there are programming levels, where you actually run programs with your custom instruction set on these computers.

The Game teaches you

![[Pasted image 20260221211353.png]]

You start from the ground up, each level builds upon the previous ones, every component you build will be added to your tool built, that you then use to solve next challenges.

You start by building basic logic gates, and then using them to solve some logic circuits puzzles, in order to have a solid understanding of not of just how they work, but also of what's their use, you will start picking up a lot of patterns, as you progress, and there are also solutions if you got stuck so don't worry.

You will also build the basic components of a computer individually first, then combine them, for example you will build the ALU and an instruction decoder, RAM module, that you will then assemble together when building the computer.

The first computer you build is very basic, so it won't have I/O devices, it doesn't even have a stack, the memory is used only for instructions, and you will be working mainly with registers, so you will really start appreciating stacks and the ability to call functions, because you will be forced to program without using a stack :D, instructions are also strict in that computer, for example to add two numbers you have to load them in registers `r1` and `r2`, and the result will be in `r3`, so its unpleasant when used as a programming platform.

The next computer is where you starting adding I/O devices, stack, more instructions, you can also make your own instruction aliases which are basically instructions valid in the assembler but translates to instructions you have already built, you will also be introduced to CPU optimizations that are in fact still of use today, these include superscalar which in the game was just having two ALUs instead of one, pipeline, also speculative execution or branch prediction, where the CPU assumes the result of a jump instruction before its executed, this is possible because we divide the execution into three stages (fetch-decode-execute-writeback), and though we only change the CPU state in the writeback stage, so if for instance the CPU got it wrong (we shouldn't have branched), then the CPU will cause a pipeline flush.

So far the game was really exciting and I had a lot of fun playing it, and I think this game actually gives you more information and perhaps more fun than Nand2Tetris.

The game teaches you a systematic approach to building a computer, I wouldn't call it a real computer, and I wouldn't really say that it explains how modern computers or processors work, but it does give you some foundation, because modern computers are crazy honestly with tons of standards, CPU optimizations, Legacy stuff still exists but is getting replaced by modern versions, so you still kinda want to look at the legacy stuff. I wouldn't say that I have made a full mental model of how a modern computer works yet, though I am on my way :D 