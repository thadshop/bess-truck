The Wakespeed config utility only runs on Windows.
That's not a problem, because it can be a Windows VM.
The Wakespeed WS500 is connected by a USB cable.
The WS500 can be in two different modes: "normal" or DFU.  DFU mode is for device firmware update. Normal mode is for everything else.

What can be tricky is that the WS500 presents itself with a different USB identification, depending on what mode it's in.

While running the config utility in Windows, it may be necessary to change the WS500's mode with the USB cable remaining attached.  Detatching the USB cable will power off the WS500, unless multi-pin connector on the WS500 is properly wired to a power source.  Powering off the WS500 while running the config utility might be cumbersome at best; it's probably possible to successfully run the config utility, while carefully timing when the WS500 is powered on or off.

In any case, keeping the WS500 powered on throughout the config utility execution is the most straightforward.  It's also more straightforward to have the WS500 power supplied through the USB cable, rather than wiring up a power source through the multi-pin connector.

So, to run the config utility, it will be most straightforward to have the USB cable remain attached throughout the process, and it will be necessary for Windows to have USB communication with the WS500, regardless of which mode it's in.

As the WS500's USB identification will change depending on what mode its in, the Windows VM's USB configuration must be such that it can communicate with the WS500 in either mode.  Shutting down the Windows VM to change its configuration would of course terminate the config utility, so the VM's USB configuration must be adaptable to the WS500's current mode, while the VM and config utility remain running.

There are various ways to effect the USB adaptability while things remain running.  It could be done by passing the VM's host machine's USB port through to the VM, but that can be problematic for the host machine as the VM may have to take over a whole wad of the host machine's USB apparatus associated with the particular port.  Another way could be putting a USB hub between the host machine and the WS500; the premise is that the USB hub would always look the same to the VM, regardless of what mode the WS500 may be in; this of course requires having a USB hub, and depending on the type of hub it may still be tricky to isolate the VM from the changing modes of the WS500.  There may be a more straightforward way to effect the adaptability.

With a virtual machine manager like https://virt-manager.org/, there is a fairly straightforward way.  With it, the VM's virtual hardware can be changed on the fly, while the VM is running.  The VM needs different USB host device virtual hardware configurations for each of the WS500's modes, as the mode changes its USB identification.  The Windows VM needs whatever USB host device configuration it has to be valid, in order to communicate with the WS500.  When the WS500 is in one mode, the VM configuration for that mode must be present and the configuration for the other mode must be absent.  The USB host device configurations can be swapped out through the GUI or the command line.

Regardless of how the VM's USB config is swapped out, it is necessary to be able to identify the WS500 attached to the host machine, in its two different modes.  It is assumed the host machine is Linux.
If the WS500 is attached to the Linux host by USB, the following command should find it, regardless of its state:
`lsusb | grep '^Bus ... Device ...: ID 0483:.... STMicroelectronics '`
The output of the above will vary, depending on the WS500's mode; for example:
-> In normal mode:  'Bus 003 Device 043: ID 0483:5740 STMicroelectronics Virtual COM Port '
-> In DFU mode:     'Bus 003 Device 044: ID 0483:df11 STMicroelectronics STM Device in DFU Mode '
The "Bus" number and "Device" numbers may vary.  In the eaxmple above the "Bus" number is 003, and the "Device" numbers are 043 and 044.  The substring to the right of ' ID ', as in the examples above should remain constant for the given WS500 mode: "0483" is known to virt-manager as the 'vendor id', and "5740" and "df11" are known as the 'product id'.

To identify the USB device in the virt-manager GUI, the "Bus" and "Device" numbers, along with the "STMicroelectronics ..." string can be used.

Example virt-manager USB host device XML, taken from GUI, for WS500 in normal mode:
<hostdev mode="subsystem" type="usb" managed="yes">
  <source>
    <vendor id="0x0483"/>
    <product id="0x5740"/>
    <address bus="3" device="43"/>
  </source>
  <alias name="hostdev0"/>
  <address type="usb" bus="0" port="4"/>
</hostdev>
What's key to identify the WS500 is to look under '<hostdev mode="subsystem" type="usb" managed="yes">' for "/source/vendor/[@id="0x0483"]'.
Then, within the "/source", the "/product/@id" value will indicate the mode of the WS500 for which the VM's USB host device is configured:
-> "/product/@id" = "0x5740":   VM configured for WS500 in normal mode
-> "/product/@id" = "0xdf11":   VM configured for WS500 in DFU mode

From the command line of the VM's host machine, the VM's config can be output as XML by:
`virsh dumpxml <name of VM>`
The XML would contain an element similar to the above XML snippet taken from the virt-manager GUI.
To see the XML element containing just the USB device for the WS500:
`virsh dumpxml <name of VM> | xmllint --xpath "//domain/devices/hostdev[@mode='subsystem' and @type='usb' and @managed='yes' and source/vendor[@id='0x0483']]" -`
To get just the 'product id' value from it, which will indacate the WS500 mode the VM is configured for:
`virsh dumpxml <name of VM> | xmllint --xpath "string(//domain/devices/hostdev[@mode='subsystem' and @type='usb' and @managed='yes' and source/vendor[@id='0x0483']]/source/product/@id)" -`

The VM's device configuration can be changed with the `virsh` command: add a configuration with the 'attach-device' argument, and remove a configuration with the 'detach-device' argument.  The device configuration can be changed while the VM is running, using the '--live' option.

So it's possible to write a script to run on the VM host which will determine what mode the WS500 is in, using the `lsusb` command, and adapt the USB host device configuration of the VM accordingly, using the `virsh` command.

