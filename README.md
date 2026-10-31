# Switcher Guitar Plus 2.0

An open-source, open-hardware loop switcher for analog guitar pedals that thinks in songs, not presets. Firmware runs on Apache NuttX; this is an independent project, not endorsed by the Apache Software Foundation (see Affiliation and trademarks).

Status: concept. The architecture, the business rules and the product brochure are written. There is no schematic, PCB or firmware yet. Everything marked proposed or open below can still change.

## What it is

A classic switcher stores combinations of pedals and leaves the player to remember which combination belongs to which moment of which song. This one stores the setlist itself.

Each song is split into parts (intro 1, intro 2, verse 1, chorus, verse 2, solo 1, solo 2). Each part remembers which pedals are on and which clip sits in each of the 4 clip slots. On stage you step on the part, and the whole rig follows.

The guitar path stays 100% analog, through latching relays. Audio clips and a stereo AUX input join the guitar only at the output mixer, so the guitar signal never passes through a converter.

It is the successor of Switch Guitar Plus (2014), a fully analog design with latching DPDT footswitches, DIP switches as preset memory, a diode matrix, a latch, relays and an OPA2134 input buffer.

## Features

- 7 true-bypass loops on dual-coil latching signal relays.
- 14 soft-touch momentary footswitches in two rows:
  - front row: 7 song parts and bypass;
  - back row: previous song, clips 1 to 4, next song.
- Setlist, song and part model stored in the pedal; the phone is not needed on stage.
- Silent switching: the guitar path is muted while relays move.
- Optional OPA1642 input buffer; its state during bypass is a user setting.
- 4 clip slots, up to 1 minute each, mono, read from a microSD card. Tap plays once, hold loops, double tap stops.
- Stereo AUX input (3.5 mm), routed to the main output or to its own outputs (3.5 mm and 6.35 mm in parallel).
- Two stereo headphone amplifiers: main mix and AUX.
- Display: ST7789 or ST7796 over SPI, chosen at build time.
- 3 push-button encoders for navigation and settings.
- Phone app: setlists over BLE, clips over Wi-Fi.
- All connectors on the rear panel.

## How it works

### Footswitches

| Switch | Action |
| --- | --- |
| Part 1 to 7 | Applies that part's pedals and clip slots |
| Bypass | Takes every loop out; press again to return to the part |
| Prev, next | Walks the setlist; a new song opens on its first part |
| Clip 1 to 4 | Tap: play once. Hold about 500 ms: loop. Double tap: stop |

Bypass returning to the part and a new song opening on its first part are proposed behaviours.

A part switch whose slot is unused in the current song does nothing, and its LED stays off.

### Switching sequence

Mute the guitar path, pulse the relay coils, wait for the contacts to settle, open the mute. The target is under 20 ms from press to sound. Clips and AUX keep playing through the switch, because the mute sits before the mixer.

### Clips

The first press always starts the clip immediately, so the pedal never waits to see whether a second press is coming. A double tap therefore lets up to about 300 ms of the clip through before the fade stops it.

Clips are prepared before or after a show: imported and trimmed in the app, then synced over Wi-Fi or copied to the microSD card. The app never edits in real time.

### Audio path

```text
Guitar in -> Buffer -> Loops 1-7 -> Mute -> Mixer -> OUT
                                      ^      |-> Phones (main)
                     Clips (codec) ---|
                     AUX (mono, padded, when routed to main)

AUX in -> AUX route -> AUX out (3.5 mm + 6.35 mm) -> Phones (AUX)
```

The guitar on headphones sounds right only when the last pedal is an amp and cabinet simulator.

## Hardware

| Block | Choice |
| --- | --- |
| MCU | ESP32-S3-WROOM-1 with PSRAM |
| Loops | 7 DPDT dual-coil latching signal relays |
| Relay drive | 74HC595 chain and ULN2803 |
| Footswitch input | 74HC165 chain, RC filter per switch |
| Mute | PhotoMOS shunt on the guitar path |
| Buffer | OPA1642, unity gain, 4.5 V reference |
| Codec | I2S codec, part not chosen |
| Headphone amps | 2 x OPA1622 class |
| Storage | Internal flash for setlists, microSD for clips |
| Display | ST7789 (240x320) or ST7796 (480x320), SPI, IPS preferred |
| Power | 9 V DC, center negative |

Power is split into five rails: a 3.3 V buck for digital, a low-noise LDO for the codec analog side, a relay rail with its own return, a filtered 9 V analog rail with a 4.5 V reference, and a headphone rail. Analog and digital returns join at one point.

The enclosure is a wedge: low at the front, high at the back. The display and the encoders sit on a raised strip at the back, out of the foot path, tilted about 15 to 30 degrees toward a standing player.

