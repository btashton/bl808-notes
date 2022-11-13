=============
Memory Layout
=============

Lets take a look at how the memory address space is partitioned. The information
here can be found in the BL808 Reference manual.

.. table:: Address mapping

    +---------------+---------------+-----------------------+-------+-----------------------------------------------------------------------------------------------------------+
    |  Module       |  Destination  |  Start Address        | Size  |         Description                                                                                       |
    +===============+===============+=======================+=======+===========================================================================================================+
    | FLASH         | FlashA        | 0x58000000            | 64MB  | Application address space                                                                                 |
    +---------------+---------------+-----------------------+-------+-----------------------------------------------------------------------------------------------------------+
    | PSRAM         | pSRAM         | 0x54000000            | 64MB  | pSRAM memory address space                                                                                |
    +---------------+---------------+-----------------------+-------+-----------------------------------------------------------------------------------------------------------+
    | RAM           | OCRAM(MCU)    | 0x22020000            | 64KB  | On Chip RAM address space mainly for M0 application data                                                  |
    +               +---------------+-----------------------+-------+-----------------------------------------------------------------------------------------------------------+
    |               | WRAM(MCU)     | 0x22030000            | 160KB | Wireless RAM address space mainly for M0 wireless network data                                            |
    +               +---------------+-----------------------+-------+-----------------------------------------------------------------------------------------------------------+
    |               | XRAM(EMI)     | 0x40000000            | 16KB  | Shared RAM mainly for data communication among M0/LP/D0                                                   |
    +               +---------------+-----------------------+-------+-----------------------------------------------------------------------------------------------------------+
    |               | DRAM(MM)      | 0x3EF80000            | 512KB | Multimedia-side RAM address space used by D0 application data and modules like H264/NPU                   |
    +               +---------------+-----------------------+-------+-----------------------------------------------------------------------------------------------------------+
    |               | VRAM(MM)      | 0x3F000000            | 32KB  | Multimedia-side RAM address space used by D0 application data and modules like H264/NPU                   |
    +---------------+---------------+-----------------------+-------+-----------------------------------------------------------------------------------------------------------+

One things that is important to call out here which is only mentioned in the
datasheet is the mapping of RAM via the cache.

.. table:: Cached RAM Memory Map 

    +-----------------+-------+-------------+----------------+-------------+----------------+
    |  Module         | Size  |  Base Address(M0)            |  Base Address(D0)            |
    +                 +       +-------------+----------------+-------------+----------------+
    |                 |       | Cache       | Non-cache      | Cache       | Non-cache      |
    +=================+=======+=============+================+=============+================+
    | OCRAM(MCU)      | 64KB  | 0x62020000  | 0x22020000     | \-          | 0x22020000     |
    +-----------------+-------+-------------+----------------+-------------+----------------+
    | WRAM(MCU)       | 160KB | 0x62030000  | 0x22030000     | \-          | 0x22030000     |
    +-----------------+-------+-------------+----------------+-------------+----------------+
    | DRAM(MM)        | 512KB | \-          | 0x3EF80000     | 0x3EF80000  | \-             |
    +-----------------+-------+-------------+----------------+-------------+----------------+
    | VRAM(MM)        | 32KB  | \-          | 0x3F000000     | 0x3F000000  | \-             |
    +-----------------+-------+-------------+----------------+-------------+----------------+


The datasheet also calls this out about the cached access:
    OCRAM and WRAM can be accessed either through AHB bus or through AXI.
    When the CPU accesses OCRAM using the address of 0x62020000, it will go
    through the internal cache and transfer AXI to AHB to achieve access to
    OCRAM. When the CPU uses the 0x22020000 address to access OCRAM, it does
    not go through the internal cache and directly accesses OCRAM through the
    AHB bus.

