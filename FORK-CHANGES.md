# What this fork changes

A fork of [IdefixRC/Monitor-Buddy](https://github.com/IdefixRC/Monitor-Buddy),
based on `main` at d857d87 (1.2.0 plus the board-check and accessibility fixes).
Everything is in `src/main.cpp` and `config/config.h.example`; the web flasher,
`lib/` and the `@MB` settings protocol are untouched, and the web flasher's
tests still pass (`node --test web-tests/*.test.mjs`, 329/329).

The commits are separate so the fixes can be taken without the features:

| Commit | What |
|---|---|
| 1 | Verified HTTPS, DST-correct clock, previous-close stock colour; opt-in 12-hour clock and night mode; `AP_SSID`/`AP_PASS` overridable |
| 2 | `millis()` wraparound fix, plus small follow-ups to commit 1 |
| 3 | Opt-in Bitcoin page |
| 4 | This file and a pointer in the README (skip when merging upstream) |

## Fixes (on for everyone)

- **Certificate checks on HTTPS.** Finnhub and GitHub were fetched with
  `setInsecure()`, so the Finnhub API key went out over TLS that anyone on the
  path could intercept. All fetches now use `useBuiltinCACertBundle()`, the
  Mozilla root bundle the ESP32 Arduino core already links in
  (`CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y`). No certificate files to maintain.
  Certificate dates aren't checked by this core config
  (`CONFIG_MBEDTLS_HAVE_TIME_DATE` unset), so it works before NTP has synced.
- **Weather over HTTPS.** Open-Meteo was plain `http://`.
- **Daylight saving on time.** After an NTP sync, the clock, date and moon
  pages extrapolated from `millis()`, so a DST change only showed at the next
  6-hourly resync. They now read the system clock (`localtime_r`) once SNTP
  has synced, so the change shows the minute it happens. Before the first
  sync they still run on from the compile-time seed.
- **Long uptime.** Blink, glance, auto-advance and the two banners compared
  raw `millis()` deadlines (`now > nextBlink` and similar). At the 32-bit wrap,
  about 49.7 days after power-on, the face stopped blinking, auto-advance
  stopped for another ~49 days, and a banner could stick on screen. They now
  compare the signed difference.
- **Stock colour.** Green/red was against today's open; it's now against the
  previous close (`pc`), as on most quote screens, with the day's change in
  percent under the ticker. Falls back to the open if `pc` is missing.

Cost: about 70 KB more flash (the root bundle) and a few dozen bytes of
static RAM.

## Options (off unless set in `config.h`)

These are compile-time only. The web flasher and `/config.json` don't know
about them, so settings saved from the flasher don't change them.

| Option | Default | Effect |
|---|---|---|
| `CLOCK_12H` | `false` | 1-12 with AM/PM above the seconds, no leading zero |
| `NIGHT_MODE` | `false` | Dims the backlight and shows a sleepy face between `NIGHT_START_HOUR` and `NIGHT_END_HOUR` (default 23 to 7, may cross midnight). A touch restores the normal brightness for 30 s and is swallowed, so it doesn't also flip the page. Waits for NTP, so it never dims at the wrong hours. The saved brightness is never changed |
| `NIGHT_BRIGHTNESS` | `8` | Backlight level at night, 8-255 |
| `SHOW_BITCOIN` | `false` | A page after the stock page: BTC/USD, 24 h change, high/low and a 24 h sparkline of hourly closes, from Coinbase's public market data (no account or key). Every minute while shown, every 5 minutes otherwise; candles every 15 minutes |
| `AP_SSID`, `AP_PASS` | as before | The setup hotspot. The password is shown on the setup screen, so a random one costs nothing. Length-checked at compile time (8-26 characters, the width the setup screen can show) |

The Bitcoin page is not in `settings.pages[]`, so `CONFIG_VERSION` and the
settings protocol are unchanged.

## Tested

On a Waveshare ESP32-C6-Touch-LCD-1.47 with pioarduino 55.03.312-1
(Arduino core 3.3.12, ESP-IDF 5.5.5):

- Boots, joins Wi-Fi, NTP sync, no errors or resets in the serial log
- Weather (verified HTTPS) about 2.0 s; Bitcoin stats plus candles about 3.6 s
- Net task stack: 8.4 KB unused of 12 KB, so certificate checking uses about
  3.8 KB. The original 7 KB stack would still leave about 3.3 KB; it's 12 KB
  here only for margin
- Lowest free heap 99 KB (the ~21 KB candles response is scanned in place,
  not parsed into a `JsonDocument`)
- Builds without warnings in `main.cpp` with the template's defaults and with
  every option on

Not tested on hardware: the stock and GitHub pages (no key or user was set),
which share the same certificate-checked client as the two that were.

The candle parser and the price formatting were also unit-tested on the PC
(newest-first order, error objects, short and malformed responses).
