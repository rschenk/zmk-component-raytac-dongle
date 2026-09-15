# ZMK Raytac USB Dongle Module

This repository adds ZMK board definitions named `mdbt50q_rx` for the [Raytac MDBT50Q-RX](https://www.raytac.com/product/ins.php?index_id=89) USB stick and `mdbt50q_cx_40` for the [Raytac MDBT50Q-CX-40](https://www.raytac.com/product/ins.php?index_id=156) USB stick, with the intention of using them as [ZMK keyboard dongles](https://zmk.dev/docs/development/hardware-integration/dongle).

![IMG_1775](./docs/images/IMG_1775.jpeg)

## Caveat: Buy The RX With A UF2 Bootloader Installed

The stock MDBT50Q-RX does not come with the UF2 bootloader installed—that's the bit of code that allows it to mount itself as a USB drive for flashing.

But! Some versions of the RX dongle do exist with the UF2 bootloader pre-installed, and that's what you want to buy: [Adafruit (USA)](https://www.adafruit.com/product/5199) and [Pi Hut (UK)](https://thepihut.com/products/nrf52840-usb-key-with-tinyuf2-bootloader-bluetooth-low-energy-mdbt50q-rx) sell them, and probably others if you search around. I used the one from Adafruit.

> **Note:** The MDBT50Q-CX-40 does not have a UF2 bootloader option. It ships with Nordic's built-in Open bootloader and is flashed via `nrfutil` over USB serial instead. See [Flashing The CX-40](#flashing-the-cx-40-built-in-bootloader) below.

### Update The Bootloader

I upgraded the version of the bootloader on my Raytac dongle and you should to. [Someone on Reddit](https://www.reddit.com/r/ErgoMechKeyboards/comments/1k4ejtx/raytac_dongle/norwr6p/) confirmed that you need to update to get it working. So go ahead and update, it's easy.

I used these [instructions](https://learn.adafruit.com/introducing-the-adafruit-nrf52840-feather/update-bootloader-use-uf2) to update the bootloader to version [0.9.2](https://github.com/adafruit/Adafruit_nRF52_Bootloader/releases/tag/0.9.2). Specifically I used [this exact one](https://github.com/adafruit/Adafruit_nRF52_Bootloader/releases/download/0.9.2/update-raytac_mdbt50q_rx_bootloader-0.9.2_nosd.uf2).

Once you update the bootloader, hold the button down while plugging it in, navigate to the drive in your file browser, and open `INFO_UF2.TXT`. You want to see this:

```
UF2 Bootloader 0.9.2 lib/nrfx (v2.0.0) lib/tinyusb (0.12.0-145-g9775e7691) lib/uf2 (remotes/origin/configupdate-9-gadbb8c7) # <-- Important, v0.9.2 from above
...
SoftDevice: S140 6.1.1 # <-- Important, v6.x of SoftDevice

```

## ZMK Versioning

ZMK v0.4 introduced breaking changes which _affect this module and you as a user._

You must [pin your ZMK version](https://zmk.dev/blog/2025/06/20/pinned-zmk) and pin this module to match.

The remainder of this readme will detail how to use the module in ZMK v0.4. 

If you are using an earlier version of ZMK, please refer to an [earlier version of this readme](https://github.com/rschenk/zmk-component-raytac-dongle/tree/v0.3).

## Usage for ZMK 0.4

Add the following entries to `remotes` and `projects` in `config/west.yml`

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: rschenk
      url-base: https://github.com/rschenk
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.4 # <-- ZMK pinned to v0.4
      import: app/west.yml
    - name: zmk-component-raytac-dongle
      remote: rschenk
      revision: v0.4 # <-- This module pinned to the same version as ZMK
  self:
    path: config
```

## Configuring Your Dongle

Follow the setup steps in the [ZMK Keyboard Dongle docs](https://zmk.dev/docs/development/hardware-integration/dongle) to configure your keyboard and dongle. When you get to the [Building the Firmware](https://zmk.dev/docs/development/hardware-integration/dongle#building-the-firmware) step, you will use `board: mdbt50q_rx` (or `board: mdbt50q_cx_40` for the CX-40 dongle) in your `build.yml` file like so:

```yaml
include:
  # Config settings for the dongle (use mdbt50q_rx or mdbt50q_cx_40)
  - board: mdbt50q_rx
    shield: my_keyboard_dongle
    
  - board: mdbt50q_rx
    shield: settings_reset

  # Whatever your keyboard uses...
  - board: nice_nano
    shield: my_keyboard
```

## Flashing The Raytac

### Flashing The RX (UF2 Bootloader)

Entering the bootloader mode is a bit annoying. Unplug the dongle, then hold the button down while plugging it back in. 

For some reason you *can't* double-click the reset button like on many boards.

### Flashing The CX-40 (Built-In Bootloader)

The CX-40 does **not** have a UF2 bootloader—there is no drag-and-drop USB drive flashing. Instead, it ships with [Nordic's Open Bootloader](https://docs.nordicsemi.com/bundle/sdk_nrf5_v17.1.0/page/sdk_app_serial_dfu_bootloader.html) (from the nRF5 SDK), which uses DFU (Device Firmware Update) over USB serial.

For full reference, see the [Zephyr board documentation for the MDBT50Q-CX-40](https://docs.zephyrproject.org/latest/boards/raytac/mdbt50q_cx_40_dongle/doc/index.html#option-1-using-the-built-in-bootloader-only).

#### 1. Install nrfutil

Install Nordic's `nrfutil` CLI via Homebrew, then add the `nrf5sdk-tools` subcommand:

```bash
brew install --cask nrfutil
nrfutil install nrf5sdk-tools==1.1.0
```

Verify it's working:

```bash
nrfutil nrf5sdk-tools --help
```

#### 2. Enter Bootloader Mode

Hold the button (on the far side of the board from the USB-C connector) while plugging the dongle into USB. The red LED should start a fade pattern, which means the bootloader is running.

> **Note:** The button doesn't face up—you push it from the outside in, towards the USB connector.

#### 3. Find Your Serial Port (macOS)

With the dongle in bootloader mode, find the serial device:

```bash
ls /dev/cu.usb*
```

You should see something like `/dev/cu.usbmodem0001` or similar. Use this path in the flash command below.

#### 4. Generate DFU Package

Package the hex file for the bootloader:

```bash
nrfutil nrf5sdk-tools pkg generate \
    --hw-version 52 \
    --sd-req=0x00 \
    --application corne_dongle-mdbt50q_cx_40-zmk.hex \
    --application-version 1 \
    corne_dongle.zip
```

#### 5. Flash

Flash the DFU package over USB serial (replace the port with the one you found in step 3):

```bash
nrfutil nrf5sdk-tools dfu usb-serial -pkg corne_dongle.zip -p /dev/cu.usbmodemXXXX
```

When the command finishes, the dongle resets and runs your firmware. The bootloader is not overwritten, so you can repeat this process any time you need to re-flash.
