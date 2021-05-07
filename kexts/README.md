# Step by step patching all kexts

These patches applied for Mac OS Big Sur (13.1) and Open Core 0.6.8. Using other OS versions may need other patches to work properly.

## 1. Audio

This device runs Realtek ALC668, layout-id 29.

### 1.1. Patch DSDT and SSDT

#### 1.1.1. Fix IRQ conflicts

- Open DSDT.dsl by MaciASL (Mac) or Visual Code (Window).

- Search the keyword `PNP0000` to go to Device PIC (it usually names IPIC or PIC).

- In the _Resource Template_ of the object _CRS, deletes two last declarations:

  ```sh
    Device (IPIC)
    {
        Name (_HID, EisaId ("PNP0000"))
        Name (_CRS, ResourceTemplate ()
        {
            IO (Decode16,
                0x0020, // Range Minimum
                0x0020, // Range Maximum
                0x01, // Alignment
                0x02, // Length
                )
            ...
            IO (Decode16,
                0x04D0, // Range Minimum
                0x04D0, // Range Maximum
                0x01, // Alignment
                0x02, // Length
                )
            IRQNoFlags ()
                {2}
        })
    }
  ```

- Repeat with IRQ0 and IRQ8 corresponding to each device RTC (`PNP0B00`) and TIMR (`PNP0100`).

```sh
    Device (RTC)
    {
        Name (_HID, EisaId ("PNP0B00"))
        Name (_CRS, ResourceTemplate ()
        {
            IO (Decode16,
                0x0070, // Range Minimum
                0x0070, // Range Maximum
                0x01, // Alignment
                0x08, // Length
                )
            IRQNoFlags ()
                {8}
        })
    }
    ...
    Device (TIMR)
    {
        Name (_HID, EisaId ("PNP0100"))
        Name (_CRS, ResourceTemplate ()
        {
            IO (Decode16,
                0x0040, // Range Minimum
                0x0040, // Range Maximum
                0x01, // Alignment
                0x04, // Length
                )
            IO (Decode16,
                0x0050, // Range Minimum
                0x0050, // Range Maximum
                0x10, // Alignment
                0x04, // Length
                )
            IRQNoFlags ()
                {0}
        })
    }
```

- Compile DSDT to check whether it has error. Save file.

> Note: _With_ `RTC`, _we need to change the value_ `Length` _in_ `IO` _from_ `0x08` _to_ `0x02`.

#### 1.1.2. Add 4 IRQs to HPET

- Search the keyword `PNP0103` to jump to `Device HPET`.

- Usually, we encounter the case `Resource Settings` of `HPET` is stored in a separate object (for example `BUF0` or `ATT3`). `_CRS` is a method that the result is a returned value `BUF0`.

- For example:

```sh
    Device (HPET)
    {
        Name (_HID, EisaId ("PNP0103"))
        Name (_UID, Zero)
        Name (BUF0, ResourceTemplate ()
        {
            Memory32Fixed (ReadWrite,
                0xFED00000,         // Address Base
                0x00000400,         // Address Length
                )
        })
        Method (_STA, 0, NotSerialized)
        {
            If (LGreaterEqual (OSYS, 0x07D1))
            {
                If (HPAE)
                {
                    Return (0x0F)
                }
            }
            Else
            {
                If (HPAE)
                {
                    Return (0x0B)
                }
            }
            Return (Zero)
        }
        Method (_CRS, 0, Serialized)
        {
            If (HPAE)
            {
                CreateDWordField (BUF0, 0x04, HPT0)
                If (LEqual (HPAS, One))
                {
                    Store (0xFED01000, HPT0)
                }
                If (LEqual (HPAS, 0x02))
                {
                    Store (0xFED02000, HPT0)
                }
                If (LEqual (HPAS, 0x03))
                {
                    Store (0xFED03000, HPT0)
                }
            }
            Return (BUF0)
        }
    }
```

- Or this case:

```sh
    Device (HPET)
    {
        Name (_HID, EisaId ("PNP0103"))
        Name (ATT3, ResourceTemplate ()
        {
            Memory32Fixed (ReadOnly,
                0xFED00000,         // Address Base
                0x00000400,         // Address Length
                )
        })
        Name (ATT4, ResourceTemplate ()
        {
        })
        Method (_STA, 0, NotSerialized)
        {
            If (LGreaterEqual (OSFX, 0x03))
            {
                If (HPTF)
                {
                    Return (0x0F)
                }
                Else
                {
                    Return (0x00)
                }
            }
            Else
            {
                Return (0x00)
            }
        }
        Method (_CRS, 0, NotSerialized)
        {
            If (LGreaterEqual (OSFX, 0x03))
            {
                If (HPTF)
                {
                    Return (ATT3)
                }
                Else
                {
                    Return (ATT4)
                }
            }
            Else
            {
                Return (ATT4)
            }
        }
    }
```

- We will fix all errors in two objects `_STA` and `_CRS` of `HPET`. So the returned value of `_STA` is _0x0F_, and the returned value of `_CRS' is a`Resource Template`, they are

  - _Memory32Fixed (ReadWrite, 0xFED00000, 0x00000400)_ (the active memory of `HPET`, it's same with every computers).
  - _IRQNoFlags () {2, 8, 11, 15}_ (4 IRQs have to be added).

- So with every DSDTs, we only need to delete all contents of `Device HPET` and replace by:

```sh
    Device (HPET)
    {
        Name (_HID, EisaId ("PNP0103"))
        Name (_STA, 0x0F)
        Name (_CRS, ResourceTemplate ()
        {
            Memory32Fixed (ReadWrite,
                0xFED00000,         // Address Base
                0x00000400,         // Address Length
                )
            IRQNoFlags ()
                {2, 8, 11, 15}
        })
    }
