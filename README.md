# nanopond
Nanopond: minimal artificial life system in C

# Text from the header of nanopond.c
```
/* *********************************************************************** */  
/*                                                                         */  
/* Nanopond version 2.0 -- A teeny tiny artificial life virtual machine    */  
/* Copyright (C) Adam Ierymenko                                            */  
/* MIT license -- see LICENSE.txt                                          */  
/*                                                                         */  
/* *********************************************************************** */    
```

## Changelog:

```
 * 1.0 - Initial release
 * 1.1 - Made empty cells get initialized with 0xffff... instead of zeros
 *       when the simulation starts. This makes things more consistent with
 *       the way the output buf is treated for self-replication, though
 *       the initial state rapidly becomes irrelevant as the simulation
 *       gets going.  Also made one or two very minor performance fixes.
 * 1.2 - Added statistics for execution frequency and metabolism, and made
 *       the visualization use 16bpp color.
 * 1.3 - Added a few other statistics.
 * 1.4 - Replaced SET with KILL and changed EAT to SHARE. The SHARE idea
 *       was contributed by Christoph Groth (http://www.falma.de/). KILL
 *       is a variation on the original EAT that is easier for cells to
 *       make use of.
 * 1.5 - Made some other instruction changes such as XCHG and added a
 *       penalty for failed KILL attempts. Also made access permissions
 *       stochastic.
 * 1.6 - Made cells all start facing in direction 0. This removes a bit
 *       of artificiality and requires cells to evolve the ability to
 *       turn in various directions in order to reproduce in anything but
 *       a straight line. It also makes pretty graphics.
 * 1.7 - Added more statistics, such as original lineage, and made the
 *       genome dump files CSV files as well.
 * 1.8 - Fixed LOOP/REP bug reported by user Sotek.  Thanks!  Also
 *       reduced the default mutation rate a bit.
 * 1.9 - Added a bunch of changes suggested by Christoph Groth: a better
 *       coloring algorithm, right click to switch coloring schemes (two
 *       are currently supported), and a few speed optimizations. Also
 *       changed visualization so that cells with generations less than 2
 *       are no longer shown.
 * 2.0 - Ported to SDL2 by Charles Huber, and added simple pthread based
 *       threading to make it take advantage of modern machines.
```

## Overview

Nanopond is just what it says: a very very small and simple artificial
life virtual machine.

It is a "small evolving program" based artificial life system of the same
general class as Tierra, Avida, and Archis.  It is written in very tight
and efficient C code to make it as fast as possible, and is so small that
it consists of only one .c file.

### How Nanopond works:

