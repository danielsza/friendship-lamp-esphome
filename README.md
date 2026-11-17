# Friendship Lamp (ESP32-C5) on ESPHome

ESPHome firmware for the Loooit / FriendLamps "Friendship Lamp" touch lamp, the version built
around an ESP32-C5-MINI-1 module (the stock firmware calls it `flp_c5`). It replaces the cloud
firmware so the lamp is a local Home Assistant light with a touch button.

Status: working on two lamps (light, colour, brightness, touch, over-the-air updates).

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
- `home-assistant/linked-lamps.yaml`: Home Assistant automations that link two lamps.
- `lamp_original_firmware.bin`: dump of the stock firmware, for going back.

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

## Linking two lamps

`home-assistant/linked-lamps.yaml` holds two automations that pair a status lamp with a call lamp:

| Action | Lamp 1 (tap action `Status`) | Lamp 2 (tap action `Call`) |
|---|---|---|
| Tap lamp 1 | Green, then red on the next tap | Copies lamp 1's colour |
| Hold lamp 1 | Off | Off |
| Tap lamp 2 | Breathes amber (call) | Breathes amber |
| Hold lamp 2 | Back to what it showed before the call | Off |
| Tap lamp 1 during a call | Answers: back to its status colour | Copies that colour |

Only the touch button triggers the link. A colour set from Home Assistant, Siri or an automation
changes that lamp alone.

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
   Power the lamp from its own USB-C cable.

   **The pads lift off the board very easily.** Do not solder a whole wire to them: the weight
   and stiffness of the wire is enough to tear a pad off. Solder just a strand or two from a
   stranded wire, use as little heat as you can, and keep the wires from being tugged.
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

## Troubleshooting

- **Over-the-air updates** work. If a lamp drops off the network after an update, check the
  WiFi details in the secrets the build used: a wrong SSID looks exactly like a failed update,
  and pulling power within the first minute makes the lamp fall back to its previous firmware.

