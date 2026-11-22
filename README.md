# ESP32 bridge for JBD batteries

Each ESP32 board sits near your batteries, powered from USB, and talks to up to
4 JBD BMS units over Bluetooth. It joins your Wi-Fi and serves a live page with
every battery's charge, voltage, current, cell voltages, temperatures and
errors. You open that page on the Android tablet. No app to install.

```
 JBD BMS ))) Bluetooth ))) ESP32 ~~~ Wi-Fi ~~~ router ~~~ tablet (browser)
 JBD BMS )))                 |                       \~~ Home Assistant (optional)
                       USB power
```

Files in this folder:

| File | What it is |
|---|---|
| `jbd-bridge.yaml` | The board's configuration. You edit the board name and the battery list. |
| `jbd-bms.yaml` | The per-battery part, included once per battery. No need to edit. |
| `secrets.yaml.example` | Wi-Fi name and passwords. Copy to `secrets.yaml` and fill in. |

The config uses [syssi/esphome-jbd-bms](https://github.com/syssi/esphome-jbd-bms)
and passes ESPHome 2026.10.0 validation with 4 batteries.

## 1. Before you start: the antenna

The ESP32-DevKitC-**VIE** has no antenna on the board, only a tiny U.FL (also
called IPEX or IPX) socket on the metal module. **You need a 2.4 GHz antenna
with a U.FL plug**, usually sold as a "2.4G WiFi antenna with IPEX pigtail".
The same antenna serves both Wi-Fi and Bluetooth.

- Clip the antenna on **before** you power the board. Running the radio with
  nothing attached gives terrible range and can stress the transmitter.
- U.FL connectors are fragile. Press straight down until it clicks, pull
  straight up by the plug, never by the cable.
- An external antenna is the reason to pick this board: you can mount the
  antenna outside a metal battery box or cabinet and keep the board inside.

## 2. Find the BMS MAC addresses

Every JBD BMS has a Bluetooth address like `A4:C1:37:12:34:56`. You need one
per battery.

1. Fully close the Xiaoxiang / JBD app on every phone. **A JBD BMS accepts
   only one Bluetooth connection at a time**, so a phone that's connected hides
   it from everything else, including the ESP32 later.
2. On the Android tablet or phone, install **nRF Connect for Mobile** (by
   Nordic Semiconductor) from the Play Store and tap **Scan**.
3. Look for names the BMS app shows, typically `xiaoxiang BMS`, `SP04S…`,
   `SP17S…`, `DP04S…` or `JBD-…`. The MAC address is printed under the name.
   Holding the phone right next to one battery makes it the strongest signal
   in the list, which tells you which MAC belongs to which battery.
4. Write each one down with a label you'll recognise ("Shed 1", "Van house").

(iPhones hide MAC addresses, so use Android for this step.)

## 3. Edit the config

Open `jbd-bridge.yaml` in any text editor.

**EDIT 1, the board name.** Every board needs its own:

```yaml
  name: jbd-bridge-1
  friendly_name: "JBD Bridge 1"
```

**EDIT 2, the batteries.** One line per BMS, up to 4. Remove the `#` to turn
on lines 3 and 4, or delete lines you don't need:

```yaml
  bms1: !include { file: jbd-bms.yaml, vars: { id: bms1, name: "Shed 1", mac: "A4:C1:37:12:34:56" } }
  bms2: !include { file: jbd-bms.yaml, vars: { id: bms2, name: "Shed 2", mac: "A4:C1:37:65:43:21" } }
```

Then copy `secrets.yaml.example` to `secrets.yaml` and put in your Wi-Fi name
and password, plus two passwords of your choice.

Keep 4 or fewer batteries per board, and keep the board within roughly 5 to 10
metres of each BMS (less through metal or walls). More batteries means more
boards, which you have.

## 4. Flash the board

You need ESPHome on a computer once. Pick one:

- **Home Assistant users:** install the *ESPHome Device Builder* add-on, create
  a new device, and paste in these files.
- **Any Windows / Mac / Linux computer** with Python:
  ```
  pip install esphome
  esphome run jbd-bridge.yaml
  ```
  Put all three files (plus your `secrets.yaml`) in one folder and run that
  from inside it. The first build downloads the ESP32 toolchain and takes
  10 to 15 minutes. Later builds take a minute or two.

The first flash goes over a **USB data cable** (some cables are charge-only).
If the computer doesn't see the board, install the **CP210x USB driver** from
Silicon Labs (some clone boards use the **CH340** chip instead). If it hangs at
"Connecting…", hold the **BOOT** button on the board until it starts writing.

After that, `esphome run` offers to update over Wi-Fi, so you never need the
cable again. If a wireless update fails, switch off the "Bluetooth link"
switches on the board's page first, or just use USB.

For a second board, change `name`, `friendly_name` and the battery list, then
run it again.

## 5. View the batteries on the tablet

At the end of flashing, the log prints the board's IP address (and your
router's device list shows it too, as `jbd-bridge-1`).

1. On the tablet, open Chrome and go to `http://<that IP>/`, for example
   `http://192.168.1.57/`. On many networks `http://jbd-bridge-1.local/` also
   works.
2. The page shows one section per battery, plus a "Bridge" section with signal
   strength and uptime. It updates live, every 5 seconds.
3. In Chrome's menu, tap **Add to Home screen** to get a one-tap icon.
4. Give each board a fixed IP (a "DHCP reservation" in your router settings),
   so the bookmark never breaks.

Things you'll see per battery: state of charge, total voltage, current
(negative while discharging), power, remaining capacity, cycles, each cell's
voltage, the min/max/delta between cells, temperatures, charging /
discharging / balancing status, and any BMS error text.

**Home Assistant (optional).** If you run Home Assistant, it discovers each
board automatically under Settings → Devices. Then the Home Assistant app on
the tablet gives you history graphs and low-battery alerts across every board
on one screen.

**A custom Android app later.** The board's web server also offers a live
event stream and a simple HTTP interface, so a dedicated tablet app that
gathers every board onto one screen can be built on top of this without
changing the firmware.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Battery "online" is off, values blank | Phone app still connected, wrong MAC, or out of range. Close the app, check the MAC, move the board closer. |
| Phone app can't connect anymore | The ESP32 holds the connection. Turn off that battery's "Bluetooth link" switch on the board's page, use the app, then turn it back on. |
| Board never joins Wi-Fi | After a minute it opens a hotspot named `jbd-bridge-1 setup`. Join it from a phone (password: `fallback_ap_password`) and pick your network. |
| Board keeps restarting / power bank turns off | Some power banks shut off when the load is small. Use a USB charger, or a 12 V-to-USB converter on the battery bank. |
| Cells 9 to 16 are blank | Normal for a 4S or 8S pack. |

## Safety notes

- The board only **reads** the BMS. Switching charge/discharge MOSFETs from the
  web page is deliberately left out, since anyone on your Wi-Fi could open the
  page. `jbd-bms.yaml` has the lines to add it back if you want it.
- Bluetooth gives you a wireless link with no electrical connection to the
  battery, so there's no ground-loop or isolation concern like there is with a
  wired UART connection.
