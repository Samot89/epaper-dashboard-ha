# Household E-Paper Dashboard (LaskaKit Live-7.5 + ESPHome + Home Assistant)

*[Přečíst v češtině](README.cs.md)*

A battery-powered 7.5" e-paper display (800×480, black/white) showing a
household overview: indoor/outdoor temperature, current power draw and
water heater temperature, grid feed-in, day-ahead electricity spot
prices (24h chart), a 5-day weather forecast, upcoming calendar events,
a name-day/holiday line, and optionally a national weather-service
alert banner. Built on ESPHome and integrated with Home Assistant.

The device runs in `deep_sleep`, waking for only a few seconds every
~10 minutes to redraw the display and go back to sleep - a single
battery charge should last weeks to months, depending on battery
capacity and the chosen wake interval.

## Hardware

- **[LaskaKit Live-7.5](https://www.laskakit.cz/laskakit-live-7-5-e-paper-stavebnice-s-wifi-pro---zivy-obraz-/?variantId=13529)**
  kit - the e-paper carrier board is **ESPink v3.6**
  with an **ESP32-S3** (note: older revisions v2.7/v3.5 use a classic
  ESP32, not the S3 - pinout differs, see below).
- Panel: **Good Display GDEY075T7**, 800×480, black/white.
- Optional: a single-cell 3.7V LiPo/Li-ion battery for wireless operation.

### Pinout (ESPink v3.6, ESP32-S3)

| Function                      | GPIO |
|--------------------------------|------|
| Display CS                     | 10   |
| Display DC                     | 48   |
| Display BUSY (inverted!)       | 38   |
| Display RESET                   | 45   |
| SPI CLK                        | 12   |
| SPI MOSI                       | 11   |
| I2C SDA (SHT40 sensor)          | 42   |
| I2C SCL (SHT40 sensor)          | 2    |
| Battery ADC                     | 9    |
| Display power (LDO, HIGH=on)   | 47   |

## What you need in Home Assistant

- A running Home Assistant instance with the **ESPHome** add-on (or a
  standalone ESPHome Dashboard).
- Some electricity spot-price sensor exposing hourly/15-min attributes
  (e.g. a "Spot Price" style integration for your market).
- At least one `calendar.*` entity (Google calendar, birthdays,
  holidays, ...).
- A working `weather.*` entity that supports `weather.get_forecasts`.
- Optional: a weather-alert sensor exposing a `severityLevel` attribute
  (e.g. a HACS integration providing national weather-service alerts)
  - easy to remove if you don't have one.

## Installation

1. **Copy `esphome/epaper-dashboard.yaml`** into your ESPHome folder
   (`/config/esphome/` in Home Assistant).
2. **Create `esphome/secrets.yaml`** from `secrets.yaml.example` and
   fill in your WiFi password, an API encryption key (generate one at
   https://esphome.io/encryption-key), and a fallback AP password.
3. **Copy `packages/ha_spot_price_template.yaml`** into
   `/config/packages/` and make sure `configuration.yaml` has:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
   (This does not recurse into subfolders - the file must live directly
   in `packages/`.)
4. **Update the substitutions and templates** to match your own
   entities - look for "CUSTOMIZE" / "PŘIZPŮSOB" comments in both
   files. Mainly:
   - entity IDs for temperature/power/water heater/feed-in/pool sensors,
   - the calendar list in `packages/ha_spot_price_template.yaml`,
   - your `weather.xxx` entity for the forecast,
   - the source spot-price sensor.
5. **Validate the configuration** (Developer Tools → YAML → Check
   Configuration) and **restart Home Assistant** (a plain `reload` is
   not always reliable for newly added trigger-based template sensors -
   see Troubleshooting).
6. In the ESPHome Dashboard, **compile and install** the firmware
   (first install over USB, later updates can go over the network/OTA).

## Battery and deep_sleep

`sleep_duration` (a substitution in the ESPHome YAML) controls how long
the device sleeps between redraws. Shorter intervals mean fresher data
but shorter battery life (more wake cycles = more WiFi-handshake
energy spent). 10 minutes is a reasonable compromise for a household
dashboard; 20-30 minutes will meaningfully extend battery life if
slightly staler data is acceptable.

Battery is shown both as a percentage and in volts - the percentage is
a rough piecewise-linear approximation of a single-cell Li-ion/LiPo
discharge curve (4.2V = 100%, 3.7V ≈ 40%, 3.3V = 0%). If you use a
different battery chemistry or cell count, adjust the breakpoints in
the `batt_v`/`batt_pct` calculation inside the `lambda`.

## Troubleshooting

These are real bugs we hit while building this project - so you don't
have to rediscover them.

### Display stays completely blank, no error in the logs
Most common cause: **wrong `cs_pin`**. The ESPink v3.6 board's
"Display:" silkscreen label lists two CS pins - `CS: IO10` (primary,
for single-chip panels like the GDEY075T7) and `CS2*: IO18` (an
optional second chip-select, only for dual-chip panel variants). If you
accidentally use 18 instead of 10, the SPI write "succeeds" (SPI is a
one-way bus with no acknowledgement), but the display never receives
the right signal and stays blank - with no error logged anywhere.
Always verify pins from a sharp photo of the actual board label, not
just an older GitHub guide (those often describe the older v2.7/v3.5
revision with a classic ESP32).

### `busy_pin` without `inverted: true`
For this specific panel/controller (7.50inv2 / GDEY075T7),
`inverted: true` on `busy_pin` is **mandatory** - per ESPHome's own
documentation, omitting it risks permanent display damage. Don't skip
this, even for quick testing.

### `weather.get_forecasts` keeps failing with "Referenced entities X
are missing or not currently available"
Even after restarting Home Assistant and even after trying a
templated `entity_id`. The real cause is usually a **duplicate/orphaned
weather entity** - the same integration set up more than once, where
one instance carries identical data but has no live backing
connection. Check `Settings → Devices & Services → Entities` for more
than one similarly-named `weather.*` entity from the same integration,
and use the one that's actually live.

### The calendar template sensor goes "unavailable" after a restart
Jinja's `sort()` throws if you mix "naive" (no timezone - typically
all-day events like birthdays/holidays) and "aware" (timezone-attached)
`datetime` objects in the same list. Fix: normalize every timestamp
through `as_local()` **before** storing it in the sort pair, and
coerce `ev.start` through the `| string` filter (some calendars return
a datetime object instead of a string).

### Font logs "Codepoint 0x____ not found in font"
The custom glyph set (`glyphs:`) only contains the characters you
listed - if a calendar event title or any other text contains a
character that isn't in there (typically an en dash "–", or other
punctuation), the font won't render it correctly. Add the missing
character to the `cz_glyphs` string in the `font:` section.

