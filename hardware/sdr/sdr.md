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

The HackRF One is a two-stage design: a wideband mixer brings any frequency from 1 MHz to 6 GHz to an intermediate frequency (IF) around 2.5 GHz, and a zero-IF transceiver chip converts that IF to baseband I and Q.

```
 ANTENNA (SMA) ─ RF switches ─┬─ RX amplifier (on/off) ─┐
                              └─ TX amplifier (on/off) ─┤
                                                        │  frequency plan (approximate):
                       ┌─ low-pass path, below ~2.15 GHz: RFFC5072 mixer, up to the IF
         RF filters ───┼─ bypass, ~2.15 to 2.75 GHz:      straight to the transceiver
                       └─ high-pass path, above ~2.75 GHz: RFFC5072 mixer, down to the IF
                                                        │
       MAX2839 (r9) or MAX2837 transceiver, 2.3 to 2.7 GHz:
         RX: LNA ─ quadrature mixer ─ baseband filter ─ VGA ─┐
         TX: VGA ─ baseband filter ─ quadrature mixer ───────┤
                                                             │
       MAX5864 ADC / DAC (8 bits, I and Q) ─ CPLD ─ LPC4320 microcontroller ─ USB 2.0 high speed ─ host

       Si5351C clock generator: on-board crystal, or a 10 MHz reference on CLKIN
         ─> sample clock, mixer and transceiver references, CLKOUT
```

**Gain stages.** Each tool names them differently; this table maps the names:

| Stage | Direction | Range | `hackrf_transfer` | SoapySDR | gr-osmosdr |
|---|---|---|---|---|---|
| RF amplifier | RX and TX | Off, or on (about 11 to 14 dB) | `-a 0` / `-a 1` | `AMP` | RF gain (0 or 14) |
| LNA (in the transceiver, at IF) | RX | 0 to 40 dB in 8 dB steps | `-l` | `LNA` | IF gain |
| Baseband VGA | RX | 0 to 62 dB in 2 dB steps | `-g` | `VGA` | BB gain |
| TX VGA | TX | 0 to 47 dB in 1 dB steps | `-x` | `VGA` | IF gain |

**Sample rate and filter.** The sample rate goes from 2 to 20 Msps. The HackRF documentation recommends **8 Msps or more**: the ADC is specified from 8 Msps up, and the baseband filter's narrowest setting (1.75 MHz) can't prevent aliasing at lower rates. To work on a narrow signal, sample at 8 Msps or more and decimate in software (section 10). The **baseband filter** is programmable from 1.75 to 28 MHz. The tools set it to about three quarters of the sample rate unless you set it yourself (`-b`).

**Limits to respect:**

| Limit | Value | Consequence |
|---|---|---|
| Maximum input power | **-5 dBm** | Above it, the front end can be permanently damaged. Never connect a transmitter, including another HackRF, without attenuators (section 14) |
| TX output power | **Typical**: up to about 10 to 15 dBm below 2.7 GHz, falling to about 0 dBm at 4 GHz and below 0 dBm near 6 GHz | Varies with frequency and unit; measure it if it matters |
| Duplex | Half: one antenna port, receive or transmit, not both | No two-way real-time protocols (cellular stacks need full duplex) |
| Resolution | 8 bits | About 48 dB of dynamic range in the full band (section 3) |
| Frequency accuracy | An on-board crystal, **typically** within ±10 to 20 ppm | Tens of kHz of error at GHz frequencies until calibrated (section 8) |
| Antenna port power | 3.3 V, up to 50 mA, when enabled (`-p 1`) | Powers active antennas and LNAs; can damage equipment that shorts DC. Off by default |

**Clocks.** CLKIN accepts a 10 MHz reference (check the hardware documentation for the level), and the HackRF switches to it when it detects one. CLKOUT outputs 10 MHz to drive another device. A header on the board carries a trigger signal to start several HackRFs together (section 17).

