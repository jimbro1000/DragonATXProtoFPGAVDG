# Dragon ATX Prototype FPGA VDG

This repository contains the KiCad project files
to produce the VDG component board for my
ATX Prototype backplane.

![Render of PCB top](./DragonATXProtoFPGAVDG.png)

This design requires the ATX backplane board in order 
to operate
See https://github.com/jimbro1000/DragonATXProto

## Notes

Revision 1 includes a more modern RGB fed modulator
design that removes a significant problem in the 
form of the old LM1889 modulator. The board also
provides the mounting points for the various mezzanine
boards that have been previously developed. If the
mezzanine option is to be used the rest of the card
should be left unpopulated

This design has been completed using KiCad 9. Earlier
versions of KiCad are not compatible

## Design

Unlike previous iterations that attempt to simply
replicate the original 6847 using CPLD hardware, this
design is aimed at a full upgrade and replacement for 
the 6847 and 6883. By combining the functionality of
the two chips are significant saving in complexity is
achieved. There is no longer a requirement to 
artificially synchronise the video against the CPU
clock cycle, this is inherent as the video clock is
tied directly to the cpu clock.

## SAM upgrades

The SAM replacement is based around the work done by
Ciaran Anscomb to upgrade the SAM to include RAM paging.
His design extended to 512KB which, while "adequate", 
is wasteful of the target FPGA device which includes
16MB of RAM. Of that 16MB only 2MB is directly accessed
as a single block. The 2MB is 16bit wide (technically 4MB)
but accessing both halves of the RAM adds extra complexity
to the read/write cycle, just using one half is definitely
simpler.

### Internal Registers ###

As a result of the SAM having full access to the databus the
internal registers can now be fully programmed with a single
write operation (or per bit if required). The registers are
expanded as necessary, for example the video base address is
extended to allow addressing of a much larger memory space,
and the introduction of new registers for process and memory
mapping.

### Memory Paging ###

The upgraded SAM provides MMU capabilities, both paging and
process (page protection). With 256 8K pages in the address
space and each page can be allocated to RAM or ROM. Each 
process has its own page map.

Processes are either:
1. supervisor  
2. display
3. library (shared)
4. user process 1
5. user process 2
6. user process 3
7. user process 4

The shared process and display are used purely for 
identification of shared pages.

#### Display Process ####

The display process is used only for read operations
by the VDG side of the clock cycle. The same pages can
be shared with the supervisor or a user process.

### Paging ###

Each process has its own map of 8K pages composed into
the 64K memory space available directly to the CPU. The
only exception to this is the display process which has
its own 128K memory space and is addressed only by the
VDG in this manner.

Each page has a permissions byte associated with it. The
page is either unallocated (and can be acquired by any
process), or owned by one of the seven process ids. The
display and library process pages are an exception and
can be shared between multiple processes.

The MMU maintains a page allocation table for all 256
available pages. Each process has its own address
translation map, only one map is presented to the CPU
at a time, with the CPU address register translated
into the larger MMU address space.

### Rom Paging ###

Further to the available RAM pages the address map can
reference pages from the ROM space. ROM pages are always
shared.

#### Rom Mapping At Cold Start ####

The MMU will always allocate the last page of the 
supervisor address space to the last ROM page to cover
the CPU vectors. The last ROM page should also include
the bootstrap code to set the host into an operational
state.

At warm start the host will enter the supervisor 
process. Holding the host in reset for 2 seconds will
be treated as a cold start.

### Blitter ###

The blitter provides out-of-process memory copy in one
of 5 operations

1. Full page copy (8K copy from one page to another)
2. Continuous block copy (within process address space)
3. Shaped to continuous (within process address space)
4. Continuous to shaped (within process address space)
5. Shaped to shaped (within process address space)

All operations are defined by an origin, destination and 
byte length (maximum is 8K)

Shaped operations have two further parameters - width
and gap. The width defines how many bytes in a row
are copied and the gap the number of bytes to skip to
the next run of bytes

Copying can only be performed between pages allocated
to the initiating process

Only a single blit operation can be performed at a time,
all operations can be queued

The blit operates at the highest clock speed achievable
by the RAM and SAM. Nominally this is 166MHz with a 3 cycle 
latency, the SAM operates at 28.6MHz and each byte copy
takes 3 cycles. Effectively this means a copy rate of
8 bytes per CPU clock cycle (using interleaved access
between CPU, VDG and Blitter) at regular 0.89MHz CPU
clock speeds. Running at a faster CPU speed would reduce
the available slots for blit operations

The number of cycles for the blitter will be adjusted
to suit the cpu and vdg clock speeds

A maximum copy speed can be achieved by halting the CPU
and timing the blit start to the end of the vertical
display viewport. The slice allocation will reallocate
CPU and VDG portions to the blitter - the result is
lightning fast block copy of ram. Page copy can run
faster as it is guaranteed contiguous on both sides
meaning 64 bytes per copy cycle instead of 8 bytes.
That is 128 CPU cycles (equivalent) to copy 8K at
page copy, 1024 cycles at shaped copy speeds but at
the regular 0.895MHz that is still 0.001s to copy a
full 8K - in reality a shaped copy is likely to be
much less than 8K in length. A regular high-res
screen is just 6K...

## VDG upgrades

