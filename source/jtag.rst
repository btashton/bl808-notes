==========
Using JTAG
==========

I tried using my Tigard and my RV-Debugger Plus adapters with openocd, but was not able to get it to talk the core.
The documentation talks mostly about using the CKLink which is something from T-Head/C-SKY.  I do not have one of
these, although I did order one off aliexpress.  I noticed that the Lichee RV Dock Pro from SipeedIO provides a CKLink
interface to the D1 via a BL702 chip, so I reached out and Bouffalo Lab released the firmware for the RV-Debugger Plus
to be able to use it as a CKLink device.

You can find this information here https://github.com/bouffalolab/bl_mcu_sdk/tree/master/tools/cklink_firmware

One I restarted it, it shows up as a CKLink device.

.. code-block::

    ❯ lsusb
    ....
    Bus 001 Device 046: ID 42bf:b210 Bouffalo C-Sky CKLink-Lite

CKLink Software
----------------

You can find the DebugServer `here <https://occ-oss-prod.oss-cn-hangzhou.aliyuncs.com/resource//1666331533949/T-Head-DebugServer-linux-x86_64-V5.16.5-20221021.sh.tar.gz>`_.
When you try to install it, it will assert that you need to be a root user.
I am not a big fan of giving software like this root permissions so I installed
it within a restricted docker container and then copied out the
installation files.  Likely the reason that it wants to run as root is to
install udev rules.

If you run the server without root permissions you will see the adapter show up
as *Busy*.

.. code-block::

    ❯ ./DebugServerConsole.elf -list-ice
    +---                                                    ---+
    |  T-Head Debugger Server (Build: Oct 21 2022)	           |
    User   Layer Version : 5.16.05 
    Target Layer version : 2.0
    |  Copyright (C) 2022 T-HEAD Semiconductor Co.,Ltd.        |
    +---                                                    ---+
    The current connected CKLink list:
        [Nr]  CKLink Devices             State
        [ 0]  CKLink_Lite_Vendor-001-045  Busy

If you add a udev rule such as this you will be able to interact with the device
without root. I leave it up to the reader to determine how they want to
configure this rule, including limiting to specific users/groups.

.. code-block::

    ❯ cat /etc/udev/rules.d/99-cklink.rules
    ACTION=="add", ATTR{idVendor}=="42bf", ATTR{idProduct}=="b210", MODE:="666"

The device now shows up as IDLE

.. code-block::

    ❯ ./DebugServerConsole.elf -list-ice
    +---                                                    ---+
    |  T-Head Debugger Server (Build: Oct 21 2022)	           |
    User   Layer Version : 5.16.05 
    Target Layer version : 2.0
    |  Copyright (C) 2022 T-HEAD Semiconductor Co.,Ltd.        |
    +---                                                    ---+
    The current connected CKLink list:
        [Nr]  CKLink Devices             State
        [ 0]  CKLink_Lite_Vendor-rog E39700  IDLE

Debug Server
-------------

Now that we have a CKLink device and the DebugServer software working, connect the JTAG pins as defined above
to the pins on the RV-Debugger Plus silkscreen pinout.


1. On the ox64, press the boot button, connect power, release the boot button.
2. Start the DebugServer

.. code-block::

    ❯ ./DebugServerConsole.elf
    +---                                                    ---+
    |  T-Head Debugger Server (Build: Oct 21 2022)	           |
    User   Layer Version : 5.16.05 
    Target Layer version : 2.0
    |  Copyright (C) 2022 T-HEAD Semiconductor Co.,Ltd.        |
    +---                                                    ---+
    T-HEAD: CKLink_Lite_V2, App_ver unknown, Bit_ver null, Clock 2526.316KHz,
        5-wire, With DDC, Cache Flush On, SN CKLink_Lite_Vendor-rog E39700.
    +--  Debug Arch is RVDM.  --+
    +--  CPU 0  --+
    RISCV CPU Info:
        WORD[0]: 0x0814050d
        WORD[1]: 0x11080000
        WORD[2]: 0x202bf97b
        MISA   : 0x40909125
    Target Chip Info:
        CPU Type is E907FP, Endian=Little, Version is R1S2P0.
        DCache size is 16K.
        ICache size is 32K.
        MGU zone num is 8.
        MGU zone size is 4B.
        HWBKPT number is 3, HWWP number is 3.
        MISA: (RV32IMAFCXP, Imp M-mode, U-mode)

    GDB connection command for CPUs(CPU0):
        target remote 127.0.0.1:1025
        target remote 192.168.86.211:1025
        target remote 192.168.122.1:1025

    ****************  DebuggerServer Commands List **************
    help/h
        Show help informations.
    *************************************************************


