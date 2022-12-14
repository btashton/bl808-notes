=================
Booting the BL808
=================

BootROM
-------

The BootROM is located at 0x90000000 and will either start the application
from Flash/UART/USB depending on pin and efuse configuration.

If the bootpin is active at start this also provides a bootloader which
BLDevCube can use to write the flash and efuse configuration. It is also able
handle firmware decryption and verification if used.

@lupyuen has written in more detail about this here:
https://lupyuen.github.io/articles/boot

Boot2
-----
This optional second stage bootloader can be used for booting using a partition
table style boot configuration.  More details in the following section. We will
be mostly using this going forward. In BLDevCube this shows up when using the
"IOT" configuration.

Partition Table
---------------

There is a bunch of information to fill in here since this is not (as far as I
have found) publicly documented. We will start by looking at
the default partition table for the M1s Dock

.. code-block:: toml
    :caption: partition_cfg_16M_m1sdock.toml

    [pt_table]
    #partition table is 4K in size
    address0 = 0xE000
    address1 = 0xF000
    # If version is 2, It will use dynamic mode.
    version = 2

    [[pt_entry]]
    type = 16
    name = "Boot2"
    device = 0
    address0 = 0
    size0 = 0xE000
    address1 = 0
    size1 = 0
    # compressed image must set len,normal image can left it to 0
    len = 0
    # If header is 1, it will add the header.
    header = 1

    [[pt_entry]]
    type = 0
    name = "FW"
    device = 0
    address0 = 0x10000
    size0 = 0xF0000
    address1 = 0
    size1 = 0
    # compressed image must set len,normal image can left it to 0
    len = 0
    # If header is 1, it will add the header.
    header = 1

    [[pt_entry]]
    type = 2
    name = "D0FW"
    device = 0
    address0 = 0x100000
    size0 = 0x200000
    address1 = 0
    size1 = 0
    # compressed image must set len,normal image can left it to 0
    len = 0
    # If header is 1, it will add the header.
    header = 1

    [[pt_entry]]
    type = 5
    name = "media"
    device = 0
    address0 = 0x300000
    size0 = 0xC00000
    address1 = 0
    size1 = 0
    # compressed image must set len,normal image can left it to 0
    len = 0
    # If header is 1, it will add the header.
    header = 1

    [[pt_entry]]
    type = 11
    name = "unused"
    device = 0
    address0 = 0xF00000
    size0 = 0x100000
    address1 = 0
    size1 = 0
    # compressed image must set len,normal image can left it to 0
    len = 0
    # If header is 1, it will add the header.
    header = 1

    [[pt_entry]]
    type = 8
    # It shows Dts in DevCube 
    name = "factory"
    device = 0
    address0 = 0x910000
    size0 = 0
    address1 = 0
    size1 = 0
    # compressed image must set len,normal image can left it to 0
    len = 0
    # If header is 1, it will add the header.
    header = 1



Lets first take a look at the `[pt-table]` entry:

.. code-block:: toml

    [pt_table]
    #partition table is 4K in size
    address0 = 0xE000
    address1 = 0xF000
    # If version is 2, It will use dynamic mode.
    version = 2

There can be two partition table IDs, only one of them will be active.
The two here are defined as starting at 0xE000 and 0xF000. The offset between
the two makes sense if the at 4K (0x1000).  But why does it not start at 0?
That appears to be because the second stage bootloader starts at 0x0000 and
has been given a length of 0xE000. This makes sense if the BootROM starts
executing at the start of the Flash.

.. code-block:: toml
    :caption: boot2

    [[pt_entry]]
    type = 16
    name = "Boot2"
    device = 0
    address0 = 0
    size0 = 0xE000
    address1 = 0
    size1 = 0
    # compressed image must set len,normal image can left it to 0
    len = 0
    # If header is 1, it will add the header.
    header = 1


address1 and size1 are not used in any of these entries likely because
we do not use the second partition table.

.. note::
    We do not yet understand the `version` or `header` fields.

These two entries will be important when we define the linker scripts for
building the M0 and D0 firmware.

.. code-block:: 

    [[pt_entry]]
    type = 0
    name = "FW"
    device = 0
    address0 = 0x10000
    size0 = 0xF0000

    [[pt_entry]]
    type = 2
    name = "D0FW"
    device = 0
    address0 = 0x100000
    size0 = 0x200000


Lets take a look at the memory map defined for boot2

.. code-block:: 
    :caption: blsp_boot2_iap_flash.ld

    MEMORY
    {
        xip_memory  (rx)  : ORIGIN = 0x58000000, LENGTH = 48K
        itcm_memory (rx)  : ORIGIN = 0x62020000, LENGTH = 48K
        dtcm_memory (rx)  : ORIGIN = 0x6202C000, LENGTH = 4K
        nocache_ram_memory (!rx) : ORIGIN = 0x2202D000, LENGTH = 84K
        ram_memory  (!rx) : ORIGIN = 0x62042000, LENGTH = 16K
    }

There is not too much interesting here the XIP address is the Flash address we
see in the memory map, and 48+4KB of ram is carved out of the OCRAM. There is
some additional non-cached memory the spans the remainder of the OCRAM and into
the WRAM (ends at 0x22042000) where the ram_memory region picks up.

