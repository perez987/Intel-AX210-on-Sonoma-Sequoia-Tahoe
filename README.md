# Wi-Fi 6 Intel AX210 on macOS Sonoma, Sequoia and Tahoe

<p align="center">
<img src="img/Wi-Fi.png">
</p>

macOS Sonoma removed drivers for Broadcom Wi-Fi cards found in Mac models prior to 2017. One of the affected cards is the Fenvi T-919, widely used in Hacks. OCLP developers published a fix that allows these Wi-Fi to work in Sonoma and Sequoia, adding this feature to the root patches that OCLP can apply. In order to apply root patches, OCLP requires macOS to run with some relaxed security features: SecureBootModel disabled and SIP partially disabled. This still represents a certain loss of security in macOS, as some users have noted.

Here I propose a model of Intel Wi-Fi card that by default lacks support but can be used in macOS thanks to the work of the OpenIntelWireless site. This is the Intel AX210S PCIe WiFi 6E card. This card can work with regular macOS security conditions without needing to relax Apple Secure Boot or SIP. It may be interesting for those who have lost Broadcom Wi-Fi support in macOS Sonoma+ or for those who want to keep the security of their system without resorting to OCLP patches.

---

### Hardware

The card can be purchased in 2 different ways:

* fully assembled by _Ziyituod_ and others: WiFi 6E Intel AX210S PCIe
* in 2 parts, card itself (WiFi 6E Intel AX210 NGW Bluetooth Card for Laptop with M.2/NGFF Connector) and adapter (PCIE X1 to M.2/NGFF A+E Key Adapter for WiFi Bluetooth Module).

<table>
<tr>
<td>
  <img src="img/Card and adapter.png">
 </td>
<td>
  <img src="img/AX210 card.jpg">
 </td>
</tr>
</table>

---

### Revert OCLP patch and config.plist changes

If you have been using Fenvi or Broadcom Wi-Fi, you must revert all the settings related to config.plist and OCLP root patch.

In config.plist:

* disable kexts (`IOSkywalk.kext`, `IO80211FamilyLegacy.kext` and `AirPortBrcmNIC.kext`)
* disable `IOSkywalk.kext` blocking
* change `csr-active-config` to 00000000
* change `SecureBootModel` to a value other than Disabled.

From OpenCore-Patcher (OCLP) >> Post-Install Root Patch >> Revert Root Patches.

---

### Installing wifi module

The kexts are available on the OpenIntelWireless site. There are 2 ways to install Wi-Fi:

* `itlwm.kext`: uses `IOEthernetController` instead of `IO80211Family` so the connection spoofs as Ethernet even though it works as wifi. It does not use the macOS Wi-Fi menu, instead you have to use the HeliPort application. On Ventura you need itlwm v2.2.0. On Sonoma you need itlwm v2.3.0.
* `AirportItlwm.kext`: uses `IO80211Family` so it works like the rest of the system's Wi-Fi connections. It provides minimal Continuity features (Handoff and Universal Clipboard, not always available, no Airdrop) but cannot connect to hidden networks. No HeliPort app needed. On Ventura you need AirportItlwm v2.2.0. On Sonoma prior to 14.4 you need AirportItlwm v2.3.0 for Sonoma 14.0.

Both kexts should not be used at the same time, only one of them. I have tried both and they seem to have worked well. The card is well detected, as you can see in Hackintool.

<p align="left">
<img width="740" src="img/AX210 Hackintool.png">
</p>

**Note about macOS 14.4 Sonoma**: Apple has changed parts of the Wi-Fi stack. For this macOS you must get AirportItlwm v2.3.0 for Sonoma 14.4. itlwm.kext 2.3.0 + Heliport 1.5.0 work fine too.

**Note about macOS 15 Sequoia and macOS 26 Tahoe**: itlwm.kext 2.3.0 + Heliport 2.0.0 alpha work fine. AirportItlwm.kext 2.3.0 for 14.4 doesn't work, this kext must be updated by the dev.

