---
title: "ESP32-S3 SDR mode throughput improvement"
date: 2026-10-02
slug: esp32-s3-sdr-mode-throughput-improvement
description: "This is a very technical article on how we achieved a massive improvement on ESP32 SDR mode throughput complemented with measurement data"
authors: ["Zoltan Doczi"]
categories: ["R&D"]
tags: ["randd"]
aliases: ["/article/esp32-s3-sdr-mode-throughput-improvement"]
cover:
  image: SDR_front_panel.png
  alt: "ESP32-S3 SDR mode throughput improvement"
  relative: true
weight: 1
---

# Preface

An ESP32-S3 can do more than you might expect: it can act as an SDR receiver, capturing raw IQ samples on the chip, processing them, streaming them to a host, or turning them into a live spectrum display.

Curious about what's possible with single-chip SDR solutions, we started experimenting with the existing [ESP-SDR](https://github.com/ESPARGOS/esp-sdr) firmware from the [ESPARGOS](https://github.com/ESPARGOS) project. Many thanks to [Florian Euchner](https://github.com/Jeija) for the original work! We managed to push the spectrum update rate up by about **50×**, and the improvements have been merged into ESP-SDR ([commit db62f8a](https://github.com/ESPARGOS/esp-sdr/commit/db62f8a)).

