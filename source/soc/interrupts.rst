==========
Interrupts
==========

The BL808 has some important call-outs when it comes to Interrupts so
I am documenting that here as I work through the boot process.


M0
--

The M0 core supports an implementation of the Core-Local Interrupt Controller (CLIC)
extension. At the time of implementation there was not an official spec, only
drafts. This means that there are implementation details here that do not
fully line up with what is documented in the RISC-V specification.

The working document can be found here https://github.com/riscv/riscv-fast-interrupt/blob/master/clic.adoc

The RISC-V Machine specification defines `mtvec` as:

.. wavedrom::

    {
        reg: [
            {bits: 2, name: 'MODE', attr: 'WARL'},
            {bits: 30, name: 'BASE[32-1:2]', attr: 'WARL (assuming MXLEN=32)'},
        ]
    }


.. table:: MODE

    +-------+----------+-------------------------------------------------+
    | Value |  Name    | Description                                     |
    +=======+==========+=================================================+
    | 0     | Direct   |  All exceptions set `pc` to BASE.               |
    +-------+----------+-------------------------------------------------+
    | 1     | Vectored | Asynchronous interrupts set pc to BASE+4×cause. |
    +-------+----------+-------------------------------------------------+
    | >=2   |     -    | `Reserved`                                      |
    +-------+----------+-------------------------------------------------+

With CLIC the `Reserved` mode of 3 is set which changes the `mtvec` register.
Only a submode of `0000` is currently supported.  Because of the 6 bits being
used for indicating operation mode, the base address must be 64-byte aligned.

.. wavedrom::

    {
        reg: [
            {bits: 2, name: 'MODE', attr: 'WARL'},
            {bits: 4, name: 'SUBMODE', attr: 'WARL'},
            {bits: 26, name: 'BASE[32-1:6]', attr: 'WARL (assuming MXLEN=32)'},
        ]
    }

.. table::

    +---------+------+--------------------------------------------------------------------------+
    | Submode | Mode | Action on Interrupt                                                      |
    +=========+======+==========================================================================+
    | aaaa    | 00   |  All exceptions set `pc` to BASE.                                        |
    +---------+------+--------------------------------------------------------------------------+
    | aaaa    | 01   | Asynchronous interrupts set pc to BASE+4×cause.                          |
    +---------+------+--------------------------------------------------------------------------+
    | 0000    | 11   |CLIC Mode                                                                 |
    |         |      |                                                                          |
    |         |      |.. code-block::                                                           |
    |         |      |                                                                          |
    |         |      |    (non-vectored)                                                        |
    |         |      |    pc := NBASE                              if clicintattr[i].shv = 0    |
    |         |      |                                             || if NVBITS = 0             |
    |         |      |                                                (vector not supported)    |
    |         |      |                                                                          |
    |         |      |    (vectored)                                                            |
    |         |      |    pc := M[TBASE + XLEN/8 * exccode)] & ~1  if clicintattr[i].shv = 1    |
    |         |      |                                                                          |
    +---------+------+--------------------------------------------------------------------------+
    | 0000    | 10   |                                   Reserved                               |
    +---------+------+--------------------------------------------------------------------------+
    | xxxx    | 1?   | (xxxx!=0000)                      Reserved                               |
    +---------+------+--------------------------------------------------------------------------+


Working with CLIC
~~~~~~~~~~~~~~~~~

For the CLIC there are a series of memory mapped registers that control it.
In some cores this address can be found using the mclicbase CSR, but that is
not supported by many cores including those in the BL808.  Instead we have
to find the base from the core header file supplied in the SDK.  This file for
the M0 list several important base addresses:

.. code-block:: c

    /* Memory mapping of THEAD CPU */
    #define TCIP_BASE   (0xE000E000UL) /*!< Titly Coupled IP Base Address */
    #define CORET_BASE  (0xE0004000UL) /*!< CORET Base Address */
    #define CLIC_BASE   (0xE0800000UL) /*!< CLIC Base Address */
    #define SYSMAP_BASE (0xEFFFF000UL) /*!< SYSMAP Base Address */
    #define DCC_BASE    (0xE4010000UL) /*!< DCC Base Address */
    #define CACHE_BASE  (TCIP_BASE + 0x1000UL


The memory map is layed out like this:
https://github.com/riscv/riscv-fast-interrupt/blob/master/clic.adoc#clic-memory-map