### The big one: `deep_sleep` seems to "not work" - the device stays
awake continuously, even after flashing firmware with deep_sleep enabled

This one is non-obvious and cost us significant time, so it deserves
top billing for anyone combining **ESPHome OTA + deep_sleep**.

**Symptom:** In the live log (`ESPHome Dashboard → Logs → On the
network`), the device keeps printing sensor readings continuously,
without the expected ~10-minute silent gaps. Home Assistant's sensor
history shows updates every few tens of seconds for hours on end,
instead of short bursts separated by long gaps.

**Root cause:** ESPHome has built-in **OTA rollback protection** (the
`safe_mode` component). Freshly flashed firmware is considered
"pending" until it has run uninterrupted for at least
`boot_is_good_after` (default **1 minute**) since boot - only then is
it permanently marked as valid, so the bootloader will boot it
confidently next time. If your `deep_sleep` script goes to sleep
**before** that window elapses (common for a "wake up, send data, go
back to sleep" flow that only runs a few seconds), the firmware is
never marked valid. Every subsequent wake from deep sleep looks like a
plain reset to the bootloader - and because the last-run version was
never confirmed, the ESP32 performs an **automatic rollback to the
previous (older) firmware partition** on the next reset. From the
outside, it looks like your new changes (including enabling
deep_sleep itself) simply never took effect - because the device is
actually running the old version, which might not have had
deep_sleep at all.

You'll see this in the log as:
```
[W][safe_mode]: OTA rollback detected! Rolled back from partition 'app1'
[W][safe_mode]: The device reset before the boot was marked successful
```

**Fix:** add a `safe_mode.mark_successful` action to your wake script,
**right after a successful API connection, before calling
`deep_sleep.enter`**:

```yaml
script:
  - id: update_and_sleep
    then:
      - switch.turn_on: epaper_power
      - delay: 200ms
      - wait_until:
          condition:
            api.connected:
          timeout: 15s
      - safe_mode.mark_successful   # <-- this is the key line
      - delay: 3s
      - component.update: epaper_displej
      - delay: 6s
      - deep_sleep.enter: deep_sleep_1
```

This marks the firmware valid within a few seconds of boot instead of
waiting out the default 1-minute window, so the rollback guard no
longer interferes with a deliberately short wake cycle. After this
fix, we verified hundreds of clean ~10-minute cycles on real hardware
with zero anomalies.

## Extending

- **Solar (PV) production:** substitutions and a column slot for solar
  production/battery are noted in the YAML comments as an extension
  point - add another `homeassistant` sensor plus a column in the
  `lambda`, following the "Power draw / Water heater" pattern.
- **More/fewer upcoming events:** adjust `MAX_EVENTS` in the ESPHome
  lambda and `sorted_items[:N]` in `ha_spot_price_template.yaml` -
  both must match, or the display will receive truncated data
  regardless of how many rows it has room to show.

## License

MIT, see [LICENSE](LICENSE). The display and firmware are not an
official LaskaKit product - this is a community configuration for
their hardware.