.. note:: 
    Unfortunately we do not know too much about what happens inside of the boot2
    loader, we we will document what we discover as we go.

Now lets move on to the next stage of the boot process.  The M0 core, first
we take a look at the linker file that was used.

::

    M1s_BL808_SDK/components/platform/soc/bl808/startup_bl808/evb/ld/bl808_flash.ld

.. code-block::
    
    BOOT2_PT_ADDR = 0x22057C00;
    BOOT2_FLASHCFG_ADDR = 0x22057c18;

    MEMORY
    {
        flash    (rxai!w) : ORIGIN = 0x58000000, LENGTH = 4M
        xram_memory (!rx) : ORIGIN = 0x22020000, LENGTH = 16K 
        ram_memory  (!rx) : ORIGIN = 0x22024000, LENGTH = 48K
        ram_wifi    (!rx) : ORIGIN = 0x22030000, LENGTH = 96K 
        ram_psram  (!rx)  : ORIGIN = 0x50000000, LENGTH = 1M
        bugkill   (rxai!w) : ORIGIN = 0xD0000000, LENGTH = 16M
    }


There are a couple interesting things carved out here.

.. note::
    
    There is the 4M of Flash at 0x58000000, remember this is also the address
    that boot2 was placed at.

This leads to some more details about how the flash is presented on the bus at
that address. I will describe my understanding in more detail later, but the
tl;dr; is the sflash controller is configured by boot2 to map for the M0
the physical address at offset 0x10000 to 0x58000000.  This can be configured
per core.

.. note:: 
    
    The xram is called out as the first 16K of the OCRAM (on-chip ram).  I would
    have expected this to be the xram at 0x40000000.  I have asked for more
    clarity on this point from Sipeed
    https://github.com/sipeed/M1s_BL808_SDK/issues/2



Early Boot
----------

Now lets take a look at what the first part of the boot process looks like for
the e907 once the boot2 bootloader has mapped 0x58000000 to the start of our
executable and has made the jump.

This is based off of M1s_BL808_SDK/components/platform/soc/bl808/startup_bl808/evb/src/boot/gcc/startup.S)
and I have removed the code that is conditioned on RUN_IN_RAM, since that
this is focused on booting from flash.  I may return back to talk about booting
from RAM as that would potentially speed up our development iteration time.

.. code-block:: asm

    _start:
        .section      .text.entry
        .align  2
        .globl  risc_e906_start
        .type   risc_e906_start, %function
    risc_e906_start:
    .option push
    .option norelax
        la      gp, __global_pointer$
    .option pop


There are already some interesting things to call out here:

First, the entry point function is defined as risc_e906, and this corresponds
to the ENTRY_POINT(risc_e906) defined in the linker script.

Second, the initialization of the global pointer.  This is important for us to
allow the linker to perform global-pointer relaxations.  This specific line
that sets us up for this needs to disable the relaxations or we will end up
making this assignment itself a noop.  Prior to setting the norelax we use the
push option to store the current options on the option stack so they can be
restored with pop. See https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md#pushpop
For more information on global pointer relaxations see this section of the RISCV
ELF specification https://github.com/riscv-non-isa/riscv-elf-psabi-doc/blob/master/riscv-elf.adoc#global-pointer-relaxation


Now we start setting up the interrupt handling.

.. note:: 

    I highly recommend reading :doc:`interrupts` for additional context.

.. code-block:: asm

        la      a0, freertos_risc_v_trap_handler
        ori     a0, a0, 3
        csrw    mtvec, a0 

        la      a0, __Vectors
        csrw    mtvt, a0

First we set the machine trap vector csr (mtvec).  This is the address with the
mode set.  Note that we are setting the mode to 3 which is normally a reserved
mode, _except_ this processor supports the CLIC interrupt controller.
See :doc:`interrupts`.

Next the csr `mtvt` is configured to point at the base of a table of handler
functions (not instructions).  For RV32 4-byte pointers, for RV64 8-byte.

.. note:: 

    Both CLIC and CLINT support "vectored" modes, but the key difference
    in the table is that CLIC is a table of handler pointers, where CLINT is is
    table of instructions to jump to to handlers.

    For CLINT this might look like:

    .. code-block:: asm

            __Vectors:
            j   exception_common		/* 0 */
            j   Default_Handler			/* 1 */
            j   Default_Handler			/* 2 */
            j   Default_Handler			/* 3 */
            j   Default_Handler			/* 4 */

    vs for CLIC:

    .. code-block:: c

        const pFunc __Vectors[] __attribute__ ((section(".init"),aligned(64))) = {
        Default_Handler,
        Default_Handler,
        Default_Handler,


This section is just storing the top of the stack in the scratch machine
mode scratch space so it can be swapped to if needed when entering a trap
vector:

.. code-block:: asm

    .weak __StackTop
    la      sp, __StackTop
    csrw    mscratch, sp


To be continued...

* D0 used: M1s_BL808_SDK/components/platform/soc/bl808/bl808/evb/ld/bl808_flash.ld
