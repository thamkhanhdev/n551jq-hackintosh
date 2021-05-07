# USBMapping

Refer to this link for more details: [USBMapping][intel_mapping]

USB mapping on Intel is super easy mainly because both the ACPI is sane and more tools available for the platform. For this guide we'll be using the [USBmap tool][usb_map] (opens new window) from CorpNewt.

***

## Step by step

- Open up USBmap.command and select D. Discover Ports:

- Plug all devices to all usb ports.

- Once you're done discovering your ports, select Press Q then [enter] to stop then head to P. Edit Plist & Create SSDT/Kext from the main menu.

- Select K. Build USBMap.kext and let it build our kext for us.

- Activate USBMap.kext in config.plist

[usb_map]:<https://github.com/corpnewt/USBMap/>
[intel_mapping]:<https://dortania.github.io/OpenCore-Post-Install/usb/intel-mapping/intel.html>

***