## Firmware

The firmware is a NuttX application on a custom board. The application uses only POSIX calls on standard NuttX device nodes; the board layer hides how the display and the relay coils are wired.

### Device nodes (proposed)

| Node | Purpose |
| --- | --- |
| /dev/buttons | 14 footswitches |
| /dev/qe0 to /dev/qe2 | 3 encoders |
| /dev/audio/pcm0 | Codec, clip playback and recording |
| /dev/fb0 | ST7789 or ST7796 framebuffer |
| /dev/loops | Relay mask and mute |
| /dev/mmcsd0 | microSD, mounted at /sd |
| flash partition | Setlists, mounted at /data |

### Tasks

| Task | Role | Priority |
| --- | --- | --- |
| Input | Footswitch and encoder events | High |
| Show state | Current setlist, song and part | High |
| Switching | Mute, coil pulse, settle, unmute | Highest |
| Clip engine | Streams microSD to PSRAM to I2S | High |
| UI | LVGL screens and encoder menus | Low |
| BLE, Wi-Fi sync | Setlists and clips from the app | Low |
| Storage | Loads and saves setlists | Low |

Tasks talk only through message queues.

### Kconfig options

- Display: ST7789 or ST7796, which also selects the UI layout.
- Number of loops, default 7.
- Input buffer present or not.
- BLE editing on or off.
- Clip player on or off.

## Repository layout (proposed)

```text
hardware/    KiCad schematics, PCB and BOM
enclosure/   Mechanical files
firmware/
  boards/    Out-of-tree NuttX board for the custom PCB
  apps/      Switcher application
app/         Phone app
docs/        Architecture document and brochure
```

## Building (planned)

There is no board configuration yet. Once there is, the build will follow the standard NuttX flow for the ESP32-S3; the board and configuration names below are placeholders.

```bash
cd nuttx
./tools/configure.sh -l <board>:<config>
make -j$(nproc)
make flash ESPTOOL_PORT=/dev/ttyUSB0 ESPTOOL_BINDIR=./
```

See the NuttX documentation for the ESP32-S3 toolchain and flashing details.

## Open decisions

- Bypass and song-change behaviour (proposed above).
- What prev does on the first song and next on the last.
- A tap on a clip that is already playing: restart or ignore.
- What playing clips do when the part changes.
- Live recording: a part records into a slot from entering to leaving it, and an empty slot records when pressed (proposed).
- AUX routing global or per song.
- Headphone volume: analog pots or a digital attenuator on the encoders.
- Codec part, mixer topology, gains and AUX pad value.
- Power budget and adapter rating.
- BLE GATT server and codec driver support on NuttX for the ESP32-S3.
- Final pin map, programming and debug port.
- Setlist storage format: compact binary or JSON.

## Backlog

- Pedal order per part. It needs an 8x8 routing matrix, either 64 relays or an analog crosspoint IC with buffers on every input and output, which gives up true bypass.

## License

Firmware, schematics, PCB and enclosure files will be published as open source and open hardware. The proposed licenses are:

| Part | License | Status |
| --- | --- | --- |
| Firmware and phone app | Apache License 2.0 | Proposed |
| Schematics, PCB, enclosure | CERN-OHL, variant to be chosen | Proposed |
| Documentation | To be chosen | Open |

Third-party components keep their own licenses:

- Apache NuttX is licensed under the Apache License 2.0. Firmware binaries built from this project include NuttX, so anyone who distributes them must keep the NuttX license and NOTICE files with them.
- LVGL is licensed under the MIT License.
- The BLE and Wi-Fi controllers of any ESP32 are closed Espressif binaries, distributed under Espressif's own terms. They are the only closed parts of the firmware.

Each file will carry its license header; the full texts will live in LICENSE files at the repository root and in each top-level folder.

## Affiliation and trademarks

This project uses Apache NuttX as its operating system. It is an independent project and is not part of, affiliated with, sponsored by or endorsed by the Apache Software Foundation or the Apache NuttX project.

Apache, Apache NuttX, NuttX and the Apache feather logo are trademarks of the Apache Software Foundation. Espressif and ESP32 are trademarks of Espressif Systems. Other names belong to their respective owners.

## Contributing

The project is at the design stage. Issues and discussion on the open decisions above are the most useful contributions right now. Code follows the NuttX coding style.

## References

- NuttX OS documentation: https://nuttx.apache.org/docs/latest/reference/os/index.html
- NuttX repository: https://github.com/apache/nuttx
- NuttX coding style: https://nuttx.apache.org/docs/latest/contributing/coding_style.html
- NuttX inviolable principles: https://nuttx.apache.org/docs/latest/introduction/inviolables.html