All kexts are available in the [releases](https://github.com/OpenIntelWireless/itlwm/releases) page.

Heliport app is part of the OpenIntelWireless project, you can get it in the [releases](https://github.com/OpenIntelWireless/HeliPort/releases) page.

---

### Installing Bluetooth module

On Monterey and newer you have to install 3 extensions:

* `IntelBTPatcher.kext`, requires Lilu 1.6.2 or newer: fixes a bug in bluetoothd by correctly initializing the bluetooth module.
* `IntelBluetoothFirmware.kext`: load the firmware to the device and set the device name in USB Host Controller to Bluetooth USB Host Controller.
* `BlueToolFixup.kext` (available in Acidanthera's [BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) package): on macOS Monterey `IntelBluetoothInjector.kext`stopped working due to changes made by Apple to the Bluetooth stack.

`IntelBTPatcher.kext `and `IntelBluetoothFirmware.kext` are inside the `IntelBluetooth` package available in the [releases](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases) page.

### Instant wake after sleep

There are users, myself included, experience this behavior of waking up immediately after sleep when using the Bluetooth module on Intel cards together with OpenIntelWireless kexts.

I reported this as an issue to the developer almost long time ago ([Sleep/instant wake when using Bluetooth kexts](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/477)), but he didn't consider it a problem with the kexts.

There is a fix. Adding `SSDT-GPRW.aml` to the ACPI folder and config.plist, and adding the patch `Change GPRW to XPRW, needs SSDT-GPRW.aml` to ACPI >> Patch in config.plist, the PC properly enters sleep, but only wakes up from the power button; the ability to wake from the keyboard or mouse is lost. It's a drawback, certainly, but preferable to losing the sleep functionality.

#### SSDT-GPRW

```c++
DefinitionBlock ("", "SSDT", 2, "DRTNIA", "GPRW", 0x00000000)
{
    External (XPRW, MethodObj)    // 2 Arguments

    Method (GPRW, 2, NotSerialized)
    {
        If (_OSI ("Darwin"))
        {
            If ((0x6D == Arg0))
            {
                Return (Package (0x02)
                {
                    0x6D, 
                    Zero
                })
            }

            If ((0x0D == Arg0))
            {
                Return (Package (0x02)
                {
                    0x0D, 
                    Zero
                })
            }
        }

        Return (XPRW (Arg0, Arg1))
    }
}
```

#### Change GPRW to XPRW patch

```xml
	<key>ACPI</key>
	<dict>
		<key>Patch</key>		
                <array>
			<dict>
				<key>Base</key>
				<string></string>
				<key>BaseSkip</key>
				<integer>0</integer>
				<key>Comment</key>
				<string>Change GPRW to XPRW, needs SSDT-GPRW.aml</string>
				<key>Count</key>
				<integer>0</integer>
				<key>Enabled</key>
				<true/>
				<key>Find</key>
				<data>R1BSVwI=</data>
				<key>Limit</key>
				<integer>0</integer>
				<key>Mask</key>
				<data></data>
				<key>OemTableId</key>
				<data></data>
				<key>Replace</key>
				<data>WFBSVwI=</data>
				<key>ReplaceMask</key>
				<data></data>
				<key>Skip</key>
				<integer>0</integer>
				<key>TableLength</key>
				<integer>0</integer>
				<key>TableSignature</key>
				<data></data>
			</dict>
		</array>
	</dict>
```

You can read more deeply about this SSDT and patch at [0D/6D patch](https://github.com/jsassu20/OpenCore-HotPatching-Guide/tree/master/12-060D%20Patch/12-1-Common%20060D%20patch).

---

### Summary

This hardware is a valid option for those who do not have Broadcom Wi-Fi in macOS Sonoma+ or do not want to apply OCLP root patches. It is not expensive and easy to install. As a main drawback, the features of the Apple ecosystem are lost (all with `itlwm.kext` and most with `AirportItlwm.kext`). Airdrop does not work in any way and this is the feature that I miss the most with respect to the Fenvi.

---

### Note about Hackintool

Very small issue.

Hackintool by [headkaze](https://github.com/benbaker76/Hackintool) has an Extensions tab where it reports the installed kexts. In the list of kexts, `itlwm` is seen when we use `itlwm` and also when we use `AirportItlwm`. This is because, in the file `/Applications/Hackintool.app/Contents/Resources/Kexts/kexts.plist`, both kexts have `Name=itlwm`.

This is correct since the project is called `itlwm`. But I prefer Hackintool to show `AirportItlwm` when this is the active kext. This can be solved by modifying the value of the Name property in the `AirportItlwm` key inside the file.

By default it is like this:

<p align="left">
<img width="660" src="img/itlwm name.png">
</p>
<p align="left">
<img width="440" src="img/Hackintool itlwm.png">
</p>

But, changing to `Name=AirportItlwm`, Hackintool displays the active kext in a way I like better.

<p align="left">
<img width="660" src="img/AirportItlwm name.png">
</p>
<p align="left">
<img width="440" src="img/Hackintool AirportItlwm.png">
</p>
