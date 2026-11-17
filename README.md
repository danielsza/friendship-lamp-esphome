# Friendship Lamp (ESP32-C5) on ESPHome

ESPHome firmware for the Loooit / FriendLamps "Friendship Lamp" touch lamp, the version built
around an ESP32-C5-MINI-1 module (the stock firmware calls it `flp_c5`). It replaces the cloud
firmware so the lamp is a local Home Assistant light with a touch button.

Status: working on one lamp (light, colour, brightness, touch). Over-the-air updates are not
reliable yet, see Known issues.

## Hardware

| Function | Pin | Notes |
|---|---|---|
| LED data | GPIO27 | 16 x WS2812, GRB order. GPIO27 is a strapping pin. |
| Touch pad | GPIO3 | Wire to the top cap. Reads 3.3 V idle and 0 V while touched, so it works as a plain digital input (inverted). |
| Serial | TXD / RXD header pads | UART0. Used for flashing and logs. |
| BOOT | unlabelled button | Hold while pressing RST to enter the serial bootloader. |
| RST | RST button | Reset. |

The board also has `3V3`, `GND`, `IO` and `EN` pads. The USB-C socket carries power only, so
flashing needs a 3.3 V USB-to-serial adapter on the header pads.

The LED pin is not obvious from the board because the traces run under the module. It was read
out of a dump of the stock firmware: the LED init routine takes its GPIO number from a byte in
the data segment, and that byte is 27.

## Files

- `common/friendship-lamp.yaml`: the firmware, shared by every lamp.
- `vine-lamp.yaml`, `vine-lamp-2.yaml`: one small file per lamp (name and default tap action).
- `secrets.yaml.example`: copy to `secrets.yaml` and fill in.

## What you get in Home Assistant

- **Lamp**: RGB light with brightness and the effects Breathe, Flash and Rainbow.
- **Touch**: on while the pad is touched.
- **Touch gesture**: an event entity that fires `tap` and `hold`, for automations.
- **Tap action**: what a tap does on the lamp itself. Change it in Home Assistant, no rebuild needed.
  - `Status`: off -> green, then each tap swaps green and red.
  - `Call`: breathing amber.
  - `Toggle`: light on/off.
  - `None`: nothing locally; Home Assistant still gets the event.
- **Call**: switch that is on while the lamp is breathing amber. Can be turned on or off from Home Assistant.
- **Restart** button and **Reset reason** sensor, for troubleshooting.

Holding the pad for about 0.8 s always turns the lamp off.

## Using it from the Home Assistant ESPHome add-on

The add-on can pull the shared firmware straight from this repo, so each lamp only needs a few
lines there. Add `lamp_api_key`, `lamp_ota_password` and `lamp_ap_password` to the add-on's
secrets, then:

```yaml
substitutions:
  name: vine-lamp-2
  friendly_name: "Vine Lamp 2"
  default_tap_action: "Call"

packages:
  lamp:
    url: https://github.com/danielsza/friendship-lamp-esphome
    ref: main
    files: [common/friendship-lamp.yaml]
    refresh: 0s
```

## Flashing over serial

1. Open the lamp and connect the adapter: adapter TX -> `RXD`, adapter RX -> `TXD`, GND -> `GND`.
   Power the lamp from its own USB-C cable. The pads are small and lift easily, so use thin
   wire and little heat.
2. Enter the bootloader: hold the unlabelled button, press and release RST, release the button.
3. Back up the stock firmware first:

   ```bash
   esptool --chip esp32c5 -p /dev/cu.usbserial-0001 -b 115200 read-flash 0x0 0x400000 stock.bin
   ```

4. Build and flash:

   ```bash
   esphome compile vine-lamp.yaml
   esptool --chip esp32c5 -p /dev/cu.usbserial-0001 -b 150000 erase-flash
   esptool --chip esp32c5 -p /dev/cu.usbserial-0001 -b 150000 write-flash 0x0 \
     .esphome/build/vine-lamp/build/firmware.factory.bin
   ```

5. Press RST (or pull power) to start the new firmware. esptool's automatic reset does nothing
   with only TX, RX and GND connected.

Notes:

- esptool 5.x is needed for the ESP32-C5. Older versions report an unknown chip ID.
- 115200 to 150000 baud worked on a poor connection; 9600 did not.
- Logs: `screen /dev/cu.usbserial-0001 115200`. ESPHome logs to USB by default on the C5, which
  is why the config sets `hardware_uart: UART0`.

## Known issues

- **Over-the-air updates**: the upload succeeds, but the lamp drops off the network at the
  restart and needs a power cycle. Pulling power soon after an update has ended up with the lamp
  back on its previous firmware. Cause not found yet.

