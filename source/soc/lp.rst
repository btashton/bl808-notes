=======
LP Core
=======

This is based off the T-head e902 design `(datasheet)
<https://www.t-head.cn/product/e902?lang=en>`__

The core is implemented as RV32EMC. The interrupt controller
is implemented with the CLIC interrupt standard.

Implemented standards:
 * The RISC-V Instruction Set Manual, Volume I: RISC-V User-Level ISA, Version 2.2
 * The RISC-V Instruction Set Manual, Volume II: RISC-V Privileged Architecture, Version 1.10
 * RISC-V Core-Local Interrupt Controller (CLIC) Version 0.8