**Firmware.** The LPC4320 runs firmware stored in SPI flash. It configures the chips, moves samples between the CPLD and USB, and implements the sweep mode. The CPLD bitstream is part of the firmware image and is loaded at boot by current releases, so one file updates both. The host library (libhackrf) and the firmware talk through a versioned USB API: use the **same release on both sides** (section 6).

**USB identities:** `1d50:6089` in normal operation, `1fc9:000c` (the NXP boot ROM) in DFU recovery mode.

**Add-ons not covered here:** the Opera Cake antenna switch, and the PortaPack screen with its community firmware (Mayhem), which turn the HackRF into a stand-alone device.

## 6. Installing the host software and updating the firmware

**Packages:**

```sh
$ sudo apt install hackrf libhackrf-dev \
      gnuradio gnuradio-dev gr-osmosdr \
      soapysdr-tools soapysdr-module-hackrf \
      gqrx-sdr inspectrum rtl-433 python3-numpy python3-matplotlib
$ apt policy hackrf gnuradio
```

| Package | What it brings |
|---|---|
| `hackrf`, `libhackrf-dev` | `hackrf_info`, `hackrf_transfer`, `hackrf_sweep`, `hackrf_spiflash`, `hackrf_clock`, `hackrf_debug`; the library and its header |
| `gnuradio`, `gnuradio-dev` | GNU Radio 3.10, GNU Radio Companion (`gnuradio-companion`), `gr_modtool` and the development headers (section 13) |
| `gr-osmosdr` | The *osmocom Source/Sink* blocks: one block for HackRF, RTL-SDR, bladeRF, USRP and others |
| `soapysdr-tools`, `soapysdr-module-hackrf` | `SoapySDRUtil` and the HackRF module for SoapySDR, used by GNU Radio's own *Soapy* blocks and by many applications (section 15) |
| `gqrx-sdr`, `inspectrum`, `rtl-433` | A GUI receiver, a recording viewer, a decoder for ISM-band devices |

**Versions.** `apt policy` shows what Debian provides. The host tools and the firmware should come from the same HackRF release. If the packaged tools are older than the firmware you want, either flash the firmware of the packaged release, or build the host tools from the release tarball:

```sh
$ cd $SDR
$ wget https://github.com/greatscottgadgets/hackrf/releases/download/v2026.01.3/hackrf-2026.01.3.tar.xz
$ tar xf hackrf-2026.01.3.tar.xz
$ cd hackrf-2026.01.3/host
$ cmake -B build -DCMAKE_INSTALL_PREFIX=/usr/local
$ cmake --build build -j$(nproc)
$ sudo cmake --install build && sudo ldconfig
```

Remove the Debian `hackrf` package first, or check with `which hackrf_info` and `ldd $(which hackrf_info)` which copy runs: two libhackrf versions on one system cause API mismatches.

**USB permissions.** The package installs a udev rule that gives the `plugdev` group access to the device. Add yourself to the group, then log out and in again:

```sh
$ sudo usermod -aG plugdev $USER
$ lsusb | grep -i hackrf
```

**Expected:**

```
Bus 001 Device 007: ID 1d50:6089 OpenMoko, Inc. Great Scott Gadgets HackRF One SDR
```

**hackrf_info:**

```sh
$ hackrf_info
```

**Expected** (the serial number and part ID are placeholders):

```
hackrf_info version: 2026.01.3
libhackrf version: 2026.01.3 (0.9)
Found HackRF
Index: 0
Serial number: 0000000000000000xxxxxxxxxxxxxxxx
Board ID Number: 2 (HackRF One)
Firmware Version: 2026.01.3 (API:1.09)
Part ID Number: 0xa000cb3c 0x00xxxxxx
Hardware Revision: r9
Hardware appears to have been manufactured by Great Scott Gadgets.
Hardware supported by installed firmware:
    HackRF One
```

Check that the firmware version matches the `libhackrf` version, and that the hardware revision is listed as supported.

**Updating the firmware** of a working HackRF, from the release tarball above:

```sh
$ hackrf_spiflash -w $SDR/hackrf-2026.01.3/firmware-bin/hackrf_one_usb.bin
```

