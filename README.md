# CarPlay and Bluetooth custom firmware for the 2017 Subaru BRZ head unit (Clarion CA645UL)

Custom firmware for the stock 6.2-inch Clarion radio in a U.S. 2017 Subaru BRZ. It fixes Bluetooth song titles and album artwork with iPhones, adds a modern Bluetooth audio screen, and runs wireless Apple CarPlay through a Carlinkit CPC200-CCPA USB adapter, decoded in software on the radio's single 600 MHz-class processor.

Everything is installed through the radio's own USB update procedure. Nothing is soldered, and the stock radio, reverse camera and tuner keep working.

**Status:** The Bluetooth fixes, the new Bluetooth screen and CarPlay (picture, touch, music, navigation audio, reverse camera priority) are all working on the car, and CarPlay is in daily use. The [roadmap](#roadmap) covers the remaining polish.

## Key numbers

| Measurement | Before | After |
|---|---|---|
| H.264 High profile decode on the radio, 800x480 test clip | 11.1 fps (stock decoder) | 26.5 fps, or 33.9 fps in fast mode (new NEON decoder) |
| H.264 Baseline decode, same conditions | 20.6 fps | 44.7 fps, or 59.2 fps in fast mode |
| Radio UI work per CarPlay picture | One full-screen Qt repaint, 50-75 ms (build 512) | No repaint; the display hardware composites the picture (build 513) |
| CarPlay pictures decoded per second, 15 fps requested | 10.7 (build 512) | About 15 when parked (build 513) |
| Bluetooth song title, time and track changes with an iPhone | frozen on the first song | working |
| Bluetooth album artwork | placeholder only | working |

## The hardware

Read from the radio itself by the diagnostics and "CarPlay check" builds, and from the stock firmware.

| Item | Value |
|---|---|
| Unit | Clarion CA645UL, unit code PF3764EB, Subaru package 86201CA640 (stock 215) |
| Processor | TI TI811x (the kernel reports TI's TI811x evaluation board), one ARM Cortex-A8 r3p2 with NEON and VFPv3 |
| Chip ID register (0x48140600) | 0x1b96802f: part 0xB968, revision 1 |
| Speed | BogoMIPS 598.01, which suggests about 600 MHz. No frequency scaling in the kernel. |
| Memory | 381,588 kB visible to Linux (boot maps 176 MB + 256 MB; 80 MB is held for coprocessor firmware) |
| Free memory, radio idle | 134-144 MB free plus about 101 MB of file cache |
| Other load | Stock software uses about 21% of the CPU while a decode test runs |
| Coprocessors | Two Cortex-M3 cores (Clarion tuner firmware, TI display firmware) and a C6000 DSP (HD Radio) |
| Graphics | PowerVR SGX530 (OpenGL ES 2), TI HDVPSS display processor, 800x480 at 60 Hz |
| Software | Linux 2.6.37, U-Boot 2010.06, glibc 2.11.1 (soft-float ABI), Qt 4.8.2 with QtQuick 1.1 on eglfs |
| Stock video decoder | libavcodec 54 compiled for ARMv5T: no NEON, no VFP, size-optimized |
| USB | High-speed host only (musb, libusb 1.0.9). No Wi-Fi hardware was found. |

## How the firmware is modified

The update package is a ZIP of 12 components. The one that matters holds the main Linux filesystem: a 512-byte header followed by a gzip-compressed ext3 image.

- **Format:** the header carries a 32-bit additive checksum at offset 0x84 and the size at 0x88. The updater strips the header, decompresses the image and writes it to the system partition of the internal storage. The update routines that were examined check format identifiers, sizes, versions and checksums; no cryptographic signature check was found in them.
- **Sizes:** the image is 99.9 MB compressed and 368,050,176 bytes unpacked (4,421 of 90,112 inodes, 229 of 351 MiB used). The whole package is 18 files and 202,469,319 bytes on the USB drive.
- **Build:** each release starts from a copy of the previous filesystem image. Only the changed files are replaced inside it, keeping their permissions and owner. The image is then checked for consistency, recompressed, and its header and manifest are repaired.
- **Checks before release:** the radio's own decompressor unpacks the image under emulation, the filesystem check must pass cleanly, and every custom file (43 in build 514) is read back and compared. A standalone verifier then confirms the USB drive byte for byte.
- **Round trip:** the stock package rebuilds byte-identically from its unpacked parts, including all 4,014 Qt resource entries (703 QML screens, 3,285 PNG images) and 68 translation catalogs.

Custom programs and screens are added beside the stock software, not in place of it. The stock screens are changed in as few places as possible, and each new screen is a separate file that the stock screen loads. If it is missing or broken, the stock screen shows unchanged.

## Part 1: Bluetooth audio (builds 500-508)

The stock radio showed the first song's title forever with an iPhone, lost the song length after a skip, and never showed artwork.

| Build | What it did | Result on the car |
|---|---|---|
| 500 | First title and artwork patch | Failed. Also introduced a 5 s audio dropout on every track change. |
| 501 | Fixed a Bluetooth socket address byte-order bug (the dropout), artwork advertisement, metadata routing | Shipped inside 502 |
| 502 | Diagnostics: a fixed 512 KiB RAM ring (8,192 records of 64 bytes), off by default, exported to USB | Produced the captures used below |
| 503 | One-byte patch in the stock Bluetooth manager: stop waiting for an AVRCP browsing channel before subscribing to track changes | Titles, time and track changes work |
| 504 | Added the zero terminator OBEX requires on the image request's Type header | Phone accepts the request; radio rejects the reply |
| 505 | OBEX Single Response Mode, tolerant end-of-body handling, 1,000-byte OBEX packets so no L2CAP frame is split | Shipped inside 507 |
| 506 | New Bluetooth audio screen (290x290 artwork, progress bar, transport and shuffle/repeat buttons) and nine accent colors | Shipped inside 507 |
| 507 | "CarPlay check": chip ID, memory, display and USB details, stock decoder benchmark | Artwork and new screen work. Stock decoder: 12.3 fps High. |
| 508 | NEON FFmpeg benchmark, phone name fix, artwork retries after 0.5, 3, 6, 12 and 24 s | Decoder numbers in the table below |

Artwork is fetched with the Bluetooth Basic Imaging Profile over OBEX on L2CAP, by a new library written for this project, and every image is fully decoded by libjpeg before it is shown.

## Part 2: CarPlay (builds 509-514)

### Design

```
iPhone ──Wi-Fi/Bluetooth──> CPC200-CCPA ──USB──> CarPlay helper ──> display APP layer (video)
                                                     │  ▲            sound device (audio)
                                              status │  │ touches, keys, settings
                                                     ▼  │
                                          CarPlay screen inside the radio UI
```

- **Adapter.** The CCPA does the Apple side: authentication, Wi-Fi and the CarPlay session. It hands the host an H.264 stream, PCM audio and control messages over USB (ID 1314:1520). The radio never talks to the phone directly.
- **CarPlay helper.** A separate program in C (about 2,000 lines, 1.78 MB stripped) plays the head unit. It starts idle at boot and does nothing until the CarPlay screen asks. It includes FFmpeg 6.1.2 (H.264 decoder and color conversion only, LGPL) and uses the radio's own USB, sound and C libraries.
- **CarPlay screen.** A small Qt 4 plugin (34 KB) and a QML screen (about 300 lines) inside the stock radio UI. They forward touches and draw the menu.
- **Protocol.** Messages have a 16-byte header (magic 0x55AA55AA, length, type, inverted type). The message layout follows the MIT-licensed node-CarPlay project and was verified against the real adapter. Touch is message type 0x05 with coordinates on a 0-10,000 scale.

### What the phone actually sends

Captured on a Mac with a small test client before any radio work (iPhone 15 Pro, adapter firmware 2025.10.15.1127).

| Property | Value |
|---|---|
| Video | 800x480, H.264 High profile, level 3.1, one slice per picture, no B frames |
| Picture rate | About 30 fps while a map moves, about 2.4 fps on a still screen |
| Bit rate | Median 2.3 Mbit/s moving, 95th percentile 5 Mbit/s |
| Picture size | Average 17.5 KB, largest 91 KB (1.8 times the bits of the benchmark clips) |
| Sample capture | 3,409 pictures, 48 key frames, 59.7 MB in 277 s |
| Media audio | 44.1 kHz stereo PCM |
| Navigation prompts | 44.1 kHz stereo, with duck messages (media to 0.20 volume over 0.5 s) |
| Siri | 16 kHz mono, and the host is asked for microphone audio |

Every picture is a reference for the next one, so none can be dropped to save time.

### Decoder speed

The stock decoder cannot play this stream. An H.264-only FFmpeg 6.1.2 build for the Cortex-A8 with NEON was benchmarked on the radio against it, using 150-frame 800x480 clips at 2.5 Mbit/s.

| Decoder | High profile | Baseline profile |
|---|---|---|
| Stock libavcodec 54 (ARMv5T) | 11.1 fps, 10.3 with color conversion | 20.6 fps |
| FFmpeg 6.1.2 NEON | 26.5 fps, 23.2 with color conversion | 44.7 fps, 35.8 with conversion |
| FFmpeg 6.1.2 NEON, fast mode | 33.9 fps | 59.2 fps |

- **Speed-up:** 2.4 times for High profile, and 3.1 times with fast mode (about 90 ms per picture down to about 30 ms). The gain comes from the newer decoder and NEON together; they were not measured separately.
- **Correctness:** the normal-mode pictures matched the stock decoder exactly (luma checksum over 148 frames).
- **Fast mode** skips the deblocking filter and uses FFmpeg's fast flag. On the real CarPlay capture its worst picture was 39 dB PSNR against the normal decode. CarPlay runs in fast mode.

### Getting the picture on screen

This took five builds. All figures are from the car, with 15 fps requested unless noted. A "skip" is the decoder falling more than 1 s behind and jumping to the next key frame.

| Build | Display path | Phone sent | Decoded | Shown | Skips | CPU: CarPlay / radio UI |
|---|---|---|---|---|---|---|
| 509 | RGB565 pictures in shared memory, painted by Qt, 30 fps asked | not measured | not measured | laggy and smeared | - | - |
| 510 | Same, 20 fps, clean-picture rules | 19.9 fps | 20.9 fps at 48 ms | 8 fps | 21 | saturated |
| 511 | YUV planes converted by a GLES2 shader on the SGX (9.4 ms a draw) | 15.0 fps | 14.2 fps at 71 ms | 8 fps | 4 | 47% / 42% |
| 512 | Same, background items frozen, decoder at lower priority | 15.4 fps | 10.7 fps at 93 ms | 4 fps | about 600 | 41% / 46% |
| 513 | Display APP layer, color key, parked | not recorded | about 15 fps | not recorded | at most 1 | not recorded |
| 513 | Same, real drive with Waze | not recorded | below 15 fps | not recorded | over 500 | 60-80% / about 25% |

What each step found:

- **509 to 510: smearing.** Pictures were being lost between USB and the decoder. The fix was 16 KB USB reads (one transfer each), a check of every picture's H.264 frame number, and holding the last clean picture while asking the phone for a key frame. The lost-picture counter read 0 in the car tests that followed.
- **511: the graphics chip helps, but not enough.** Shader conversion worked, yet the CPU stayed at 100%.
- **512: the radio UI is the cost.** Profiling showed each full-screen Qt refresh takes 50-75 ms of the UI's main thread, of which only about 15 ms was CarPlay drawing. All other programs together used under 5%.
- **513: stop using Qt for video.** The display processor has four layers, bottom to top: APP, camera video, radio UI, and a bitmap layer. The helper now writes each picture straight into the APP layer with NEON color conversion and flips between two buffers, and the CarPlay screen paints the radio's color key so the layer shows through. The UI no longer repaints per picture.

**Reverse camera.** The camera's video layer sits above the APP layer, and the radio's mode manager blanks the APP layer whenever a video source is shown. The helper checks the layer order before using the APP layer, and pauses decoding while the camera is on. The camera took over correctly in the 511 and 513 car tests.

### Tuning for a moving map

One picture has a budget of 66.7 ms at 15 fps. With a moving map the phone's bit rate rises and decoding uses most of that budget, so the drive in the table above skipped far more often than the parked test.

Decode time follows the bit rate. The 277 s capture was re-encoded at lower bit rates and decoded single-threaded on a Mac with the same settings:

| Stream size | Average bit rate | Decode time | Relative |
|---|---|---|---|
| 59.7 MB (as captured) | 1.72 Mbit/s | 5.27 s | 100% |
| 40.5 MB | 1.17 Mbit/s | 3.69 s | 70% |
| 27.8 MB | 0.80 Mbit/s | 2.76 s | 52% |
| 19.7 MB | 0.57 Mbit/s | 2.14 s | 41% |
| 15.2 MB | 0.44 Mbit/s | 1.79 s | 34% |

That is close to a fixed 12% plus a part proportional to the bits. Shrinking the picture to 640x384 at the same bit rate saved only about 7%, so picture size is not the lever. USB is not the limit either: the stream is about 2 Mbit/s on a high-speed link and no pictures are lost.

**Build 514** acts on this:

- Sends the adapter's video bit-rate setting (4, 2, 1 or automatic) when CarPlay connects.
- Cuts the screen plugin's polling from 66 to 5 times a second while the APP layer is in use.
- Records the drive every 2 s without drawing anything: real Mbit/s, decode and conversion CPU time per picture, pictures shown, skips, CPU shares and touch rate. One summary page and a spreadsheet file come out of it.
- Adds a 12 fps choice and remembers frame rate and bit rate.

### Hardware video decoding

TI's larger sister chips have an IVA-HD (HDVICP2) hardware H.264 decoder, so this was investigated in depth.

- **The radio's kernel says no.** Its power-domain and clock tables register the IVA-HD block for the TI814x and DM385 chips but not for TI811x, while TI811x does get the DSP, SGX, HDVPSS and MLB entries.
- **The usual software route is taken.** TI's design runs codec firmware on the video Cortex-M3. On this radio that core runs Clarion's tuner firmware (HD Radio, DAB, SiriusXM control), and no codec firmware or driver is in the stock software.
- **TI's decoder library could run on the main processor.** TI publishes its IVA-HD H.264 decoder (version 02.00.22.01) as a prebuilt library. It is plain Thumb-2 with no M-profile-only instructions, and apart from TI's own hardware-access layer it needs only three trivial external functions. The radio's Linux also allows direct access to hardware registers. So a driver would be possible if the hardware existed.
- **Build 514 carries a probe** to confirm it on the radio itself: it reads the chip ID and the IVA-HD power and clock registers, tries TI's power-up sequence, and restores every register, also if it is stopped part-way.

### Memory use

| Item | Size |
|---|---|
| CarPlay helper program, stripped | 1,775,132 bytes |
| Screen plugin | 33,948 bytes |
| Shared picture memory (two RGB565 buffers plus header) | about 1.5 MB |
| Incoming video queue, hard cap | 6 MiB |
| Audio buffers | 2 s of 44.1 kHz stereo per stream |
| APP layer buffers (two, in the display driver's existing 16 MB reservation) | about 3 MB |

The helper's resident memory on the radio has not been recorded. The idle radio has 134-144 MB free.

### Audio and input

- The helper mixes media, navigation and Siri audio itself, applies the phone's duck messages, and plays the result through the radio's sound device. The device is opened only while CarPlay has sound and released after 3 s of silence, so the radio's own sources get it back. Media at 44.1 kHz stereo is copied without resampling.
- Touch works. The volume knob works.

### Adapter security

The stock adapter firmware was decrypted and inspected. SSH and telnet are off. Its web server does not properly verify the firmware update packages it accepts (CVE-2025-2764). It is reachable only from the adapter's own Wi-Fi, which ships with a weak, hard-coded default password (CVE-2025-2765). Bluetooth pairing appears open to anyone nearby; that was read from the configuration, not tested. The planned hardening is a random Wi-Fi password and a pairing lock, applied over USB at start-up with no reflash.

## Testing without the radio

There is one radio and it is in the car, so every build is tested offline first.

- **Emulated ARM.** The real programs run under ARM emulation against the radio's own filesystem and libraries.
- **Fake adapter.** A stand-in for the USB library replays a recorded CarPlay session with its original timing, splitting messages across reads the way the adapter does, and logs everything the helper sends back.
- **Fake display.** A stand-in for the display driver covers the APP-layer and camera-pause tests.
- **Real Qt.** The screens load in the radio's own Qt 4.8.2 with its font, and are rendered to pictures.
- **Probe.** Run against stand-in register memory for three cases: never powers up, present, and bus error part-way.

Emulation says nothing about speed. Every performance figure here came from the radio.

## Roadmap

The remaining work is polish and integration:

1. A steady 15 fps with a moving map on every drive (bit rate first, 12 fps as a fallback).
2. Microphone for Siri and calls.
3. Touches go only to CarPlay while it is full screen.
4. Block stock pop-ups and the stock call screen during CarPlay, keeping the reverse camera, volume display and safety warnings.
5. Steering-wheel buttons.
6. Clean-up: CarPlay as an entry on the Apps page, test features behind one Developer-mode switch, dead drawing paths removed.
7. Adapter hardening: random Wi-Fi password and a pairing lock.
8. Restyle the remaining stock screens to match the new Bluetooth screen.

## Credits

- [node-CarPlay](https://github.com/rhysmorgan134/node-CarPlay) (MIT) for the adapter protocol.
- [CPC200-CCPA_resources](https://github.com/lvalen91/CPC200-CCPA_resources) for adapter firmware documentation.
- [FFmpeg](https://ffmpeg.org) 6.1.2 under the LGPL.
- Texas Instruments' public reference code for the IVA-HD decoder.
- Subaru service bulletin [15-199-16R](https://static.nhtsa.gov/odi/tsbs/2020/MC-10171930-0001.pdf) for the stock update procedure.

Clarion's stock firmware is proprietary and is not part of this project's own work. This project is not affiliated with Subaru, Clarion, Apple, Texas Instruments or Carlinkit. CarPlay is a trademark of Apple Inc.
