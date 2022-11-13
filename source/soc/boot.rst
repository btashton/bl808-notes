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


.. note:: 
    More digging to be sorted here as we use the same linker script for the
    M0 and D0 and the code_memory region is an interesting address. There must
    be some remapping going on.

* M0 used: M1s_BL808_SDK/components/platform/soc/bl808/startup_bl808/evb/ld/bl808_flash.ld
* D0 used: M1s_BL808_SDK/components/platform/soc/bl808/bl808/evb/ld/bl808_flash.ld