The default operation mode for the VDG is full
compatibility with the original MC6847. In practical
terms this means fixed display palettes (although
these could still be modified through HDL source) and
standard resolutions.

With compatibility mode enabled all of the VDG control
is as per the original device - external input for
Alpha/Semi, Alpha/Graphic, Graphic Mode, Inverse,
Colour Set and Internal/External

With compatibility mode disabled the VDG expands 
control to a wide set of registers in the new
device (operated through the SAM for memory
configuration and direct through a new memory
mapped device for mode). The default modes are
still present but can be modified to suit, or
a new set of display modes can be used.

### VDG Registers

The upgraded VDG exists in mapped memory space,
occupying 16 bytes. These form a set of registers
controlling how the video operates.

0. Video Mode Control - 8 bit flags for enabling features
1. Zero RAM Mode - left byte
2. Zero RAM Mode - right byte
3. Zero RAM Mode - palette
4. Palette - register select (0-47)
5. Palette - register data
6. Geometry - register select (0-7)
7. Geometry - register data
8. Sprite - sprite id select
9. Sprite - register select
10. Sprite - register data
11. Tile - tile id select
12. Tile - register select
13. Tile - register data
14. Counter - register select
15. Counter - register data

#### Video Mode Control

Bit 7 controls video display format, 
1 = NTSC, 0 = PAL

Bit 6 controls the use of zero mode,
1 = enabled, 0 = disabled. Zero mode
is a zero ram mode that simply uses
2 static bytes of data per row, set 
using the zero ram mode registers.

Bit 5 enables tile mode. This is 
ignored in compatibility mode.

Bit 4 enables sprites. This is 
ignored in compatibility mode.

Bit 3 enables zero mode as an overlay.
In this state the background colour is
always transparent.

Bit 2 enables the extended graphic
modes. This is ignored in compatibility
mode.

Bit 1 is reserved for future use.

Bit 0 controls compatibility. 1 =
extended options enabled, 0 = 6847
compatible.

#### Zero RAM Mode

Zero RAM mode operates without the
need to load any data from RAM, combined
with a suitable ROM image this means
the host system can display useful
diagnostic data without the need for
working RAM. Zero RAM can also be 
enabled as an overlay over other
display modes.

In Zero mode the display resolution
is 16 pixels wide. The left hand side
of the display is register 1, the right
hand side is register 2.

Register 1 is the first 8 bits of
each row, the same data is displayed
vertically for the entire viewport unless
changed mid-frame. Register is the final
8 bits of each row with the same behaviour
vertically.

Register 3 controls the colour of the
display, the first 4 bits define the
foreground, the last 8 bits define the
background. In overlay mode the background
is always transparent.

The colours are each 4 bit TTL RGB with
half bright. In the case of black the half
bright equivalent is a darker shade of 
grey than white with half bright.

#### Palette Control

The VDG supports 24 different programmable
entries in its palette. Each one is 12 bits
of RGB and require two palette registers to
access each one. Even palette registers represent 
the upper 4 bits (red) of each entry, odd palette
entries represent the lower 8 bits (green and
blue) of each entry.

The first palette register (4) selects which
palette entry is being referenced. With two
registers per entry the value must be multiplied
by two for each entry.

The second palette register (5) defines the
data of the selected palette entry.

#### Geometry Control

In the extended modes the display geometry
can be modified, significantly increasing the
flexibility of thos modes.

VDG Register 6 selects which of the geometry
registers is active. VDG Register 7 accesses 
the value of each.

0. Upper 2 bits of the viewport X position on
screen.
1. Lower 8 bits of the viewport X position.
2. Upper 1 bit of the viewport y position.
3. Lower 8 bits of the viewport y position.
4. Viewport width (in bytes, 0-63).
5. Viewport height (in rows, 0-255).
6. Upper 1 bit of the display height (in rows).
7. Lower 8 bits of the display height.

More geometry registers may be added as new
features are introduced.

#### Sprite Control

Sprites are relatively complex objects
in this system. Each sprite Id can be a graphic
definition, or it can be a path object that
sets how the sprite moves

VDG Register 8 selects which sprite Id is active,
VDG Register 9 selects the address of the object
data. Each sprite object can be up to 255 bytes
long.

VDG Register 10 access the data of the object.

#### Tile Control

Tiles are handled in much the same way as 
sprites for the purposes of defining them.

VDG Register 11 selects which tile Id is
referenced (0-255). VDG Register 12 sets
the address within the selected tile object.
VDG Register 13 accesses the tile data at
the given address.

#### Counters

A small number of counters can be defined
to automatically trigger actions based on
the row or column of the current display
frame.

VDG Register 14 selects which counter
register is active. VDG Register 15
accesses the data for the selected
counter.

### Display Layers

The display presented to the screen is
composed of a series of layers.

Top most are sprites, these always
have a transparency and appear over all
other graphical detail.

The next layer down is the "regular"
bitmap graphic mode. If no other layers
are enabled the "0" colour will be
visible, otherwise it will be transparent.

Below the bitmap mode lies the zero mode.
As with the bitmap mode if tile mode
is not enabled the background colour is
visible, otherwise it will be transparent.

The lowest layer is the tile mode. This
has no transparency and every position
on the display must have a tile.

The composition of layers in this way 
offers the maximum flexibility from the
capabilities of the enhanced VDG