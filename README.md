# Redragon Tophis G821 Linux Mode Investigation

## Goal

Document and test the G821 mode that exposes analog trigger values to Linux. The controller has keyboard, digital-trigger gamepad, D-input, and Apple-identified wired HID gamepad modes. The four-quadrant blue ring around the Home button showed the two right LEDs in keyboard mode and the two left LEDs in the D-input mode.

## Confirmed Device

- The receiver is USB vendor/product ID `2563:0575`.
- The user confirmed this is the G821 receiver, including when the gamepad is turned off.
- `lsusb` labels it `ShenZhen ShanWan Technology Co., Ltd. ZD-V+ Wired Gaming Controller`.
- The USB descriptors identify the manufacturer as `HORI CO.,LTD.` and the product as `Gamepad`.
- The Linux kernel has a USB quirk entry for `2563:0575`, described as `ShanWan Wireless Gamepad`, with `USB_QUIRK_WINDOWS_CONFIG_REQ_SIZE` in [`drivers/usb/core/quirks.c`](https://github.com/torvalds/linux/blob/master/drivers/usb/core/quirks.c).
- The receiver exposes one HID 1.11 interface with interrupt endpoints `0x82` (IN) and `0x02` (OUT), each with 64-byte maximum packets.
- The HID report descriptor is 144 bytes. It was retrieved with `usbhid-dump` while the gamepad was off.
- After powering on the gamepad in keyboard mode, `lsusb` reports a device with ID `0283:0001`, product name `Controller`.
- Holding Home for 3 seconds selects another mode, observed as USB ID `2564:0578`, product name `Gamepad`.
- The user reports that powering on with the dongle by pressing Home also selects `2564:0578`; its triggers are digital in evdev. The manual's claim that normal PC dongle startup enters X-input is not confirmed on this unit.
- Holding Start + Select while the controller is on changes the dongle's USB identity to `4c4a:2342 BR23UBOOT1.00`.
- With the dongle removed and USB cable connected, holding D-pad Up and pressing Home selects USB ID `05ac:033e`, reported as `Apple, Inc. Gamepad`.
- The different IDs were observed with the gamepad off (`2563:0575`) and powered on in keyboard mode (`0283:0001`). A live kernel log of the transition has not yet been captured, so the exact disconnect/reconnect sequence remains unverified.

## HID Report Descriptor

The descriptor declares one consumer-control input and three vendor-defined top-level collections. The `0xFFF0` collection switches to usage page `0xFF00` for its report fields. Each vendor report has a 63-byte payload; with the report ID, the report occupies 64 bytes on the wire.

| Usage page | Report ID | Direction | Usage | Payload |
| --- | --- | --- | --- | --- |
| Consumer `0x000C` | `0xDD` | Input | Consumer control | 2 data bits, remaining bits constant |
| Vendor `0xFF32` | `0x41` | Input | `0x75` | 63 bytes |
| Vendor `0xFF32` | `0x41` | Output | `0x76` | 63 bytes |
| Vendor `0xFF58` | `0x49` | Input | `0x01` | 63 bytes |
| Vendor `0xFF58` | `0x4F` | Output | `0x01` | 63 bytes |
| Vendor `0xFF00` (collection `0xFFF0`) | `0x10` | Input | `0x00` | 63 bytes |
| Vendor `0xFF00` (collection `0xFFF0`) | `0x0F` | Output | `0x22` | 63 bytes |

The vendor-defined reports have no semantic descriptions in the descriptor. Their presence shows that the interface supports output reports, but does not reveal whether any report selects a mode or what payload it would require. The descriptor does not describe a standard Xbox/XInput report format.

## Powered-On Keyboard Mode (`0283:0001`)

The user supplied a HID report descriptor for the device seen after powering on in keyboard mode. It contains:

- Report ID `0x03`: an 8-bit keyboard modifier bitmap and six keyboard key slots.
- Report ID `0x09`: a mouse-style report with five buttons, 16-bit X/Y axes, and an 8-bit wheel value.
- Vendor collection usage `0xFFF0`, with report fields on usage page `0xFF00`: report ID `0x10` for 63 bytes of input data and report ID `0x0F` for 63 bytes of output data.

The vendor output report `0x0F` is a possible protocol surface to investigate, but its purpose and payload are unknown. This descriptor does not expose analog gamepad trigger axes in the observed keyboard mode.

Powering on with the dongle while holding D-pad Up and pressing Home was also observed to leave the device at `0283:0001` with this same keyboard/mouse descriptor; the right-side ring LED blinks. This is consistent with the manual's LED1 calibration-start state, not evidence of a separate analog gamepad USB profile. The manual says to press A after this state to enter calibration.

## Home-Hold Gamepad Mode (`2564:0578`)

Holding Home for 3 seconds selects this mode; the two left LEDs in the blue Home-button ring are lit. The user supplied descriptors for two HID interfaces:

- Interface `001:010:000` identifies as a Generic Desktop Game Pad. Its input report contains 13 buttons, a hat switch, 8-bit `X`, `Y`, `Z`, and `Rz` axes, and four vendor-defined 16-bit values (usages `0x2C` through `0x2F`) with a logical range of `0` to `1023`.
- Interface `001:010:001` contains the same keyboard/mouse-style reports and vendor reports seen in keyboard mode: keyboard report ID `0x03`, mouse report ID `0x09`, vendor input report ID `0x10`, and vendor output report ID `0x0F`.

The gamepad interface has standard axes and extra 16-bit values, but the descriptor alone does not identify any field as a trigger. The vendor-defined values are not mapped to named controls by this descriptor.

An `evtest` test in this mode showed the left trigger as `EV_KEY` `BTN_TL` (code 310) and the right trigger as `EV_KEY` `BTN_TR` (code 311), each emitting only press (`1`) and release (`0`) events. Linux therefore exposes both triggers as digital buttons in this mode; the supplied trace contains no analog trigger `EV_ABS` events.

## Dongle Bootloader (`4c4a:2342`)

Holding Start + Select while the controller is on makes the dongle enumerate as `4c4a:2342 BR23UBOOT1.00`. This is a bootloader/update identity, not a gamepad input mode and not a way to enable analog triggers. No G821-specific bootloader exit or recovery procedure has been verified. Do not flash firmware or send guessed USB commands; first try disconnecting the dongle and reconnecting it without holding Start + Select, then check whether its normal USB identity returns. If it remains in bootloader mode, obtain the correct G821 firmware/recovery instructions from Redragon support before proceeding.

## Wired Apple-Identified Gamepad Mode (`05ac:033e`)

The user confirmed this working sequence: remove the 2.4 GHz dongle, connect the USB cable, hold D-pad Up, then press Home. The device enumerates as `05ac:033e Apple, Inc. Gamepad`.

The supplied HID descriptor defines a Game Pad input report (report ID `0x01`) containing 15 buttons, a hat switch, 8-bit X/Y/Z/Rz axes, and two 8-bit Simulation Controls values: `Brake` (usage `0xC5`) and `Accelerator` (usage `0xC4`), each with logical range 0-255. `evtest` confirms these are the analog triggers in this mode:

- Left trigger: `BTN_TL2` (code 312) plus `ABS_BRAKE` (code 10), varying from 0 to 255.
- Right trigger: `BTN_TR2` (code 313) plus `ABS_GAS` (code 9), varying from 0 to 255.

Each trigger emits both a digital button state and a continuously varying analog axis. Linux receives the full analog range in this wired mode.

The report descriptor also defines an 8-bit output report (`0x02`) with four LED usages and a second mouse-style collection (report ID `0x05`); neither changes the gamepad trigger interpretation.

Wireless handoff was tested: after entering this analog mode by cable, connecting the dongle, and removing the cable, the controller reconnected through the dongle in keyboard mode (`0283:0001`). The Apple analog-trigger profile did not persist over the wireless handoff.

## Wired Mode and Trigger Calibration

The [G821 manual linked from the product page](https://downloads.electronicrebellion.com/manuals/ERRXA710/G821%20manual.pdf) is titled `IG01C 2.4G Manual`. It says to connect the controller to a PC by USB cable and press Home after plugging it in; otherwise the cable only charges the controller. It also warns that if the dongle and cable are connected at the same time, the cable only charges. The user's successful wired mode requires removing the dongle, connecting USB-C to the PC, holding D-pad Up, then pressing Home.

The manual does not name an Apple mode; the Apple identification is confirmed by the observed USB ID `05ac:033e`. The manual's listed compatibility is PC, PS3, Android, and Nintendo Switch. It documents two trigger-calibration entry procedures. The first is to power off, hold D-pad Up, then press Home; LED1 lights, after which A enters calibration and LED2 lights. The user's dongle-connected Up+Home test produced `0283:0001` and a blinking right LED, consistent with the manual's LED1 calibration-start state, but did not change the HID descriptor to a gamepad profile. With the dongle removed and cable connected, Up+Home instead produced the Apple-identified gamepad profile:

1. With the controller off, hold D-pad Up and press Home; LED1 lights. Press A to enter calibration; LED2 lights. Rotate both sticks through three full circles, fully press each trigger three times with normal force, then press A to confirm and exit.
2. The manual also gives an alternative calibration entry: hold L1 + Select while connecting the controller to the computer; LED1 flashes slowly. Press A to enter calibration; LED2 flashes slowly. Repeat the stick and trigger movements above, then press A to finish.

The manual separately describes wireless PC X-input mode (LED1 and LED2 lit) and switching to D-input by holding Home for 2 seconds. However, the user's observed Home startup with the dongle reaches `2564:0578`, where triggers are digital, so the manual's X-input startup instructions do not match the observed result on this unit. The manual does not explicitly identify the wired mode as Apple-compatible; the observed USB ID does. It also does not say whether entering `05ac:033e` automatically completes calibration; its calibration instructions continue with A and the stick/trigger exercise.

The manual documents four vibration levels: off, weak, medium, and strong. Hold Turbo (the button below Home) and push the right stick up to increase vibration. Push the right stick down to decrease it; the user confirmed repeated downward adjustments reduce vibration and eventually turn it off.

## Reported Behavior

- The keyboard mode (`0283:0001`) and Home-hold HID gamepad mode (`2564:0578`) have supplied descriptors. A gamepad mode with analog sticks but digital triggers was also reported; whether that is the Home-hold mode is not yet confirmed.
- The Home-hold mode exposes the triggers as digital `BTN_TL` and `BTN_TR` events through evdev. The raw HID report has not yet been checked to see whether it contains additional analog data that the kernel does not map.
- The wired Apple-identified mode (`05ac:033e`) exposes analog triggers as `ABS_BRAKE` and `ABS_GAS`, each with a 0-255 range.
- After switching from wired Apple mode to the dongle and removing the cable, the controller reconnects as keyboard mode (`0283:0001`), not as the analog gamepad profile.
- The receiver has been observed as `2563:0575` while the gamepad is off and `0283:0001` when powered on in keyboard mode. Mode switching is reported to cause the receiver to disconnect and reconnect as a device.

## Unknowns / Next Evidence

The wired Apple-identified mode selection and failed wireless handoff are confirmed; remaining wireless investigation is:

- What USB ID and HID descriptor appear in the reported wireless analog-stick/digital-trigger mode, and whether it is the `2564:0578` Home-hold mode.
- What exact sequence enters the manual's wireless PC X-input mode on this controller; Home selects digital-trigger `2564:0578`, while dongle-connected Up+Home leaves `0283:0001` in its keyboard/mouse HID profile.
- How to exit the dongle's `4c4a:2342 BR23UBOOT1.00` bootloader state if a normal power cycle does not restore the receiver.
- The meanings and valid payloads of the vendor-defined output reports.

The tested handoff returns to keyboard mode. It is not yet known whether another wireless PC X-input mode exposes analog trigger axes; the manual's generic X-input startup instructions have not matched the modes observed so far. Further protocol investigation would require capturing and comparing wireless reports or identifying a controller-side mode switch. For the currently identified powered-off receiver, retrieve the descriptor with:

```sh
sudo usbhid-dump -d 2563:0575 -e descriptor
```

Include the exact buttons or switch used. Do not send guessed output-report payloads: the descriptor gives report sizes and IDs, not command meanings.