Building on the same ring engine, the new [z2labs/esp-sdr-bridge](https://github.com/z2labs/esp-sdr-bridge) makes the ESP32-S3's USB IQ stream available as **SpyServer** and **rtl_tcp**, so SDR++, SDR#, gqrx and GNU Radio can receive and process real IQ from the chip. You can even tune in from an Android phone over Wi-Fi: SDR++ on the phone connects to the bridge running on the PC, while the ESP32-S3 stays on USB. Measurement results: [detailed results](https://github.com/z2labs/esp-sdr-bridge/blob/main/docs/MEASUREMENTS.md) and [PDF report](https://github.com/z2labs/esp-sdr-bridge/blob/main/docs/ESP32-S3_IQ_bridge_measurements.pdf).

The IQ-streaming firmware is open as a pull request to ESP-SDR ([ESPARGOS/esp-sdr#4](https://github.com/ESPARGOS/esp-sdr/pull/4)), so it can later be installed from the ESP-SDR web flasher. The bridge stays a separate network server, complementing Florian's upcoming SoapyESPSDR driver, and a shared stream format will let it support the ESP32-S31 as well.

In this post we walk through how we achieved the throughput improvement. The deep dive is based on a detailed technical report, backed by real measurements and aimed at RF-savvy engineers. The measurements and the report were prepared with the help of Claude.

Before diving into the details, check out the new processing pipeline in action in [video1](https://www.youtube.com/watch?v=9M-mIbKxqZ4) and [video2](https://www.youtube.com/watch?v=wqLQqpOyzJ8).

---

# ESP32-S3 Turbo Mode (SPEC): firmware, web viewer and VSG60 characterisation

> [esp-sdr](https://github.com/ESPARGOS/esp-sdr) / [esp-web-sdr](https://github.com/ESPARGOS/esp-web-sdr) (ESPARGOS) · Z2Labs · Turbo Mode developed by Zoltan Doczi from [z2labs.io](https://www.z2labs.io/)
> 

## Highlights

|  | Before | After |
| --- | --- | --- |
| Spectrum updates / s | 23 | ~1300 (**46–56×**) |
| Longest blind gap | 42 ms | 48–145 µs (**290–880×** shorter) |
| CRC errors / gaps / drops in long runs |  | **0** |
| Start/stop cycles |  | 300, **0** failures |

## 1. What SPEC mode is

The original firmware captures one burst (16 380 samples), sends the raw IQ to the PC and waits. Most of the time is spent on the link, so the receiver is blind for ~42 ms between bursts.

In SPEC mode the dump engine never stops. It runs as a **gapless ring over three SRAM banks**, and gaplessness is verified at runtime: sentinels plus a binary search, and every unit must start exactly where the previous one ended. While one bank fills, the CPU unpacks, windows (Hann), FFTs and accumulates the finished banks in short slices, polling the ring between slices so that the bank switch is never late. Only dB-coded spectra plus a CRC go to the PC, straight into the USB Serial/JTAG FIFO. Each frame carries a 64-bit sample index (the gapless clock), the number of FFTs merged, the gain and drop counters. Interrupts are masked for the whole run, a full output queue drops whole frames (counted), and if no frame is accepted for 2 s the run ends by itself.

| Command | Purpose |
| --- | --- |
| `SPEC ms stride upf det [rate [nfft]]` | stream spectra (ms 0 = until a newline); rate 0/1/6 = 80/40/16 MS/s; det 0 mean, 1 max-hold |
| `RING ms rate` / `RINGCAP units rate` | ring statistics / 1–3 gapless units of raw IQ |
| `CAPS` | advertises `RING SPEC SPECN`; the web viewer enables SPEC only when present |

The ESPARGOS web viewer shows the stream as a spectrum analyzer (Start/Center/Span/Stop/RBW bar, waterfall, auto scale, auto gain).

## 2. Why native USB and not the UART

- **Throughput.** The UART path goes through the board's USB-UART bridge (CH343) at 2 Mbaud: at most 200 kB/s. SPEC streams 310–400 kB/s at the default settings, about twice that. The native USB Serial/JTAG port was measured at ~0.77 MB/s.
- **Real-time output without interrupts.** During a run all interrupts are masked (the lateness budget is 2000 sample pairs = 125 µs at 16 MS/s). Output is written by polling straight into the USB TX FIFO from a 16 KiB queue, with no driver and no ISR.
- **Possible on UART:** fewer frames per second (e.g. 256 bins at ≤ ~600 frames/s) would fit and still be ~25× the burst rate. Not implemented yet.

## 3. Benchmark (same board, native USB, vs upstream 36be3fa)

|  | 16 MS/s | 40 MS/s | 80 MS/s |
| --- | --- | --- | --- |
| Spectrum updates / s (burst → SPEC 256) | 23.2 → 1301 (56×) | 23.4 → 1083 (46×) | 23.8 → 1296 (54×) |
| Longest blind gap | 42.1 ms → 48 µs | 42.3 ms → 70 µs | 41.8 ms → 145 µs |
| Share of RF time analysed | 2.37 % → 25 % | 0.96 % → 8.3 % | 0.49 % → 2.2 % |
| USB link used | 760 → 375 kB/s | 767 → 312 kB/s | 781 → 373 kB/s |

With the second core (merged in db62f8a) the analysed share rose further, to **50 %** at 16 MS/s, **20 %** at 40 MS/s and **10 %** at 80 MS/s (256 bins), and 1024/2048-bin FFTs became possible at 80 MS/s.

## 4. Screenshots (ESPARGOS web viewer)

![Original firmware: burst IQ, 80 MS/s, 2048-point host FFT, 16.6 frames/s](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/original_burst_80M.jpg)

*Original firmware and web app: burst IQ, 80 MS/s, 2048-point host FFT, 16.6 frames/s.*

![SPEC 80 MS/s / 256 bins: 1295 spectra/s](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/turbo_80M_256.jpg)

*SPEC, 80 MS/s / 256 bins, filter wide open: 1295 spectra/s, gapless on chip.*

![SPEC 40 MS/s / 2048 bins](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/turbo_40M_2048.jpg)

*SPEC, 40 MS/s / 2048 bins (RBW 28 kHz): 190 spectra/s.*

## 5. Robustness

- Long runs (80M/256, 40M/2048, 16M/1024, 16M/256): **0 CRC errors, 0 sequence/index gaps, 0 dropped frames.**
- 300 SPEC start/stop cycles: 0 failures; stop latency median 3.1 ms, max 9.3 ms.
- Slow reader: a 1.5 s host stall is reported as 1865 dropped frames with clean recovery; after 2 s without reading, the firmware watchdog ends the run and the device keeps answering.
- **Browser freeze, root cause:** with the tab hidden, Chrome throttles timers to 1 Hz; the web reader yielded with `setTimeout(0)`, starved, and the 2 MB queue overflowed in ~4 s. Fixed with a `MessageChannel` yield, lossy SPEC reading with resync, always stopping the device on exit, and bounded backlogs. After the fix: 30 s hidden, queue peak 1.7 kB, 0 drops.

## 6. VSG60 characterisation (relative)

Signal Hound VSG60 driven over SCPI, a whip antenna 2 cm from the S3 PCB antenna, geometry fixed with tape. At 2 cm the coupling is in the reactive near field, so all results are relative. The VSG level is capped at −50 dBm in the tooling.

![Tone vs VSG level per gain index and tone-minus-image](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/vsg_level_sweep.png)

*Level sweep, 16 MS/s / 1024, CW +4 MHz from the LO: tone vs VSG level per gain index (left), tone minus image (right).*

| Gain index | 0 | 10 | 20 | 30 | 40 | 50 | 60 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Relative gain [dB] | 0 | +10.3 | +21.1 | +20.8 | +30.2 | +41.9 | +51.3 |
| Slope [dB/dB] | 1.11 | 1.05 | 1.02 | 1.00 | 1.03 | 1.03 | 1.01 |
| Noise floor [dBFS/bin] | −80.3 | −78.5 | −77.4 | −77.5 | −73.9 | −73.0 | −68.4 |
- About 1 dB per gain step, except 20→30, which is flat (the web auto gain steps across it).
- Hard ADC clipping at ~0 dBFS with no soft compression before it; dynamic range 62–78 dB per bin.
- Index 60 is the noise optimum (floor ≈ −90.7 dBm/bin referred to the VSG).
- Image rejection ≈ 41 dB unclipped.

![Passband vs analog bandwidth, 80 MS/s](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/passband_80M.png)

*Passband at 80 MS/s / 256, CW swept ±38 MHz: default filter (BANDWIDTH 20) vs wide open.*

- Default 20 MHz filter: −3 dB at ±10 MHz, −20 dB at ±13 MHz. At 40/80 MS/s the edges of the display would only show filtered noise, so the web now opens the filter automatically there.
- Wide open: flat within ~2 dB over ±30 MHz.

![Response through DC](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/dc_response_cw.png)

*CW at −40 dBm stepped over ±7.5 MHz in 100 kHz steps (16 MS/s / 1024), RF-off power subtracted.*

The CW level is flat within ±0.5 dB across ±7.5 MHz, including 0 Hz: there is no DC notch for a signal. The DC bin itself (zero-IF DC offset and LO leakage) sat 35–45 dB above the floor in this report; later firmware subtracts a slowly tracked DC estimate on chip.

![DC spike before the fix](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/dc_spike_before.jpg)

*Before the DC handling: the zero-IF DC spike at the centre of a 40 MS/s view.*

## 7. Scheduling check: no late bank switches

A bank switch later than 2000 sample pairs cannot be verified as gapless, so the run stops with status LATE. Every rate / FFT size in the web viewer was checked 8 times with both detectors and RF off. Only 16 MS/s / 1024 with the mean detector at stride 4–5 aborted; the web now uses stride 6 there (100 % of FFTs done, late max 176 pairs). The full matrix is in the report.

## 8. The 0 Hz artifact: an esp-dsp bug

Weak signals showed a one-sided noise ramp with a step at the centre of the spectrum. It was visible with RF off at every LO and gain, so it was a noise-floor effect, not a gain notch.

> **Root cause:** the ESP32-S3 SIMD FFT in Espressif's esp-dsp library (`dsps_fft2r_sc16_aes3`) truncates the scaling and the twiddle products separately. The sum branch of every butterfly is biased by about −1 LSB per stage, the difference branch is not.
> 

A bit-exact model reproduces it (8.8 dB asymmetry and a 15.7 dB step at DC for a 2048-point FFT). Adding one rounding step to the sum branch in all three loops removes it completely (step 0.0 dB, left–right within ±0.5 dB). We reported it upstream as [espressif/esp-dsp#123](https://github.com/espressif/esp-dsp/issues/123); the firmware ships the fixed kernel until it is merged.

![Bit-exact model: esp-dsp aes3 vs proposed fix](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/espdsp_model.png)

*Bit-exact model, N = 2048, complex noise of 1 LSB: the current esp-dsp kernel (red) vs the kernel with the rounding fix (blue) and a float FFT reference (grey).*

![Before the fix](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/espdsp_before.jpg)

*On the chip, before the fix: 40 MS/s / 2048 bins, ramp on one side and an abrupt step at the LO.*

![After the fix](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/blog/espdsp_after.jpg)

*After the fix: the same view without the ramp and the step at the LO.*

## 9. Still open

- **Mean vs max detector:** only the mean detector overran when overloaded (section 7). It is avoided by the stride choice, but the root cause is not found yet.
- **Coverage at 40 MS/s / 2048:** the stream is gapless, but a 2048-point FFT barely fits between bank switches, so many blocks are skipped.
- **Wideband OFDM tests** (Wi-Fi-like and LTE-like ARB files) need more level or a conducted path to give clean results.
- **Absolute calibration** (dBm, noise figure) needs a conducted path: a pigtail at the antenna feed, or a module with a U.FL connector.
- **Scope:** tested on the ESP32-S3; the ESP32-S31 is untested. SPEC over UART is not implemented.

---

# Real IQ into SDR++ and SDR#: esp-sdr-bridge

The next step was real IQ instead of spectra. A two-stage decimating FIR (DDC) on the S3's second core, written with the PIE SIMD instructions, turns the 16 MS/s ring into a gapless **250 / 125 / 62.5 kS/s** complex stream that fits through the USB Serial/JTAG port. The LO is tuned 4 MHz below the wanted band and the chip shifts the samples by +fs/4 before filtering, so the LO leakage and the 1/f hump fall outside the output band. The bridge on the PC serves the stream as SpyServer and rtl_tcp.

![SDR++ through the SpyServer bridge: 50 ksym/s QAM16 at 2456 MHz](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/sdrpp_qam16.jpg)

*SDR++ connected to the SpyServer bridge (250 kS/s, int16), receiving a 50 ksym/s QAM16 signal from the VSG60 at 2456 MHz.*

## ~90-minute soak test

2350 MHz, CW −50 dBm 40 kHz above, int16 at 250 kS/s, with a chaos block every 5 minutes: random hops across 2210–2790 MHz, rate and format switches, gain changes, client reconnects. The test client sends exactly the SpyServer commands SDR++ sends.

![Soak test](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/soak.png)

- **~2.5 million frames, 0 CRC errors, 0 SpyServer sequence gaps.** The firmware flagged 8 dropped-frame events (0.003 % of the samples) when the PC briefly stopped reading USB; every gap is visible to the host from the 64-bit sample index.
- Tone SNR **58 dB** per 30 Hz bin, stable over the whole run; 61 dB at 125 kS/s and 65 dB at 62.5 kS/s.
- Frequency error: a constant **−1.07 ppm** crystal offset of this board after a 0.18 ppm warm-up.

## 1 MHz CW sweep

![CW sweep, 1 MHz steps](https://raw.githubusercontent.com/z2labs/esp-sdr-bridge/main/docs/cw_sweep_1mhz.png)

- SNR 57.5 dB from 2.2 to 2.4 GHz; the receive gain falls smoothly by ~17 dB towards 2.65 GHz and the SNR follows it.
- The dips in 2.40–2.50 GHz are live Wi-Fi and Bluetooth traffic in the lab.
- Tuning in 1 kHz steps; the residual error is a ±0.1 ppm sawtooth from the PLL's fractional resolution.

## Try it

- Firmware: [ESPARGOS/esp-sdr#4](https://github.com/ESPARGOS/esp-sdr/pull/4) (branch [s3-iq-stream](https://github.com/zodoczi/esp-sdr/tree/s3-iq-stream))
- Bridge: [z2labs/esp-sdr-bridge](https://github.com/z2labs/esp-sdr-bridge), then in SDR++: Source *SpyServer*, `localhost:5555`, Int16
- Full measurement report: [PDF](https://github.com/z2labs/esp-sdr-bridge/blob/main/docs/ESP32-S3_IQ_bridge_measurements.pdf)