The Nanopond world is called a "pond."  It is an NxN two dimensional
array of Cell structures, and it wraps at the edges (it's toroidal).
Each Cell structure consists of a few attributes that are there for
statistics purposes, an energy level, and an array of POND_DEPTH
four-bit values.  (The four-bit values are actually stored in an array
of machine-size words.)  The array in each cell contains the genome
associated with that cell, and POND_DEPTH is therefore the maximum
allowable size for a cell genome.

The first four bit value in the genome is called the "logo." What that is
for will be explained later. The remaining four bit values each code for
one of 16 instructions. Instruction zero (0x0) is NOP (no operation) and
instruction 15 (0xf) is STOP (stop cell execution). Read the code to see
what the others are. The instructions are exceptionless and lack fragile
operands. This means that *any* arbitrary sequence of instructions will
always run and will always do *something*. This is called an evolvable
instruction set, because programs coded in an instruction set with these
basic characteristics can mutate. The instruction set is also
Turing-complete, which means that it can theoretically do anything any
computer can do. If you're curious, the instruciton set is based on this:

http://www.muppetlabs.com/~breadbox/bf/

At the center of Nanopond is a core loop. Each time this loop executes,
a clock counter is incremented and one or more things happen:

- Every REPORT_FREQUENCY clock ticks a line of comma seperated output is printed to STDOUT with some statistics about what's going on.
- Every INFLOW_FREQUENCY clock ticks a random x,y location is picked, energy is added (see INFLOW_RATE_MEAN and INFLOW_RATE_DEVIATION) and it's genome is filled with completely random bits. Statistics are also reset to generation==0 and parentID==0 and a new cell ID is assigned.
- Every tick a random x,y location is picked and the genome inside is executed until a STOP instruction is encountered or the cell's energy counter reaches zero. (Each instruction costs one unit energy.)
 
The cell virtual machine is an extremely simple register machine with
a single four bit register, one memory pointer, one spare memory pointer
that can be exchanged with the main one, and an output buffer. When
cell execution starts, this output buffer is filled with all binary 1's
(0xffff....). When cell execution is finished, if the first byte of
this buffer is *not* 0xff, then the VM says "hey, it must have some
data!". This data is a candidate offspring; to reproduce cells must
copy their genome data into the output buffer.

When the VM sees data in the output buffer, it looks at the cell
adjacent to the cell that just executed and checks whether or not
the cell has permission (see below) to modify it. If so, then the
contents of the output buffer replace the genome data in the
adjacent cell. Statistics are also updated: parentID is set to the
ID of the cell that generated the output and generation is set to
one plus the generation of the parent.

A cell is permitted to access a neighboring cell if:
   
- That cell's energy is zero
- That cell's parentID is zero
- That cell's logo (remember?) matches the trying cell's "guess"

Since randomly introduced cells have a parentID of zero, this allows
real living cells to always replace them or eat them.

The "guess" is merely the value of the register at the time that the
access attempt occurs.

Permissions determine whether or not an offspring can take the place
of the contents of a cell and also whether or not the cell is allowed
to EAT (an instruction) the energy in it's neighbor.

If you haven't realized it yet, this is why the final permission
 criteria is comparison against what is called a "guess." In conjunction
 with the ability to "eat" neighbors' energy, guess what this permits?

Since this is an evolving system, there have to be mutations. The
MUTATION_RATE sets their probability. Mutations are random variations
with a frequency defined by the mutation rate to the state of the
virtual machine while cell genomes are executing. Since cells have
to actually make copies of themselves to replicate, this means that
these copies can vary if mutations have occurred to the state of the
VM while copying was in progress.

What results from this simple set of rules is an evolutionary game of
"corewar." In the beginning, the process of randomly generating cells
will cause self-replicating viable cells to spontaneously emerge. This
is something I call "random genesis," and happens when some of the
random gak turns out to be a program able to copy itself. After this,
evolution by natural selection takes over. Since natural selection is
most certainly *not* random, things will start to get more and more
ordered and complex (in the functional sense). There are two commodities
that are scarce in the pond: space in the NxN grid and energy. Evolving
cells compete for access to both.

If you want more implementation details such as the actual instruction
set, read the source. It's well commented and is not that hard to
read. Most of it's complexity comes from the fact that four-bit values
are packed into machine size words by bit shifting. Once you get that,
the rest is pretty simple.

Nanopond, for it's simplicity, manifests some really interesting
evolutionary dynamics. While I haven't run the kind of multiple-
month-long experiment necessary to really see this (I might!), it
would appear that evolution in the pond doesn't get "stuck" on just
one or a few forms the way some other simulators are apt to do.
I think simplicity is partly reponsible for this along with what
biologists call embeddedness, which means that the cells are a part
of their own world.

Run it for a while... the results can be... interesting!

## Running Nanopond:

Nanopond can use SDL (Simple Directmedia Layer) for screen output. If
you don't have SDL, comment out USE_SDL below and you'll just see text
statistics and get genome data dumps. (Turning off SDL will also speed
things up slightly.)

After looking over the tunable parameters below, compile Nanopond and
run it. Here are some example compilation commands from Linux:

For Pentiums: 

`gcc -O6 -march=pentium -funroll-loops -fomit-frame-pointer -s -o nanopond nanopond.c -lSDL`

For Athlons with gcc 4.0+:

`gcc -O6 -msse -mmmx -march=athlon -mtune=athlon -ftree-vectorize -funroll-loops -fomit-frame-pointer -o nanopond nanopond.c -lSDL`

The second line is for gcc 4.0 or newer and makes use of GCC's new
tree vectorizing feature. This will speed things up a bit by
compiling a few of the loops into MMX/SSE instructions.

This should also work on other Posix-compliant OSes with relatively
new C compilers. (Really old C compilers will probably not work.)
On other platforms, you're on your own! On Windows, you will probably
need to find and download SDL if you want pretty graphics and you
will need a compiler. MinGW and Borland's BCC32 are both free. I
would actually expect those to work better than Microsoft's compilers,
since MS tends to ignore C/C++ standards. If `stdint.h` isn't around,
you can fudge it like this:

```
 #define uintptr_t unsigned long (or whatever your machine size word is)
 #define uint8_t unsigned char
 #define uint16_t unsigned short
 #define uint64_t unsigned long long (or whatever is your 64-bit int)
```
When Nanopond runs, comma-seperated stats (see doReport() for
the columns) are output to stdout and various messages are output
to stderr. For example, you might do:

`./nanopond >>stats.csv 2>messages.txt &`

To get both in seperate files.

Have fun!


### Tunable parameters

*Frequency of comprehensive reports*-- lower values will provide more
info while slowing down the simulation. Higher values will give less
frequent updates. This is also the frequency of screen refreshes if SDL is enabled.  
`#define REPORT_FREQUENCY 200000`

*Mutation rate* -- range is from 0 (none) to 0xffffffff (all mutations!) To get it from a float probability from 0.0 to 1.0, multiply it by
 4294967295 (0xffffffff) and round.  
`#define MUTATION_RATE 5000`

*How frequently should random cells / energy be introduced?*
Making this too high makes things very chaotic. Making it too low
might not introduce enough energy.   
`#define INFLOW_FREQUENCY 100`

*Base amount of energy to introduce per INFLOW_FREQUENCY ticks*   
`#define INFLOW_RATE_BASE 600`

A random amount of energy between 0 and this is added to
INFLOW_RATE_BASE when energy is introduced. Comment this out for
no variation in inflow rate.  
`#define INFLOW_RATE_VARIATION 1000`

*Size of pond in X and Y dimensions.*  
`#define POND_SIZE_X 800`
`#define POND_SIZE_Y 600`

*Depth of pond in four-bit codons* -- this is the maximum
genome size. This *must* be a multiple of 16!  
`#define POND_DEPTH 1024`

This is the divisor that determines how much energy is taken
from cells when they try to KILL a viable cell neighbor and
fail. Higher numbers mean lower penalties.
`#define FAILED_KILL_PENALTY 3`

Define this to use SDL. To use SDL, you must have SDL headers
available and you must link with the SDL library when you compile.
Comment this out to compile without SDL visualization support.
`#define USE_SDL 1`

Define this to use threads, and how many threads to create  
`#define USE_PTHREADS_COUNT 4`