Then press **RESET** on the board (or unplug it) and run `hackrf_info` again. Use `hackrf_one_usb.bin` for a HackRF One; other boards (HackRF Pro, Jawbreaker, rad1o) have their own files.

**DFU recovery**, when a bad flash leaves the board without working firmware:

1. Hold the **DFU** button and press **RESET** (or plug it in while holding DFU), then release DFU. `lsusb` shows `1fc9:000c`.
2. Load the firmware into RAM and run it, then write it to flash from there:

```sh
$ sudo apt install dfu-util
$ dfu-util --device 1fc9:000c --alt 0 --download $SDR/hackrf-2026.01.3/firmware-bin/hackrf_one_usb.dfu
$ hackrf_spiflash -w $SDR/hackrf-2026.01.3/firmware-bin/hackrf_one_usb.bin
```

The `.dfu` file runs from RAM only; the second command makes the update permanent.

**SoapySDR** sees the HackRF through its module:

```sh
$ SoapySDRUtil --info                    # lists the installed modules; look for hackrf
$ SoapySDRUtil --find="driver=hackrf"
$ SoapySDRUtil --probe="driver=hackrf"   # sample rates, gains and their ranges, antennas
```

**Expected** for `--find`:

```
Found device 0
  device = HackRF One
  driver = hackrf
  label = HackRF One #0 xxxxxxxx
  part_id = a000cb3c00xxxxxx
  serial = 0000000000000000xxxxxxxxxxxxxxxx
  version = 2026.01.3
```

**Device strings** select a device in each framework:

| Framework | String | With several devices |
|---|---|---|
| SoapySDR | `driver=hackrf` | `driver=hackrf,serial=<serial>` |
| gr-osmosdr (GQRX, osmocom blocks) | `hackrf=0` | `hackrf=<serial>` |
| hackrf tools | (first device) | `-d <serial>` |

**A GUI receiver** is the quickest way to check that everything works. In **GQRX**, choose the HackRF device (string `hackrf=0`), an input rate of 8 Msps, and the WFM (stereo) mode, then tune to `$FM_HZ`. **SDR++** (binaries on its GitHub release page) has its own HackRF source module. Only one program can open the HackRF at a time: close GQRX before running `hackrf_transfer` or a flowgraph.

## 7. First reception with the command-line tools

**Recording to a file.** Tune 1 MHz below the FM station, so that it lands at +1 MHz and not on the spike at the center (section 8), and record 10 seconds at 8 Msps:

```sh
$ CF=$((FM_HZ - 1000000))
$ hackrf_transfer -r $REC/fm_${CF}_8M.cs8 -f $CF -s 8000000 -n 80000000 -a 0 -l 16 -g 20
```

| Option | Meaning |
|---|---|
| `-r FILE` | Receive to a file (`-` for standard output) |
| `-f HZ` | Center frequency |
| `-s HZ` | Sample rate |
| `-n N` | Number of samples, then stop (10 s at 8 Msps here); without it, Ctrl-C stops |
| `-a 0`, `-l 16`, `-g 20` | RF amplifier off, LNA 16 dB, VGA 20 dB (section 8) |
| `-b HZ` | Baseband filter bandwidth, when the default isn't wanted |
| `-p 1` | Antenna port power on (active antennas only) |

**Expected** (the line format differs between releases):

```
call hackrf_set_sample_rate(8000000 Hz/8.000 MHz)
call hackrf_set_hw_sync_mode(0)
call hackrf_set_freq(97000000 Hz/97.000 MHz)
Stop with Ctrl-C
15.3 MiB / 1.000 sec = 15.3 MiB/second, average power -21.4 dBfs
15.3 MiB / 1.000 sec = 15.3 MiB/second, average power -21.5 dBfs
...
Exiting...
Total time: 10.00012 s
```

Two numbers to watch. The **rate** must match `2 × fs` bytes per second (8 Msps × 2 bytes = 16 MB/s = 15.3 MiB/s); less means samples are being lost (section 17). The **average power** should stay well below 0 dBfs; a value near 0 means clipping (section 8).