3. Connect via GDB

.. code-block::

    ❯ riscv64-unknown-elf-gdb
    GNU gdb (SiFive GDB-Metal 10.1.0-2020.12.7) 10.1
    Copyright (C) 2020 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.
    Type "show copying" and "show warranty" for details.
    This GDB was configured as "--host=x86_64-pc-linux-gnu --target=riscv64-unknown-elf".
    Type "show configuration" for configuration details.
    For bug reporting instructions, please see:
    <https://github.com/sifive/freedom-tools/issues>.
    Find the GDB manual and other documentation resources online at:
        <http://www.gnu.org/software/gdb/documentation/>.

    For help, type "help".
    Type "apropos word" to search for commands related to "word".
    (gdb) tar rem :1025
    Remote debugging using :1025
    warning: No executable has been specified and target does not support
    determining executable automatically.  Try using the "file" command.
    0x90004fb0 in ?? ()
    (gdb) set pagination off
    (gdb) info all-registers
    zero           0x0	0
    ra             0x900052e2	0x900052e2
    sp             0x6204cb60	0x6204cb60
    gp             0x62052080	0x62052080
    tp             0x0	0x0
    t0             0x62052830	1644505136
    t1             0x62052e14	1644506644
    t2             0x0	0
    fp             0x10000	0x10000
    s1             0x6204bbec	1644477420
    a0             0xfffe	65534
    a1             0x0	0
    a2             0xd39985	13867397
    a3             0x4e1f	19999
    a4             0x9f1	2545
    a5             0x0	0
    a6             0x3	3
    a7             0x7fffffff	2147483647
    s2             0x62052d1c	1644506396
    s3             0xfffe	65534
    s4             0x0	0
    s5             0x62053000	1644507136
    s6             0x10074	65652
    s7             0x90011000	-1878978560
    s8             0x90011000	-1878978560
    s9             0x0	0
    s10            0x0	0
    s11            0x0	0
    t3             0x7	7
    t4             0xc0000	786432
    t5             0x0	0
    t6             0x0	0
    pc             0x90004fb0	0x90004fb0
    ft0            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft1            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft2            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft3            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft4            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft5            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft6            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft7            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs0            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs1            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fa0            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fa1            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fa2            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fa3            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fa4            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fa5            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fa6            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fa7            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs2            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs3            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs4            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs5            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs6            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs7            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs8            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs9            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs10           {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fs11           {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft8            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft9            {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft10           {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    ft11           {float = 0x0, uint32 = 0x0}	{float = 0, uint32 = 0}
    fflags         0x0	RD:0 NV:0 DZ:0 OF:0 UF:0 NX:0
    frm            0x0	FRM:0 [RNE (round to nearest; ties to even)]
    fcsr           0x0	RD:0 NV:0 DZ:0 OF:0 UF:0 NX:0 FRM:0 [RNE (round to nearest; ties to even)]
    vxsat          0x0	0
    mstatus        0x3808	SD:0 VM:00 MXR:0 PUM:0 MPRV:0 XS:0 FS:1 MPP:3 HPP:0 SPP:0 MPIE:0 HPIE:0 SPIE:0 UPIE:0 MIE:1 HIE:0 SIE:0 UIE:0
    misa           0x40909125	RV32ACFIMPUX
    mie            0x0	0
    mtvec          0x90000203	-1879047677
    mcounteren     0x0	0
    mtvt           0x90012700	-1878972672
    mcountinhibit  0x0	0
    mhpmevent3     0x0	0
    mhpmevent4     0x0	0
    mhpmevent5     0x0	0
    mhpmevent6     0x0	0
    mhpmevent7     0x0	0
    mhpmevent8     0x0	0
    mhpmevent9     0x0	0
    mhpmevent10    0x0	0
    mhpmevent11    0x0	0
    mhpmevent12    0x0	0
    mhpmevent13    0x0	0
    mhpmevent14    0x0	0
    mhpmevent15    0x0	0
    mhpmevent16    0x0	0
    mhpmevent17    0x0	0
    mscratch       0x62052830	1644505136
    mepc           0x0	0
    mcause         0x30000000	805306368
    mtval          0x0	0
    mip            0x0	0
    mnxti          0x0	0
    mintstatus     0x0	0
    mscratchcsw    0x0	0
    pmpcfg0        0x0	0
    pmpcfg1        0x0	0
    pmpcfg2        0x0	0
    pmpcfg3        0x0	0
    pmpaddr0       0x0	0
    pmpaddr1       0x0	0
    pmpaddr2       0x0	0
    pmpaddr3       0x0	0
    pmpaddr4       0x0	0
    pmpaddr5       0x0	0
    pmpaddr6       0x0	0
    pmpaddr7       0x0	0
    pmpaddr8       0x0	0
    pmpaddr9       0x0	0
    pmpaddr10      0x0	0
    pmpaddr11      0x0	0
    pmpaddr12      0x0	0
    pmpaddr13      0x0	0
    pmpaddr14      0x0	0
    pmpaddr15      0x0	0
    dcsr           0x400000c3	1073742019
    dpc            0x90004fb0	-1879027792
    mcycle         0x3141c8ae	826394798
    minstret       0x31fbae51	838577745
    mhpmcounter3   0x0	0
    mhpmcounter4   0x0	0
    mhpmcounter5   0x0	0
    mhpmcounter6   0x0	0
    mhpmcounter7   0x0	0
    mhpmcounter8   0x0	0
    mhpmcounter9   0x0	0
    mhpmcounter10  0x0	0
    mhpmcounter11  0x0	0
    mhpmcounter12  0x0	0
    mhpmcounter13  0x0	0
    mhpmcounter14  0x0	0
    mhpmcounter15  0x0	0
    mhpmcounter16  0x0	0
    mhpmcounter17  0x0	0
    mcycleh        0x51	81
    minstreth      0x0	0
    mhpmcounter3h  0x0	0
    mhpmcounter4h  0x0	0
    mhpmcounter5h  0x0	0
    mhpmcounter6h  0x0	0
    mhpmcounter7h  0x0	0
    mhpmcounter8h  0x0	0
    mhpmcounter9h  0x0	0
    mhpmcounter10h 0x0	0
    mhpmcounter11h 0x0	0
    mhpmcounter12h 0x0	0
    mhpmcounter13h 0x0	0
    mhpmcounter14h 0x0	0
    mhpmcounter15h 0x0	0
    mhpmcounter16h 0x0	0
    mhpmcounter17h 0x0	0
    mvendorid      0x5b7	1463
    marchid        0x0	0
    mimpid         0x0	0
    mhartid        0x0	0
    priv           0x3	prv:3 [Machine]
    mscratchcswl   0x0	0
    mclicbase      0xe0800000	-528482304
    mxstatus       0xc0408000	-1069514752
    mhcr           0x103d	4157
    mhint          0x4000	16384
    mraddr         0x90000000	-1879048192
    mexstatus      0x30010	196624
    mnmicause      0x0	0
    mnmipc         0x0	0
    mcpuid         0x202bf97b	539752827
    fxcr           0x0	0
