# Software-Defined Radio: HackRF One and GNU Radio

A software-defined radio (SDR) turns a slice of the radio spectrum into a stream of numbers, IQ samples, and leaves the rest of the radio to software: filtering, demodulation, decoding, and the reverse path for transmission. This note covers what happens between the antenna and those samples, what the samples mean, and how to choose and drive SDR hardware from Linux. It then builds receivers and transmitters with GNU Radio. The example device is a HackRF One on a Debian host. An RTL-SDR (receive only, cheap) and an ADALM-Pluto (full duplex, runs embedded Linux itself) are the points of comparison.

The protocol notes ([Wi-Fi](../../wireless-protocols-and-connectivity/wifi/wifi.md), [cellular](../../wireless-protocols-and-connectivity/cellular/cellular.md)) describe what a radio sends; this note is about the tools that let you see it and generate it. The theory behind it gets only the minimum needed here: signal processing, modulation and coding, and RF design each have their own folder ([signal processing](../signal-processing/), [baseband](../baseband/), [RF](../rf/)). The embedded Linux side (Raspberry Pi 4, BeagleBone Black, the Pluto's Buildroot firmware) links back to the [embedded Linux fundamentals](../../embedded-linux/).

**Assumptions:**
- The SDR is a HackRF One (hardware revision r9 or older), with firmware and host tools from release v2026.01.3. It is driven over USB from an x86_64 laptop running Debian 13 (trixie), with GNU Radio 3.10 from the Debian packages.
- The receive-only comparison is an RTL-SDR Blog V4; the full-duplex comparison is an ADALM-Pluto with firmware v0.39.
- The embedded hosts are a Raspberry Pi 4 (Raspberry Pi OS Lite 64-bit) and the BeagleBone Black of the embedded Linux notes.
- Frequencies of local transmitters (an FM station, a 433 MHz sensor, LTE cells) are placeholders: use the ones around you.
- Regulations are given for the EU and the US. They change, and other regions differ.

**Conventions:**
- `$` is a shell on the host, `#` a root shell on an embedded board.
- **Expected** marks console output reconstructed from the tools' documentation and source, not captured. Serial numbers, rates and levels in it are placeholders.
- **Typical** marks figures given as orders of magnitude, not measured for this note.
- **Rule of thumb** marks practical advice that depends on the setup and the environment.
- Every command is generic: it follows the documentation of the tool it calls and was not run for this note.

**Verification status:** *Planned.*

**How to read it:**
- **To learn:** Parts 1 to 3 in order. They go from "what is an IQ sample" to a working FM receiver and a decoder for a 433 MHz sensor.
- **To choose hardware:** sections 4 and 5.
- **For a first signal in ten minutes:** sections 6 and 7.
- **To build with GNU Radio:** Part 3.
- **To run an SDR on an embedded board or in a product:** Part 4.
- **When it doesn't work:** section 19.

## Table of contents

**Part 1: What an SDR is**

- [1. The mental model: from the antenna to IQ samples](#1-the-mental-model-from-the-antenna-to-iq-samples)
- [2. The example setup: hardware, software, signals and rules](#2-the-example-setup-hardware-software-signals-and-rules)
- [3. IQ samples: the minimum theory](#3-iq-samples-the-minimum-theory)
- [4. SDR hardware compared: RTL-SDR, HackRF, Pluto, USRP and others](#4-sdr-hardware-compared-rtl-sdr-hackrf-pluto-usrp-and-others)

**Part 2: The HackRF One**

- [5. Inside the HackRF One](#5-inside-the-hackrf-one)
- [6. Installing the host software and updating the firmware](#6-installing-the-host-software-and-updating-the-firmware)
- [7. First reception with the command-line tools](#7-first-reception-with-the-command-line-tools)
- [8. Gain, overload and calibration](#8-gain-overload-and-calibration)

**Part 3: GNU Radio**

- [9. GNU Radio concepts: flowgraphs, blocks, streams, tags and messages](#9-gnu-radio-concepts-flowgraphs-blocks-streams-tags-and-messages)
- [10. A first flowgraph: an FM broadcast receiver](#10-a-first-flowgraph-an-fm-broadcast-receiver)
- [11. Recordings: file formats, SigMF and offline processing](#11-recordings-file-formats-sigmf-and-offline-processing)
- [12. Digital signals: decoding a 433 MHz sensor](#12-digital-signals-decoding-a-433-mhz-sensor)
- [13. Writing your own blocks](#13-writing-your-own-blocks)
- [14. Transmitting safely and legally](#14-transmitting-safely-and-legally)

**Part 4: SDR in a system**

- [15. Other drivers and APIs: SoapySDR, UHD, libiio and librtlsdr](#15-other-drivers-and-apis-soapysdr-uhd-libiio-and-librtlsdr)
- [16. SDR on embedded Linux: Raspberry Pi 4, BeagleBone Black and the Pluto](#16-sdr-on-embedded-linux-raspberry-pi-4-beaglebone-black-and-the-pluto)
- [17. Throughput, timing and synchronization](#17-throughput-timing-and-synchronization)
- [18. Applications: aircraft, sensors, satellites, GNSS and cellular](#18-applications-aircraft-sensors-satellites-gnss-and-cellular)

**Part 5: Troubleshooting**

- [19. Troubleshooting by symptom](#19-troubleshooting-by-symptom)
- [20. Not covered here](#20-not-covered-here)
- [References](#references)

## 1. The mental model: from the antenna to IQ samples

A conventional radio (a Wi-Fi chip, an FM receiver, a cellular modem) does its filtering, demodulation and decoding in dedicated hardware built for one standard. A **software-defined radio** keeps only the part that software can't do in analog hardware: filter, amplify, shift the frequency, and convert between analog and digital. Everything after the analog-to-digital converter (ADC) is code, so the same device can receive FM broadcast, aircraft transponders or an LTE cell, depending on the program that reads the samples.

**The receive chain:**

```
            analog, at RF                         analog, at baseband      digital
 antenna ─> RF filter ─> LNA ─> mixer ─────────> baseband filter ─> ADC ─> FPGA / MCU ─> USB ─> host
                                  ▲               (removes what       (I and Q)  (packing,           driver ─> application
                                  │                would alias)                   decimation)                  (GNU Radio)
                         local oscillator (LO) at fc,
                         locked to a reference clock
```

- The **LNA** (low-noise amplifier) raises weak signals above the noise of the stages after it.
- The **mixer** multiplies the signal by the **local oscillator** at the center frequency `fc`. The band around `fc` moves down to 0 Hz, which is called **baseband**.
- The **baseband filter** removes everything outside the band the ADC can represent (section 3).
- The **ADC** samples two signals, I and Q (below), at the **sample rate** `fs`.
- The device's FPGA or microcontroller packs the samples and sends them to the host over USB or Ethernet.

**The transmit chain** is the same in reverse: the host sends IQ samples, a DAC converts them to analog, the mixer moves them up to `fc`, and a power amplifier (PA) and a filter drive the antenna.

**Three receiver architectures:**

| Architecture | How | Typical artifacts | Examples |
|---|---|---|---|
| **Zero-IF** (direct conversion) | One mixer moves the band straight to 0 Hz | A spike at the center (DC offset, LO leakage); a mirror image from I/Q imbalance (section 8) | AD936x (Pluto, USRP B2xx, bladeRF 2.0), LMS7002M (LimeSDR), the MAX2837/MAX2839 stage of the HackRF |
| **Superheterodyne / low IF** | One or more mixers move the band to an intermediate frequency (IF), then a final step (often digital) brings it to 0 Hz | Images and spurs at frequencies set by the IF; usually a clean center | RTL-SDR (R820T/R828D tuner, low IF, digital downconversion in the RTL2832U), the HackRF's first mixer stage |
| **Direct sampling** | The ADC samples the RF signal itself; the downconversion is all digital | Aliases of strong signals from other Nyquist zones | RFSoC boards, many HF receivers, the RTL-SDR's HF direct-sampling mode |

**What the host receives.** A continuous stream of complex numbers, `fs` of them per second, describing the band from `fc - fs/2` to `fc + fs/2`. With the HackRF tuned to 100 MHz at 10 Msps (mega-samples per second), the stream covers 95 to 105 MHz. A transmitter at 101 MHz shows up as a component rotating at +1 MHz, and one at 99 MHz as a component rotating at -1 MHz.

**Why the samples are complex.** After a single mixer, 101 MHz and 99 MHz both land at 1 MHz: a real signal can't tell above `fc` from below `fc`. An SDR uses two mixers whose oscillators are 90° apart (cosine and sine) and samples both outputs: **I** (in phase) and **Q** (quadrature). The pair `I + jQ` is one complex sample, holding an amplitude and a phase, and the sign of the rotation tells the two sides of `fc` apart. Section 3 has the minimum math.

**What the software does** with those samples, in order: shift the wanted signal to 0 Hz (multiply by a complex exponential), low-pass filter it, **decimate** (keep one sample in M to lower the rate), then demodulate and decode. Section 10 does exactly this for FM.

**Hardware limits are final.** Software can't recover what the front end never captured:

| Decided by the hardware | Decided by the software |
|---|---|
| Frequency range, tuning step, LO phase noise | Modulation, demodulation, decoding |
| Instantaneous bandwidth (the sample rate) | Channel selection inside that bandwidth |
| Noise figure (sensitivity) | Filtering, synchronization, equalization |
| Dynamic range: ADC bits, gain stages, filters (section 8) | Protocol logic, timing above the sample level |
| Clock accuracy and stability | Frequency correction (once the error is measured) |
| Half or full duplex, number of channels, TX power | What is transmitted |

## 2. The example setup: hardware, software, signals and rules

| Element | Value |
|---|---|
| SDR | HackRF One, hardware revision r9 (older revisions behave the same for this note), firmware v2026.01.3 |
| Antenna | The telescopic ANT500 (75 MHz to 1 GHz) sold with it; any 50 Ω antenna with an SMA plug for the band you use |
| Host | x86_64 laptop, Debian 13 (trixie), a USB 2.0 or 3.x port directly on the machine |
| Host software | `hackrf` v2026.01.3 tools and libhackrf, GNU Radio 3.10 (3.10.12.0 in Debian), gr-osmosdr, SoapySDR with its HackRF module, GQRX, inspectrum, rtl_433, Universal Radio Hacker (section 6) |
| Receive-only comparison | RTL-SDR Blog V4 (RTL2832U and R828D) |
| Full-duplex comparison | ADALM-Pluto (AD9363 and Zynq-7010), firmware v0.39 |
| Embedded hosts | Raspberry Pi 4 with Raspberry Pi OS Lite 64-bit; the BeagleBone Black of the [embedded Linux notes](../../embedded-linux/) (section 16) |

**Lab accessories** worth having before section 8 and required for section 14:

| Item | Why |
|---|---|
| SMA attenuators: 10, 20 and 30 dB, rated for 1 W or more | Connect a transmitter to a receiver by cable without damaging the receiver; tame strong signals |
| SMA cables and adapters | Cable tests instead of over-the-air ones |
| 50 Ω SMA terminator (dummy load) | Transmit into nothing |
| FM broadcast band-stop filter (88 to 108 MHz) | Strong FM stations overload wideband receivers (section 8) |
| A band-pass or low-pass filter for each transmit band you use | The HackRF has no output filtering for harmonics (section 14) |
| DC block | Protects the input from DC on an antenna feed |
| Optional: a 10 MHz reference (GPS-disciplined oscillator) | Accurate frequency for GNSS and cellular work (sections 8 and 17) |

**The example signals:**

| Signal | Frequency | Used for |
|---|---|---|
| An FM broadcast station | 87.5 to 108 MHz; `98.0 MHz` here, a placeholder | The first recordings (section 7) and the first flowgraph (section 10): strong, continuous, audible |
| A 433 MHz sensor you own (weather station, doorbell) | 433.92 MHz, in the EU SRD band 433.05 to 434.79 MHz | A simple digital signal (section 12) |
| ADS-B transponders | 1090 MHz | Short digital packets (section 18) |
| An LTE cell | Depends on the band; see the [cellular note](../../wireless-protocols-and-connectivity/cellular/cellular.md#5-spectrum-bands-duplexing-and-channel-numbers) | Frequency calibration (section 8), cell scanning (section 18) |
| GPS L1 C/A | 1575.42 MHz | A signal below the noise floor (section 18) |
| Your own test transmissions | 433.92 MHz, into a cable | Section 14 |

The commands use these variables:

```sh
$ export SDR=~/sdr                 # working directory: flowgraphs and scripts
$ export REC=$SDR/rec              # recordings
$ export FM_HZ=98000000            # a local FM station (placeholder)
$ export ISM_HZ=433920000          # the 433 MHz sensor
$ mkdir -p $REC
```

**Rules followed in this note.** Radio law differs by country; these are the common ground, not legal advice:

- **Receiving** broadcast, amateur radio, aircraft and ship position reports, weather satellites and your own devices is legal in most countries. **Intercepting, using or disclosing** communications not meant for you is not: EU member states' interception laws, the US Electronic Communications Privacy Act (18 U.S.C. 2511), and similar laws elsewhere. Some countries restrict even listening to certain bands. Cellular user data is encrypted anyway; this note only decodes broadcast cell information.
- **Transmitting** needs an authorization: an amateur radio licence (its bands, power and modes), a licence-exempt band used within its rules (in the EU, ERC Recommendation 70-03; for 433.05 to 434.79 MHz that's 10 mW e.r.p. with a duty-cycle limit), or a cable. In the US, FCC Part 15 licence-exempt operation assumes certified equipment, which an SDR running your own waveform isn't.
- So every transmission in this note goes **into a cable with attenuators or into a dummy load** (section 14). Over-the-air transmission is mentioned only with a licence. Transmitting on GNSS, aviation, cellular or emergency frequencies, or jamming anything, is illegal everywhere and can endanger people.

## 3. IQ samples: the minimum theory

This is the theory the rest of the note uses, stated without proof. The [signal processing](../signal-processing/) and [baseband](../baseband/) folders go deeper.

**A complex sample** `x[n] = I[n] + jQ[n]` is a point in the plane: amplitude `A = sqrt(I² + Q²)`, phase `φ = atan2(Q, I)`. A transmitter at an offset `f` from the center frequency (positive above `fc`, negative below) produces samples that rotate at `f` turns per second:

```
x[n] = A · e^(j·2π·f·n / fs)        f > 0: counter-clockwise   f < 0: clockwise
```

**Sample rate is bandwidth.** Complex samples at `fs` represent every frequency from `-fs/2` to `+fs/2`, so the **instantaneous bandwidth** is `fs`. Real samples at the same rate represent only `0` to `fs/2`. That's why an SDR at 10 Msps shows 10 MHz of spectrum.

**Aliasing.** Anything that reaches the ADC from outside `±fs/2` folds back into the band: a signal at `fs/2 + d` appears at `-fs/2 + d`, mixed with the real signals there. The analog baseband filter has to remove it first, and no filter is a wall. The **edges** of the captured band, the outer 10 to 20%, are never trustworthy (**rule of thumb**: count on about 80% of `fs`).

**Shifting, filtering and decimating** are the three operations every receiver uses:

| Operation | How | Effect |
|---|---|---|
| Frequency shift by `-f0` | Multiply by `e^(-j·2π·f0·n / fs)` | The signal at `+f0` moves to 0 Hz |
| Low-pass filter | FIR filter (a weighted sum over the last N samples) | Removes everything outside the wanted channel |
| Decimate by M | Keep one sample in M, after the filter | New rate `fs/M`, M times less work for everything after it |

GNU Radio's *Frequency Xlating FIR Filter* does all three in one block (section 10). Decimating after a filter also improves the **signal-to-noise ratio** (SNR) of a narrow signal: the noise outside the new band is gone. This **processing gain** is `10·log10(M)` dB (13 dB for M = 20).

**The FFT** turns N samples into N frequency **bins**, each `fs/N` wide: 8 Msps and a 4096-point FFT give 1953 Hz per bin. A spectrum display shows those bins from `fc - fs/2` to `fc + fs/2`. With a bigger FFT, the bins narrow and the **noise floor drops** by 3 dB per doubling of N, while a pure tone stays at the same level. A displayed noise floor therefore means nothing without the bin width.

**Decibels:**

| Unit | Meaning | Values to remember |
|---|---|---|
| dB | A power ratio: `10·log10(P1/P2)` | +3 dB = ×2, +10 dB = ×10, +20 dB = ×100 |
| dBm | Power relative to 1 mW | 0 dBm = 1 mW, +30 dBm = 1 W, -30 dBm = 1 µW; 0 dBm into 50 Ω is 0.224 V rms |
| dBFS | Relative to the ADC's full scale | 0 dBFS is the largest value the ADC can output |

An SDR reports **dBFS**, not dBm. The level in dBm depends on every gain stage, the frequency and the temperature; turning one into the other needs a calibration against a signal generator of known level at that frequency and gain setting.

**Noise.** The thermal noise of a 50 Ω input at room temperature is **-174 dBm/Hz**. In a bandwidth B, it is `-174 + 10·log10(B)` dBm: -104 dBm in 10 MHz, -134 dBm in 10 kHz. The receiver adds its **noise figure** (NF, a few dB to 15 dB for wideband SDRs) to that. A signal is decodable when its SNR in its own bandwidth is above what its modulation needs, which is why narrow signals can be received at lower levels.

**ADC bits and dynamic range.** An ideal N-bit ADC has an SNR of `6.02·N + 1.76` dB across its full band:

| ADC | Ideal SNR | Devices |
|---|---|---|
| 8 bits | 50 dB | RTL-SDR, HackRF One |
| 12 bits | 74 dB | Pluto, LimeSDR, bladeRF, USRP B2xx, Airspy |
| 14 bits | 86 dB | SDRplay RSP, many high-end receivers |

Real ADCs reach less (their effective number of bits, ENOB). **Dynamic range** is the span between the strongest signal the receiver tolerates and the weakest it still sees at the same time. The gain must be low enough that the strongest signal in the band doesn't clip the ADC, and a weak signal then has to rise above the quantization noise. In a city full of strong transmitters, an 8-bit receiver loses weak signals that a 12- or 14-bit one keeps. Filtering out strong signals before the ADC, and decimating to a narrow band, both help (section 8).

## 4. SDR hardware compared: RTL-SDR, HackRF, Pluto, USRP and others

| Device | Frequency range | Max bandwidth | ADC bits | TX | Channels, duplex | Interface | Host driver | Price class |
|---|---|---|---|---|---|---|---|---|
| RTL-SDR Blog V4 | 0.5 MHz to 1.766 GHz (HF through a built-in upconverter) | 2.4 Msps stable, 3.2 Msps max | 8 | No | 1 RX | USB 2.0 | librtlsdr | Tens of euros |
| Airspy R2 / Mini | 24 MHz to 1.8 GHz | 10 MHz (R2), 6 MHz (Mini) | 12 | No | 1 RX | USB 2.0 | libairspy | 100 to 200 € |
| SDRplay RSP1B, RSPdx-R2 | 1 kHz to 2 GHz | 10 MHz | 14 | No | 1 RX (2 on the RSPduo) | USB 2.0 | SDRplay API (closed source) | 100 to 300 € |
| **HackRF One** | 1 MHz to 6 GHz | 20 Msps | 8 | Yes, up to about 15 dBm | 1, half duplex | USB 2.0 | libhackrf | 300 € class |
| ADALM-Pluto | 325 MHz to 3.8 GHz (70 MHz to 6 GHz with an unsupported setting, section 16) | 20 MHz (AD9363); about 4 Msps sustained over USB | 12 | Yes, a few dBm | 1 TX + 1 RX, full duplex (2 + 2 on revisions C and D) | USB 2.0, runs Linux itself | libiio | 200 € class |
| LimeSDR Mini 2.0 / LimeSDR | About 10 MHz to 3.5 GHz | 30.72 to 61.44 Msps depending on the model | 12 | Yes | 1×1 or 2×2, full duplex | USB 3.0 | LimeSuite, SoapySDR | 300 to 600 € |
| bladeRF 2.0 micro | 47 MHz to 6 GHz | 56 MHz | 12 | Yes | 2×2, full duplex | USB 3.0 | libbladeRF | 500 to 900 € |
| USRP B200 / B210 | 70 MHz to 6 GHz | 56 MHz | 12 | Yes | 1×1 / 2×2, full duplex | USB 3.0 | UHD | 1500 to 2500 € |
| USRP N3xx, X410, RFSoC boards | Up to 6 GHz and beyond | 100 to 400 MHz and more | 12 to 14 | Yes | 2 to 8 and more | 10/100 GbE, PCIe | UHD, vendor tools | Thousands of euros |

Ranges and prices are **typical** and change with revisions and vendors; check the vendor's page before buying. Great Scott Gadgets' newer **HackRF Pro**, supported by the same host tools since release v2026.01.1, isn't covered here.

**How to choose**, by question:

| Question | If yes |
|---|---|
| Only receiving, below 1.7 GHz, signals narrower than 2 MHz? | RTL-SDR. Start here even if you'll buy something else later |
| Receiving weak signals next to strong ones (HF, VHF in a city)? | More ADC bits and better filtering: SDRplay, Airspy |
| Transmitting, or a wide frequency range, on a budget? | HackRF One. Half duplex and 8 bits are its limits |
| Full duplex (a two-way protocol, a cellular base station or UE stack)? | Pluto for learning; bladeRF 2.0, USRP B210 or LimeSDR for real work |
| Running code close to the radio, on its FPGA or CPU? | Pluto (Zynq, runs Linux), USRP E-series, RFSoC boards |
| Accurate frequency and time (GNSS, LTE, TDD timing)? | A device with a TCXO and a 10 MHz / PPS input; a GPS-disciplined reference |
| An embedded host with USB 2.0 and a small CPU? | Low sample rates; RTL-SDR or a Pluto doing part of the work on board (section 16) |
| Will the software you need support it? | Check the drivers: srsRAN and OpenAirInterface favor UHD; most hobby tools support SoapySDR (section 15) |

This note uses the HackRF One because it covers both directions and the widest frequency range at a low price, and because its tools and firmware are entirely open source. Where its limits matter (8 bits, half duplex, no output filter, USB 2.0), the note says so and names the alternative.

## 5. Inside the HackRF One

*Planned: The block diagram, the frequency plan with the mixer and the IF, the gain stages, the baseband filter, the clock, half-duplex operation, the firmware and the CPLD, limits and the HackRF Pro.*

## 6. Installing the host software and updating the firmware

*Planned: Debian packages, USB permissions, hackrf_info, firmware updates and DFU recovery, SoapySDR and gr-osmosdr, GUI receivers.*

## 7. First reception with the command-line tools

*Planned: hackrf_transfer to a file, the cs8 format, hackrf_sweep, looking at a recording with inspectrum, GQRX and SDR++.*

## 8. Gain, overload and calibration

*Planned: Gain staging, ADC clipping and front-end overload, the DC spike and offset tuning, IQ imbalance and images, frequency error in ppm and how to measure it, external references.*

## 9. GNU Radio concepts: flowgraphs, blocks, streams, tags and messages

*Planned: Flowgraphs and blocks, items and sample rates, sync, decimating and interpolating blocks, the scheduler and buffers, tags, messages and PDUs, GNU Radio Companion and the generated Python, GNU Radio 3.10 and 4.0.*

## 10. A first flowgraph: an FM broadcast receiver

*Planned: The FM receiver in GNU Radio Companion block by block, the rate plan, the same flowgraph as headless Python, overruns and audio underruns.*

## 11. Recordings: file formats, SigMF and offline processing

*Planned: Sample formats of the common devices, converting between them, SigMF metadata, reading recordings in NumPy, replaying a recording through a flowgraph, Throttle and Head.*

## 12. Digital signals: decoding a 433 MHz sensor

*Planned: Finding the signal, OOK demodulation in GNU Radio, reading pulse widths, line codes, FSK and PSK receivers in outline, URH and rtl_433 as references.*

## 13. Writing your own blocks

*Planned: The Embedded Python Block, out-of-tree modules with gr_modtool, a C++ block and its work function, bindings, VOLK, installing and debugging.*

## 14. Transmitting safely and legally

*Planned: The rules, testing into cables and attenuators, the HackRF transmit chain and its levels, hackrf_transfer and a GNU Radio transmitter, harmonics and filtering, half duplex.*

## 15. Other drivers and APIs: SoapySDR, UHD, libiio and librtlsdr

*Planned: The driver layers, SoapySDR and its utilities, UHD and USRPs, libiio and the Pluto, librtlsdr and the DVB driver, network streaming, choosing an API for a product.*

## 16. SDR on embedded Linux: Raspberry Pi 4, BeagleBone Black and the Pluto

*Planned: USB and CPU budgets on small boards, decimating early, VOLK profiles, a headless receiver as a systemd service, Buildroot and Yocto, the Pluto as an embedded Linux SDR, FPGA-based SDRs.*

## 17. Throughput, timing and synchronization

*Planned: USB throughput arithmetic, overruns and underruns, buffers and latency, external clocks, triggers and multi-device synchronization, timestamps.*

## 18. Applications: aircraft, sensors, satellites, GNSS and cellular

*Planned: ADS-B, ISM sensors, weather satellites, spectrum surveys with hackrf_sweep, GNSS with gnss-sdr, LTE cell scanning and open-source cellular stacks, what each needs from the hardware.*

## 19. Troubleshooting by symptom

*Planned: Symptom tables for the device and USB, the signal, GNU Radio, transmission and the other devices.*

## 20. Not covered here

*Planned: Topics left to other notes.*

## References

*Planned.*