**The file** has no header: it is a raw sequence of bytes `I0 Q0 I1 Q1 ...`, each a signed 8-bit integer (-128 to 127). That format is called **cs8** (complex signed 8-bit), `ci8` in SigMF. The center frequency and sample rate exist only in the file name, which is why this note puts them there, or in a SigMF metadata file (section 11). Ten seconds at 8 Msps is 160 MB; at 20 Msps, 400 MB.

**Viewing it** in inspectrum, which reads the format from the extension (`.cs8`, `.cu8`, `.cs16`, `.cf32`):

```sh
$ inspectrum $REC/fm_${CF}_8M.cs8
```

Set the sample rate to 8000000 in the side panel. Time runs down, frequency across; the FM station is a wide band 1 MHz right of the center. inspectrum is most useful for bursts (section 12): its cursors measure durations and symbol rates.

**Sweeping a wide range.** `hackrf_sweep` retunes quickly across a range and outputs power per frequency bin, faster than any GUI receiver:

```sh
$ hackrf_sweep -f 80:120 -w 100000 -a 0 -l 16 -g 20 -1 -r $REC/sweep_80-120.csv
```

| Option | Meaning |
|---|---|
| `-f MIN:MAX` | Range in MHz |
| `-w HZ` | Bin width |
| `-1` | One sweep, then exit (without it, sweeps repeat until Ctrl-C) |
| `-r FILE` | Output file (standard output by default) |

Each output line covers a slice of a few MHz: date, time, low and high frequencies in Hz, bin width, number of samples, then one power value per bin, in dB. **Expected** (truncated):

```
2026-11-12, 21:04:31.512048, 80000000, 85000000, 100000.00, 20, -71.32, -70.85, -69.97, -72.10, ...
```

A short script turns sweeps into a max-hold plot, which shows every transmitter in the range:

```python
#!/usr/bin/env python3
# sweep-plot.py FILE: max-hold plot of hackrf_sweep CSV output
import sys
import numpy as np
import matplotlib.pyplot as plt

peak = {}
for line in open(sys.argv[1]):
    f = line.split(",")
    lo, step = float(f[2]), float(f[4])
    for i, db in enumerate(f[6:]):
        hz = lo + (i + 0.5) * step
        peak[hz] = max(peak.get(hz, -200.0), float(db))
hz = np.array(sorted(peak))
plt.plot(hz / 1e6, [peak[h] for h in hz])
plt.xlabel("MHz"); plt.ylabel("dB (uncalibrated)"); plt.grid(True)
plt.show()
```

```sh
$ python3 $SDR/sweep-plot.py $REC/sweep_80-120.csv
```

The power values are relative (section 3): they compare frequencies within one sweep, not with other devices.

## 8. Gain, overload and calibration

**Gain staging.** Too little gain buries weak signals under the receiver's own noise; too much clips the ADC or overloads the front end, and creates signals that don't exist. **Rule of thumb** for the HackRF:

1. Start with the RF amplifier off, LNA 16 dB, VGA 20 dB.
2. Raise the LNA in 8 dB steps while watching the noise floor on a spectrum display. Stop when it rises by a few dB: from then on, the noise you see comes from the antenna, and more gain only reduces dynamic range.
3. Use the VGA to set the digital level: peaks well below full scale, average power in the -30 to -15 dBfs range.
4. Turn the RF amplifier on only for weak signals with no strong transmitter nearby, typically above 1 GHz.

**Clipping** shows in the time domain: in a *QT GUI Time Sink*, I or Q flat at ±1.0 (±127 in the raw file). In the spectrum it shows as a raised floor and spurs everywhere.

**Front-end overload** is subtler: a strong signal outside the band you look at (FM broadcast, a nearby cell tower, a pager transmitter) saturates the amplifiers or the mixer before any filter can remove it. The result is **intermodulation**: signals at combinations of the strong ones (`2·f1 - f2`, `f1 + f2`). **The 10 dB test** tells real signals from products: lower the gain by 10 dB (or insert a 10 dB attenuator). A real signal drops by 10 dB; a third-order product drops by about 30 dB and often disappears.

