.. _dbg-gdb:

GDB Debugging (Zephyr)
######################

GDB can be used to debug SOF on Zephyr with breakpoints and memory access.

Enabling GDB in Zephyr build
****************************

``GDBSTUB=y`` must be set in the build options to use GDB. To do so, run ``west
build -t menuconfig -b build-[target (ie cnl, apl)]`` to open menuconfig, and
press ``/`` to search. If you are using the open source GDB stub, make sure to
disable ``ADSP_XT_GDB`` so that the correct register definitions are used.

Toolchain note
--------------

The open source Zephyr toolchain GCC does not handle certain registers
correctly, which can cause issues with debugging. It is recommended to use the
Xtensa proprietary toolchain when building for GDB, as well as xt-gdb.

Usage with Cavstool
*******************

If you would like to load and debug the firmware with Cavstool, simply pass the
``-d`` arg to Cavstool. It will open a virtual serial port that you can connect
to with GDB's remote functionality:

.. code-block::

    INFO:cavs-fw:cAVS firmware load complete
    INFO:cavs-fw:GDB PTY at: /dev/pts/5

To use GDB, run the following commands:

.. code-block::

    file [path to zephyr elf (build-[target]/zephyr/zephyr.elf)] # Load symbol table
    target remote /dev/pts/5 # Connect to GDB stub (make sure the number matches what was printed by Cavstool)

You should now be able to set breakpoints, inspect memory and variables, and
anything else you can do with GDB.

Usage with Kernel Driver
************************

Firstly, ``GDBSTUB_ENTER_IMMEDIATELY`` must be disabled in the FW config so that
the kernel driver can complete the boot sequence without interruption from GDB.
Additionally, the kernel must be built with the ``SND_SOC_SOF_DEBUG_FW_GDB``
option enabled. The kernel driver will add a debugfs file in
``/sys/kernel/debug/sof/fw_gdb`` which functions like the pty from Cavstool. Use
the target remote command to connect to that file, and you should now be able to
debug with GDB:

.. code-block::

    file [path to zephyr elf (build-[target]/zephyr/zephyr.elf)] # Load symbol table
    target remote /sys/kernel/debug/sof/fw_gdb # Connect to GDB stub


Connecting GDB over SSH
***********************

GDB expects to be running on the machine used to build SOF, which is likely not
the target device. You can use socat to forward Cavstool's pts or the kernel
driver's debugfs file to a network port which can be accesses over SSH. This
allows GDB to read source files and display more helpful information.

.. code-block:: bash

    # Connect over SSH and forward port 4000:
    ssh -4 -L 4000:localhost:4000 user@target
    # Cavstool:
    sudo socat /dev/pts/[number listed from cavstool],raw,echo=0 tcp-listen:4000
    # Kernel driver:
    sudo socat PIPE:/sys/kernel/debug/sof/fw_gdb tcp-listen:4000

Now, use ``target remote :4000`` (run GDB on the host machine) to connect GDB
across SSH and to the GDB stub.
