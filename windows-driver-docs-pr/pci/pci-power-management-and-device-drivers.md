---
title: PCI Power Management and Device Drivers
author: windows-driver-content
description: This article clarifies some confusion that vendors have experienced about how hardware that complies with PCI Power Management (PCI-PM) interacts with device drivers in the operating system and about how PCI-PM integrates with ACPI.
MSHAttr:
- 'PreferredSiteName:MSDN'
- 'PreferredLib:/library/windows/hardware'
ms.assetid: BA6792EE-CAD8-4C9E-AAA6-D1D8799F50C3
---

# PCI Power Management and Device Drivers


**Updated**

-   December 4, 2001

This article clarifies some confusion that vendors have experienced about how hardware that complies with PCI Power Management (PCI-PM) interacts with device drivers in the operating system and about how PCI-PM integrates with ACPI. For more information, see <http://www.acpi.info/>

## Device drivers and PCI power management


This discussion assumes that you are familiar with how Windows Driver Model (WDM) drivers handle power management events, as described in the current Windows DDK. In general, the responsibilities for device drivers are as follows:

-   **Bus drivers**: Bus drivers are responsible for enumerating, configuring, and controlling devices. For PCI-PM, the PCI driver is responsible for reading the PCI-PM registers to determine the capabilities of the hardware. When POWER IRPs request power state changes, the PCI driver writes to the PCI power management registers to set the hardware to different Dx states.

    When a device is enabled for wake-up, the PCI driver writes to PCI-PM registers to enable the device to fire PME (ACPI will also take an action, see the next section). Finally, when ACPI determines that the PCI bus is waking the system, the PCI driver scans PCI configuration space looking for which device is asserting PME, disables PME in that device, and notifies the driver for that device.

-   **Device driver**: The specific driver for the device is responsible for saving and restoring device context, and requesting power state changes as the policy owner for the device. When the device driver receives a POWER IRP requesting a lower device power state change, the device driver is responsible for saving any proprietary device context needed to later turn on the device. In some cases, there may be nothing to save.

PCI-PM registers are strictly the domain of the PCI driver--the IHV's device driver does not need to access any of these registers. Doing so would cause the system to not work reliably. The device driver's responsibility is to perform only proprietary actions.

## Integrating ACPI and PCI PM


Some devices, particularly motherboard video devices in portables, may require both PCI Power Management as well as ACPI Source Language Assembler (ASL) to completely power manage the device. The PCI Power Management registers would control the internal state of a device, such as internal clocks and power planes. ASL would control the external state, such as external clocks and power planes, or in the case of video controllers, ASL would control the video backlights. Note that ASL and PCI-PM can only be combined on motherboard devices.

The OnNow architecture is a layered architecture, handling the integration of the device driver, PCI driver, and ACPI driver (and ASL) naturally. The following scenarios show the order in which drivers are called to handle these devices.

**Note**  For the above scenarios to work as described, a WDM driver must forward POWER IRPs correctly as described in the current version of the Microsoft Windows DDK.

 

## Scenario 1: Turning off a device


1.  **Device driver**: Saves proprietary device state.
2.  **PCI driver**: Saves Plug and Play configuration, disables the device (interrupts and BARs), and puts the device in D3 using PCI-PM registers.
3.  **ACPI driver**: Runs ASL code (\_PS3 and \_OFF for power resources no longer in use) to control the state external to the chip.

## Scenario 2: PCI power management and device drivers


1.  **ACPI driver**: Runs ASL code (\_PS0 and \_ON for any OnNow required power resources) to control the state external to the chip.
2.  **PCI driver**: Puts the device in D0 using PCI-PM registers and restores Plug and Play configuration (interrupts and BARs--these might be different from what the device was previously on).
3.  **Device driver**: Restores proprietary context in the device.

## Scenario 3: Enabling wake-up


1.  **Device driver**: Sets proprietary registers in the chip to enable wake-up. For example, in pattern matching network wake-up, this is when the patterns would be programmed into the adapter.
2.  **PCI driver**: Sets the wake-up enable bits in the PCI PM registers to allow the device to assert PME.
3.  **ACPI driver**: Enables the GPE in the chip set associated with PME (as described by the \_PRW object listed under the root PCI bus).

## Scenario 4: Wake-up


1.  **ACPI driver**: Wakes and scans the GPE status bits for wake-up events, disabling GPEs for set GPE status bits, and running any \_Lxx or \_Exx methods associated with set GPE bits. In response to a wake-up notification on the PCI bus, the ACPI driver will complete the PCI driver's WAIT\_WAKE IRP to notify the PCI driver that it is waking the system.
2.  **PCI driver**: Scans configuration space looking for any devices with a set PME status bit. For each device, it disables PME and completes the WAIT\_WAKE IRP for that device to inform the driver that it is asserting wake-up. The PCI driver stops scanning for wake devices when it has made a complete pass through all PCI devices having not found any asserting PME and when PME stops being asserted.
3.  **Device driver**: Requests the device be put in D0 (see scenario 2) and sets any proprietary registers in the chip required to handle the wake-up event.

## Call to action on PCI power management and device drivers:


-   Integrate ACPI and PCI-PM capabilities into your devices as described in this article.
-   The PCI Power Management specification is available at <http://www.pcisig.com>. This link leaves the Microsoft.com site.
-   ACPI 1.0b Specification available at <http://www.acpi.info/>. This link leaves the Microsoft.com site.
-   For the current ASL Assembler/Compiler and current Device Class Power Management Reference specifications, see http://msdn.microsoft.com/windows/hardware/gg463220.aspx.

 

 


--------------------
[Send comments about this topic to Microsoft](mailto:wsddocfb@microsoft.com?subject=Documentation%20feedback%20%5Bpci\buses%5D:%20PCI%20Power%20Management%20and%20Device%20Drivers%20%20RELEASE:%20%285/19/2016%29&body=%0A%0APRIVACY%20STATEMENT%0A%0AWe%20use%20your%20feedback%20to%20improve%20the%20documentation.%20We%20don't%20use%20your%20email%20address%20for%20any%20other%20purpose,%20and%20we'll%20remove%20your%20email%20address%20from%20our%20system%20after%20the%20issue%20that%20you're%20reporting%20is%20fixed.%20While%20we're%20working%20to%20fix%20this%20issue,%20we%20might%20send%20you%20an%20email%20message%20to%20ask%20for%20more%20info.%20Later,%20we%20might%20also%20send%20you%20an%20email%20message%20to%20let%20you%20know%20that%20we've%20addressed%20your%20feedback.%0A%0AFor%20more%20info%20about%20Microsoft's%20privacy%20policy,%20see%20http://privacy.microsoft.com/default.aspx. "Send comments about this topic to Microsoft")