| Fix | When |
|---|---|
| Lower the front-end gain first (amplifier, then LNA) | Always the first step |
| A band-stop filter for the offending band (FM broadcast is the usual one) | A known strong band outside the one you want |
| A band-pass filter for the band you want | A fixed application (ADS-B at 1090 MHz, GNSS) |
| Move the antenna away from the transmitter, or indoors | A tower or a pager transmitter next door |
| A receiver with more bits and better front-end filtering | Constant strong-signal environments (section 4) |

**The DC spike.** In a zero-IF stage, DC offsets and LO leakage appear as a spike at exactly the center frequency. Two fixes:

- **Offset tuning**: tune the hardware a few hundred kHz to 1 MHz away from the signal, and shift it to 0 Hz in software (section 10). The spike stays where nothing is.
- A **DC Blocker** block (a narrow high-pass filter). It also removes the signal's own content near 0 Hz, so it's fine for a wide signal but not for one centered on the spike.

**IQ imbalance and images.** If the I and Q paths differ slightly in gain or phase, every signal gets a weak **mirror image** at the opposite offset: a signal at `fc + 1 MHz` creates a ghost at `fc - 1 MHz`, typically 25 to 40 dB lower. To tell an image from a real signal, change `fc` by 100 kHz: a real signal stays at the same absolute frequency, and an image moves by 200 kHz. Offset tuning puts the image where it does no harm. GQRX has an IQ balance setting; AD936x-based devices (Pluto, USRP B2xx, bladeRF 2.0) correct it on chip.

**Spurs** from the device's own clocks, mixers and USB activity show as narrow fixed lines. A spur stays at the same offset from the center when you retune, or jumps around; a real signal stays put in absolute frequency. Note the spurs of your device at the frequencies you use.

**Frequency error.** The crystal's error is a fraction of its frequency, given in **parts per million** (ppm), and every frequency derived from it carries the same fraction:

```
error (Hz) = ppm × f / 1e6        20 ppm at 100 MHz: 2 kHz    at 1 GHz: 20 kHz    at 2.4 GHz: 48 kHz
```

That's harmless for FM broadcast (200 kHz channels), annoying for narrow FM (12.5 kHz channels), and blocking for LTE cell search and GNSS acquisition, which only search a limited frequency range. The sample rate carries the same error, which matters for long recordings and timing measurements.

**Measuring the error:** receive a signal whose frequency is known to be accurate, and compare. Measure after 10 to 15 minutes of warm-up; the crystal drifts by a few ppm as the board heats.

| Reference | Accuracy | How |
|---|---|---|
| An LTE or NR cell | Base stations hold ±0.05 ppm (3GPP TS 36.104, TS 38.104) | A cell scanner reports the offset (section 18) |
| A GSM cell, where they still exist | Same requirement | `kal` (kalibrate) for the device |
| A GPS-disciplined signal generator or reference | Better than 0.01 ppm | Measure the peak of its tone on a fine FFT |
| FM broadcast carriers | Allowed ±2 kHz: too loose | Not a reference |

If the measured frequency of a known signal is `f_meas` and its true frequency `f_true`, the error is `ppm = (f_meas - f_true) / f_true × 1e6`. A fast crystal makes signals appear **low**. To correct, tune to `f × (1 + ppm/1e6)` with the ppm value as measured (negative when signals appear low), or use the framework's correction parameter (gr-osmosdr: *Freq. Corr. (ppm)*). Recalibrate when the temperature changes a lot.

**An external reference** removes the problem instead: a 10 MHz signal from a GPS-disciplined oscillator (GPSDO) on CLKIN brings the HackRF to the reference's accuracy, typically parts per billion. Check that it is in use:

```sh
$ hackrf_clock -i
```

The output reports whether a clock signal is detected on CLKIN.

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