```

- Finally compile DSDT again and check the result.

#### 1.1.3. Add layout id for ALC668 device

- Search `HDEF` keyword, you will see the result like this:

```sh
    Device (HDEF)
    {
        Name (_ADR, 0x001B0000)  // _ADR: Address
        OperationRegion (HDAR, PCI_Config, 0x4C, 0x10)
        Field (HDAR, WordAcc, NoLock, Preserve)
        {
            DCKA,   1,
            Offset (0x01),
            DCKM,   1,
                ,   6,
            DCKS,   1,
            Offset (0x08),
            Offset (0x09),
            PMEE,   1,
                ,   6,
            PMES,   1
        }
    ...
    }
```

- Now add a new method to this device:

```sh
    Method (_DSM, 4, NotSerialized)
    {
        If (LEqual (Arg2, Zero)) { Return (Buffer() { 0x03 } ) }
        Return (Package()
        {
            "layout-id", Buffer() { 29, 0x00, 0x00, 0x00 },
            "hda-gfx", Buffer() { "onboard-1" },
            "PinConfigurations", Buffer() { },
            //"MaximumBootBeepVolume", 77,
        })
    }
```

> Note: The layout-id shall be changed for each device. In this case Id 29 is compatible. They can be 3, 20, 27, 28, 29, etc...

- Add layout-id in config.plist

```sh
    <key>DeviceProperties</key>
    <dict>
        <key>Add</key>
        <dict>
            <key>PciRoot(0x0)/Pci(0x1B,0x0)</key>
            <dict>
                <key>layout-id</key>
                <data>HQAAAA==</data>
            </dict>
            ...
```

> Note: `HQAAAA==` is base64 format, need converting its from Hex to base64.

#### 1.1.4. Configure DSDT file

After patching DSDT file, need compiling its to .aml format and put it to EFI/OC/ACPI folder. Then active its in config.plist by these instructions.

```sh
    <key>ACPI</key>
    <dict>
        <key>Add</key>
        <array>
            <dict>
                <key>Comment</key>
                <string>DSDT.aml</string>
                <key>Enabled</key>
                <true/>
                <key>Path</key>
                <string>DSDT.aml</string>
            </dict>
            ...
```

#### 1.1.4. Install kexts

- Using `Kext Utility` or `Kext Wizard` to add kexts (_AppleALC, AppleHDA, Lilu_) to S/L/E folder.

- AppleALC and Lilu also need adding to EFI/OC/Kexts folder

  - Active kexts by these instructions:

```sh
    <key>Kernel</key>
    <dict>
        <key>Add</key>
        <array>
            <dict>
                <key>BundlePath</key>
                <string>AppleALC.kext</string>
                <key>Comment</key>
                <string></string>
                <key>Enabled</key>
                <true/>
                <key>ExecutablePath</key>
                <string>Contents/MacOS/AppleALC</string>
                <key>Arch</key>
                <string>Any</string>
                <key>MaxKernel</key>
                <string></string>
                <key>MinKernel</key>
                <string></string>
                <key>PlistPath</key>
                <string>Contents/Info.plist</string>
            </dict>
            <dict>
                <key>BundlePath</key>
                <string>Lilu.kext</string>
                <key>Comment</key>
                <string></string>
                <key>Enabled</key>
                <true/>
                <key>ExecutablePath</key>
                <string>Contents/MacOS/Lilu</string>
                <key>Arch</key>
                <string>Any</string>
                <key>MaxKernel</key>
                <string></string>
                <key>MinKernel</key>
                <string></string>
                <key>PlistPath</key>
                <string>Contents/Info.plist</string>
            </dict>
        .....
    <key>NVRAM</key>
    <dict>
        <key>Add</key>
        <dict>
            <key>7C436110-AB2A-4BBB-A880-FE41995C9F82</key>
            <dict>
                <key>SystemAudioVolume</key>
                <data>Rg==</data>
                <key>boot-args</key>
                <string>-v keepsyms=1 debug=0x100 -lilubeta -alcbeta</string>
                <key>csr-active-config</key>
        .....
```

- Active LILU and ALC kexts by adding arguments `-lilubeta -alcbeta` to `boot-args`

## 2. USB Mapping

Refer to this link for more details: [USBMapping][intel_mapping]

USB mapping on Intel is super easy mainly because both the ACPI is same and more tools available for the platform.
For this guide we'll be using the [USBmap tool][usb_map] (opens new window) from CorpNewt.

- Open up USBmap.command and select D. Discover Ports:

- Plug all devices to all usb ports.

- Once you're done discovering your ports, select Press Q then [enter] to stop then head to P. Edit Plist & Create SSDT/Kext from the main menu.

- Select K. Build USBMap.kext and let it build our kext for us.

- Activate USBMap.kext in config.plist

***

[usb_map]:<https://github.com/corpnewt/USBMap/>
[intel_mapping]:<https://dortania.github.io/OpenCore-Post-Install/usb/intel-mapping/intel.html>