.. code-block:: c
    :caption: M-mode CLIC memory map

    Offset
    ###   0x0008-0x003F              reserved    ###
    ###   0x00C0-0x07FF              reserved    ###
    ###   0x0800-0x0FFF              custom      ###

    0x0000         1B          RW        cliccfg


    0x0040         4B          RW        clicinttrig[0]
    0x0044         4B          RW        clicinttrig[1]
    0x0048         4B          RW        clicinttrig[2]
    ...
    0x00B4         4B          RW        clicinttrig[29]
    0x00B8         4B          RW        clicinttrig[30]
    0x00BC         4B          RW        clicinttrig[31]


    0x1000+4*i     1B/input    R or RW   clicintip[i]
    0x1001+4*i     1B/input    RW        clicintie[i]
    0x1002+4*i     1B/input    RW        clicintattr[i]
    0x1003+4*i     1B/input    RW        clicintctl[i]
    ...
    0x4FFC         1B/input    R or RW   clicintip[4095]
    0x4FFD         1B/input    RW        clicintie[4095]
    0x4FFE         1B/input    RW        clicintattr[4095]
    0x4FFF         1B/input    RW        clicintctl[4095]

TODO: Add more context on the configuration of these registers.

Interrupt Handling
~~~~~~~~~~~~~~~~~~

Reference here:
https://github.com/riscv/riscv-fast-interrupt/blob/master/clic.adoc#9-interrupt-handling-software


Using the hardware vectoring with CLIC we are able to have the controller
quickly jump to our interrupt handler without having to make use of a trampoline
but this does present an issue if we supply a function directly from C as we
will not have properly handled saving and restoring registers.

There is a standard attribute that has been defined, `__attribute__ ((interrupt))`,
which has been implemented by both GCC and Clang:

 * GCC - https://gcc.gnu.org/onlinedocs/gcc/RISC-V-Function-Attributes.html
 * Clang - https://clang.llvm.org/docs/AttributeReference.html#interrupt-riscv

This allows the compiler to do the work of tracking the registers that need
to be saved and restored.

Lets use Compiler Explorer to take a look at what this simple handler function
looks like:

.. code-block:: c
    :caption: https://godbolt.org/z/E7fP79ncc

    void
    fast_handler (void)
    {
        extern volatile int INTERRUPT_FLAG;
        INTERRUPT_FLAG = 0;
        extern volatile int COUNTER;
        COUNTER++;
    }

.. code-block:: asm

    fast_handler:
        lui     a5,%hi(INTERRUPT_FLAG)
        sw      zero,%lo(INTERRUPT_FLAG)(a5)
        lui     a4,%hi(COUNTER)
        lw      a5,%lo(COUNTER)(a4)
        addiw   a5,a5,1
        sw      a5,%lo(COUNTER)(a4)
        ret

This is about what we would expect for this, we use a4 and a5 to manipulate
the INTERRUPT_FLAG and COUNTER variables and then return.  There is nothing
here doing the work needed to save and restore these two variables.
Traditionally a trampoline function would be used to handle this assuming
that all registers would need to be saved.




Now adding in the attribute:

.. code-block:: c
    :caption: https://godbolt.org/z/jEx9PT19z

    void __attribute__ ((interrupt))
    fast_handler (void)
    {
        extern volatile int INTERRUPT_FLAG;
        INTERRUPT_FLAG = 0;
        extern volatile int COUNTER;
        COUNTER++;
    }

.. code-block:: asm

    fast_handler:
        addi    sp,sp,-16
        sd      a5,0(sp)
        lui     a5,%hi(INTERRUPT_FLAG)
        sd      a4,8(sp)
        sw      zero,%lo(INTERRUPT_FLAG)(a5)
        lui     a4,%hi(COUNTER)
        lw      a5,%lo(COUNTER)(a4)
        addiw   a5,a5,1
        sw      a5,%lo(COUNTER)(a4)
        ld      a4,8(sp)
        ld      a5,0(sp)
        addi    sp,sp,16
        mret

We now see that only registers a4 and a5 have been stored on the stack and
restored.  If counter is a float we we see fa5 being tracked on the stack as
well.

.. code-block:: asm
    :caption: https://godbolt.org/z/P36rjnEMr

    fast_handler:
        addi    sp,sp,-32
        sd      a5,16(sp)
        lui     a5,%hi(INTERRUPT_FLAG)
        sd      a4,24(sp)
        sw      zero,%lo(INTERRUPT_FLAG)(a5)
        lui     a4,%hi(.LC0)
        lui     a5,%hi(COUNTER)
        fsd     fa4,8(sp)
        fsd     fa5,0(sp)
        flw     fa4,%lo(.LC0)(a4)
        flw     fa5,%lo(COUNTER)(a5)
        ld      a4,24(sp)
        fadd.s  fa5,fa5,fa4
        fld     fa4,8(sp)
        fsw     fa5,%lo(COUNTER)(a5)
        ld      a5,16(sp)
        fld     fa5,0(sp)
        addi    sp,sp,32
        mret
    .LC0:
        .word   1065353216


.. note::
    
    Special consideration must be take if it is desired to support preempting
    interrupts as mcause and mepc.  For many more details I recommend reading
    the document linked at the top of this section.

D0
--

To be added later, we will be getting into details about the PLIC.