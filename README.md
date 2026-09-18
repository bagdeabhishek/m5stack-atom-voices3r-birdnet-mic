<p align="center">
  <img src="assets/birdlogo.png" alt="BirdNET microphone" width="240">
</p>

# M5Stack Atom VoiceS3R microphone for BirdNET

Turn an **M5Stack Atom VoiceS3R / EchoS3R (C126)** into a compact Wi-Fi microphone for
**BirdNET-Go** or **BirdNET-Pi**. The firmware captures the board's built-in MEMS microphone through
its ES8311 codec and serves mono 16-bit PCM audio over RTSP.

No external microphone, soldering, or I2S wiring is required.

> This is a board-specific fork of
> [Sukecz/esp32-birdnet-mic](https://github.com/Sukecz/esp32-birdnet-mic). It exists because the
> VoiceS3R requires its own ES8311 initialization and build target. The upstream web flasher and
> automatic firmware downloads do **not** support this board; build and flash it with PlatformIO.

## What works

- Built-in ES8311 codec and MEMS microphone
- RTSP stream at `rtsp://<device-ip>:8554/audio1`
- BirdNET-Go over RTSP/TCP
- BirdNET-Pi over RTP/UDP
- Web UI and JSON diagnostics on port 80
- Adjustable sample rate, gain, buffer size, high-pass filter, CPU speed, and Wi-Fi power
- Thermal protection, stream recovery, MQTT, mDNS, scheduling, and manual OTA upload

The VoiceS3R implementation has been tested on real C126 hardware with BirdNET-Go. The speaker and
NS4150B power amplifier are not used.

## Hardware

- [M5Stack Atom VoiceS3R / EchoS3R (C126)](https://docs.m5stack.com/en/core/Atom_EchoS3R)
- USB-C data cable
- Stable 5 V USB power supply
- 2.4 GHz Wi-Fi

The firmware uses these built-in connections:

| Function | GPIO |
|---|---:|
| ES8311 I2C SDA | 45 |
| ES8311 I2C SCL | 0 |
| I2S MCLK | 11 |
| I2S BCLK | 17 |
| I2S word select | 3 |
| Microphone data | 4 |
| NS4150B amplifier control | 18, left inactive |

## Build and flash

Install [PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/index.html), then:

```bash
git clone https://github.com/bagdeabhishek/m5stack-atom-voices3r-birdnet-mic.git
cd m5stack-atom-voices3r-birdnet-mic
pio run -e m5stack-atom-voices3r
pio run -e m5stack-atom-voices3r -t upload
```

If PlatformIO cannot find the board automatically, put the VoiceS3R into download mode: hold its
reset button for about two seconds until the internal green LED lights, then release it. After a
successful upload, unplug USB for a few seconds and reconnect it without holding the button.

The app-only firmware binary is generated at:

```text
.pio/build/m5stack-atom-voices3r/firmware.bin
```

Keep that file for later manual OTA updates from `http://<device-ip>/ota`.

## First boot

1. Connect to the temporary Wi-Fi network `ESP32-RTSP-Mic-AP`.
2. If the captive portal does not appear, open `http://192.168.4.1`.
3. Select your 2.4 GHz network and save its credentials.
4. Find the device in your router's client list and reserve its IP address.
5. Open `http://<device-ip>/` to check the microphone and stream status.

The default mDNS name is unique to the device, such as `esp32mic-a1b2c3.local`, but a reserved IP
address is usually more reliable for a continuously running BirdNET source.

## Connect BirdNET

Use this URL as the audio source in BirdNET-Go:

```text
rtsp://<device-ip>:8554/audio1
```

For example:

```text
rtsp://192.168.1.100:8554/audio1
```

Test the stream independently before configuring BirdNET:

```bash
ffplay -rtsp_transport tcp rtsp://<device-ip>:8554/audio1
ffprobe -rtsp_transport tcp rtsp://<device-ip>:8554/audio1
```

`/audio2` is available as a second independently configured stream. `/audio` remains an alias for
`/audio1` for compatibility.

## Recommended starting settings

The firmware defaults are suitable for initial testing. For continuous BirdNET-Go use on a weak
Wi-Fi connection, this profile is a useful starting point:

| Setting | Suggested value |
|---|---:|
| Sample rate | 32,000 Hz |
| Gain | 1.0 |
| Buffer | 2,048 samples |
| I2S shift | 0 |
| High-pass filter | On, 250-500 Hz |
| CPU | 160 MHz |
| Wi-Fi TX power | Lowest value that remains stable |
| Thermal shutdown | 80 °C |

BirdNET commonly resamples input internally, so 32 kHz reduces network and processing load without
discarding the frequency range used by most audible bird calls. Use 48 kHz if you need more
high-frequency bandwidth.

## Temperature and Wi-Fi

The temperature shown in the Web UI is the ESP32-S3's internal sensor—not ambient air temperature.
It will read significantly warmer than the room while the CPU and radio are active.

There is also an accuracy limitation in the current firmware stack: Arduino's `temperatureRead()`
requests a 10–50 °C expected range when it initializes the ESP32-S3 sensor. Values above the
driver-selected calibrated range should be treated as a strong overheating trend, not a
laboratory-accurate case or ambient temperature. This does not make a 90 °C report harmless, but it
does mean the absolute number may be off by several degrees.

This firmware is a demanding thermal workload because it continuously captures audio, packetizes
RTSP, keeps Wi-Fi awake with `WiFi.setSleep(false)`, and streams without idle periods. Weak Wi-Fi
adds retransmissions and radio-on time. The tiny plastic enclosure has little airflow, so direct sun
or a hot outdoor mounting location can push the internal reading much higher.

For reliable operation:

- Aim for RSSI better than **-70 dBm**; below roughly **-75 dBm**, improve the access-point location
  before increasing transmit power.
- Keep the device shaded and out of sealed, sun-heated enclosures.
- Leave space around the case vents; do not wrap it in foam or tape.
- Use the lowest Wi-Fi TX power that keeps the RTSP packet rate stable.
- Keep thermal protection enabled at **80 °C**. Do not treat a higher shutdown limit as a cooling fix.
- Use a stable 5 V supply and avoid unnecessary powered peripherals.

The NS4150B speaker amplifier is held in hardware shutdown when GPIO18 is low and is not part of the
microphone path. It is not expected to be a meaningful heat source in this firmware.

## Useful diagnostics

The Web UI is available at `http://<device-ip>/`. Read-only JSON endpoints include:

```text
GET /api/status
GET /api/audio_status
GET /api/perf_status
GET /api/thermal
GET /api/logs
```

Healthy streaming normally means:

- `producer_running` is `true`
- `i2s_error_count` remains at zero
- `/audio1` has one connected client
- packet rate is stable for the selected sample rate and buffer size
- RSSI is not persistently below approximately -75 dBm

## Updating

Automatic OTA from the upstream project is intentionally disabled for this board because its
generic ESP32-S3 image is not compatible with the VoiceS3R audio hardware.

To update safely:

1. Pull and rebuild the `m5stack-atom-voices3r` environment.
2. Open `http://<device-ip>/ota`.
3. Upload `.pio/build/m5stack-atom-voices3r/firmware.bin`.
4. Confirm the board ID is `m5stack-atom-voices3r` after reboot.

Most runtime settings are stored in NVS and normally survive firmware restarts and OTA updates.
Keep a USB cable available in case a development build requires recovery.

## Troubleshooting

### BirdNET receives no audio

- Confirm `ffplay` can open `/audio1`.
- Check that stream 1 is enabled and has a connected client.
- Verify `producer_running=true` and `i2s_error_count=0` in `/api/audio_status`.
- Confirm the startup log reports pins `11/17/3/4` and ES8311 address `0x18`.
- Do not flash the upstream generic ESP32-S3 binary; it does not initialize this board's codec.

### Reconnects or low packet rate

- Improve Wi-Fi first. Moving the access point is usually more effective than maximum TX power.
- Reserve the device IP and use it instead of mDNS.
- Try the 32 kHz / 2,048-sample profile above.
- Check `/api/logs` for `RTP write failed`, I2S resets, or thermal shutdown.

### Device is hot

- Compare the internal reading with the local ambient temperature; they are not the same quantity.
- Check RSSI and stream retries.
- Move the board into shade with free airflow.
- Restore the thermal cutoff to 80 °C if it was raised.
- If the case itself becomes painful to touch, smells hot, or the device behaves erratically,
  disconnect power and inspect the supply and board before continuing.

## Project layout

- `esp32-birdnet-mic/` — firmware and detailed upstream-oriented reference documentation
- `platformio.ini` — tested VoiceS3R build environment
- `boards/m5stack-atoms3r.json` — PlatformIO board definition
- `.github/workflows/arduino-build.yml` — compile validation

## Credits and license

VoiceS3R support was developed from the
[`feature/m5stack-atom-voices3r`](https://github.com/Sukecz/esp32-birdnet-mic/pull/57) work on top of
the original [ESP32 BirdNET microphone firmware](https://github.com/Sukecz/esp32-birdnet-mic) by
Sukecz and its contributors.

Released under the [MIT License](LICENSE).
