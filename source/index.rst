.. BL808 Notes documentation master file, created by
   sphinx-quickstart on Sat Nov 12 19:14:48 2022.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Notes Developing with the BL808
===============================

.. toctree::
   :maxdepth: 2
   :caption: Contents:

BL808
-----

The BL808 is Bouffalo Lab's higher end AIoT chip, they describe as:

   BL808 is a highly integrated AIoT chipset with Wi-Fi/BT/BLE/Zigbee, Multi-Core CPUs, Audio Codec , Video Codec and AI HW accelerator for high-performance & low-power application.


.. figure:: _static/images/Functionalblockdiagram.svg
   :align: center

   Block Diagram


Development Boards
------------------

Ox64
^^^^
This is a simple development board from Pine64:
 * 26 GPIO pins
 * on-board NOR flash (16Mb or 128Mb)
 * microSD
 * JTAG Header

More information on this board can be found `here <https://wiki.pine64.org/wiki/Ox64>`_.


M1s
^^^
This is a small but feature full board from Sipeed:
 * 30 GPIO pins
 * on-board NOR flash (16MB)
 * on-board Wi-Fi / BLE Antenna
 * microSD (SDHC)
 * USB 2.0 HS OTG
 * USB Dual UART (BL702)
 * 1.69in 280x240 display with CTP
 * Analog MEMS mic
 * 2MP MIPI OV2685
 * JTAG

More information on this board can be found
`here <https://wiki.sipeed.com/hardware/zh/maix/m1s/other/get_key.html>`_
and
`here <https://www.indiegogo.com/projects/sipeed-maix-new-experience-to-risc-v-aiot-tinyml>`_.

Indices and tables
==================

* :ref:`genindex`
* :ref:`search`
