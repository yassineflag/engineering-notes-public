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

**Verification status:** nothing in this note was run. No device was connected, nothing was recorded or transmitted, and no flowgraph, script or program was executed. Commands, options and APIs follow the documentation and source of the tools: the HackRF documentation and the usage of `hackrf_transfer`, `hackrf_sweep` and `hackrf_spiflash`; GNU Radio 3.10's block documentation and the Python that GRC generates; SoapySDR, libiio and pyadi-iio, librtlsdr, rtl_433 and UHD. Versions were checked online in November 2026: HackRF release v2026.01.3 (also packaged by Debian), GNU Radio 3.10.12.0 in Debian, GNU Radio 4.0 at its third release candidate (October 2026). For the Pluto, v0.39 is the latest firmware release found. Hardware figures (TX power, frequency accuracy, practical USB rates, prices) are **typical** values from vendor documentation and common experience, not measurements. Scripts and flowgraph descriptions were written for this note and are untested; read them before using them.

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

GNU Radio is a framework of signal processing **blocks** written in C++ with Python bindings, a runtime that moves samples between them, and a graphical editor, **GNU Radio Companion** (GRC). A program is a **flowgraph**: blocks connected output to input, from **sources** (an SDR, a file, a signal generator) to **sinks** (audio, a file, a display, a network socket).

```
 Soapy HackRF Source ──> Frequency Xlating FIR Filter ──> WBFM Receive ──> Rational Resampler ──> Audio Sink
   complex, 8 Msps          complex, 500 ksps                float, 50 ksps     float, 48 ksps
         │
         └──> QT GUI Frequency Sink
```

**Streams and items.** Each connection carries a stream of **items** of one type. GRC colors ports by type, and connecting two different types is an error:

| Type | C++ type | Size | GRC port color | Typical use |
|---|---|---|---|---|
| complex | `gr_complex` (two float32) | 8 bytes | Blue | IQ samples |
| float | `float` | 4 bytes | Orange | Audio, demodulated signals, magnitudes |
| int, short | `int32_t`, `int16_t` | 4, 2 bytes | Green, yellow | Raw device samples, counters |
| byte | `uint8_t` / `int8_t` | 1 byte | Purple | Bits (one bit per byte), raw cs8/cu8 samples, packet bytes |
| message | PMT | | Grey, dashed connection | Asynchronous messages (below) |

A port can also carry **vectors** of N items (an FFT frame of 1024 complex values is one item of a vector stream).

**The sample rate isn't in the stream.** A block only sees numbers; the rate is a parameter you give to the blocks that need it (filters, demodulators, displays). If two blocks disagree, nothing fails: the audio plays at the wrong pitch or a filter cuts the wrong band. Keep the rates in GRC variables (`samp_rate`, `quad_rate`) and derive every block's parameters from them.

**What sets the pace.** A hardware source produces samples at its rate and an audio sink consumes them at its rate; the flowgraph runs as fast as they allow. A flowgraph with only a file source and a GUI sink has no clock and runs as fast as the CPU can: the **Throttle** block limits it to a given rate. Use a Throttle only in flowgraphs with no hardware or audio clock (section 11); two clocks in one flowgraph drift apart.

**Kinds of blocks:**

| Kind | Items in : out | Examples |
|---|---|---|
| Sync | 1 : 1 | Multiply, Complex to Mag, Quadrature Demod |
| Decimator | N : 1 | FIR filters with decimation, Keep 1 in N |
| Interpolator | 1 : N | Repeat, interpolating FIR filters |
| General | Any ratio, decided at run time | Rational Resampler, Symbol Sync, packet decoders |
| Hierarchical | A group of blocks presented as one | WBFM Receive (a demodulator, a de-emphasis filter, a decimating low-pass filter) |

**The scheduler.** The default scheduler runs **each block in its own thread**. Between two blocks is a circular buffer. A block's `work()` function is called with as many input items as are available and as much output space as is free, and returns how many it produced. When a buffer fills, the block upstream waits. That back-pressure is fine between software blocks; a hardware source can't wait, so when the flowgraph is too slow, its driver drops samples and reports an **overflow** (section 17).

**Tags** attach metadata to one item of a stream: a key and a value at an absolute item index, traveling with the samples through the blocks. Examples: UHD sources tag the first sample with `rx_time` and `rx_freq`; packet blocks mark packet starts with a length tag (`packet_len`); a burst transmitter uses `tx_sob` and `tx_eob` (start and end of burst). The *Tag Debug* block prints the tags it sees.

**Messages** are asynchronous: a block posts a message on a message port, not tied to any sample. Examples: a GUI sends a `freq` command to the Soapy source to retune it; a decoder outputs each decoded packet as a **PDU** (protocol data unit: a dictionary of metadata plus a vector of bytes or samples); *Message Debug* prints them. Values in tags and messages are **PMTs** (polymorphic types: numbers, strings, dictionaries, vectors).

**GNU Radio Companion.** GRC saves a flowgraph as a `.grc` file (YAML) and **generates** a Python program from it, then runs that program. The *Options* block chooses the output: a Qt GUI application, a no-GUI program, or a hierarchical block for reuse in other flowgraphs. The generated Python can be read, run without GRC, and copied as the starting point of a hand-written program (section 10). Use GRC variables for values and *QT GUI Range* blocks for sliders: changing them at run time calls the matching setters in the running blocks.

```sh
$ gnuradio-companion $SDR/fm_rx.grc
```

**GNU Radio 3.10 and 4.0.** This note uses **3.10**, the stable line packaged by distributions (3.10.12.0 in Debian 13). **GNU Radio 4.0** is a rewrite (a C++23 block API, a new scheduler, compile-time graph checks), in release candidates since March 2026 and not yet tagged final in November 2026. Flowgraphs and out-of-tree modules for 3.10 don't carry over to 4.0 as they are; check the project's news before starting a long-lived project on either.

## 10. A first flowgraph: an FM broadcast receiver

FM broadcast is a good first signal: strong, continuous, and the result is audible. A station occupies about 200 kHz around its frequency. The receiver must take the 8 Msps stream from the HackRF, select that 200 kHz, demodulate it, and deliver audio at the sound card's 48 kHz.

**The rate plan** comes first. Every rate must divide cleanly into the next one:

```
HackRF, fc = station - 1 MHz        8 Msps complex      station at +1 MHz (offset tuning, section 8)
  │ Frequency Xlating FIR Filter: shift by -1 MHz, low-pass 100 kHz, decimate by 16
  ▼                                  500 ksps complex
  │ WBFM Receive: FM demodulation, de-emphasis, decimate by 10
  ▼                                  50 ksps float (mono audio)
  │ Rational Resampler: × 24 / 25
  ▼                                  48 ksps float  ──> Audio Sink
```

**The blocks in GRC:**

| Block | Parameter | Value |
|---|---|---|
| Options | Generate Options | QT GUI |
| Variable `samp_rate` | | `8e6` |
| Variable `offset` | | `1e6` |
| QT GUI Range `station` | Default, start, stop, step | `98e6`, `87.5e6`, `108e6`, `100e3` |
| QT GUI Range `vga` | Default, start, stop, step | `20`, `0`, `62`, `2` |
| Soapy HackRF Source | Sample rate | `samp_rate` |
| | Center frequency | `station - offset` |
| | Bandwidth | `0` (the driver picks the filter) |
| | Amplifier, IF (LNA) gain, VGA gain | off, `16`, `vga` |
| Frequency Xlating FIR Filter | Decimation | `16` |
| | Taps | `firdes.low_pass(1, samp_rate, 100e3, 50e3)` |
| | Center frequency | `offset` |
| | Sample rate | `samp_rate` |
| WBFM Receive | Quadrature rate | `samp_rate / 16` |
| | Audio decimation | `10` |
| Rational Resampler (float) | Interpolation, decimation | `24`, `25` |
| Multiply Const (float) | Constant | `0.5` (volume) |
| Audio Sink | Sample rate | `48000` |
| QT GUI Frequency Sink | Center frequency, bandwidth | `station - offset`, `samp_rate`; input from the source |
| QT GUI Waterfall Sink | Same | Same |

Parameter labels vary slightly between 3.10 releases. Where the WBFM Receive block offers a de-emphasis time constant, use 50 µs in Europe and 75 µs in the Americas.

**What each choice does:**

- **Offset tuning**: the station sits 1 MHz away from the DC spike and from the IQ image of the center (section 8).
- **The filter**: a 100 kHz cutoff with a 50 kHz transition keeps the station and rejects its neighbors. A narrower transition means more taps and more CPU; `len(taps)` in a Python shell shows how many (a few hundred here). The *Frequency Xlating FIR Filter* computes only the outputs it keeps after decimation, so its cost is taps × output rate.
- **Decimating early**: everything after the filter runs at 500 ksps instead of 8 Msps.
- **The resampler**: 50 kHz to 48 kHz is the ratio 24/25. Audio cards accept 44.1 or 48 kHz; feeding them another rate gives the wrong pitch or underruns.

**Console messages:**

| Message | Meaning | Fix |
|---|---|---|
| `aU` | Audio underrun: the sound card wanted samples that weren't there | Check the rate plan ends at the sink's rate; check the CPU load; check that no Throttle also paces the flowgraph |
| `O`, or an overflow message from the driver | The hardware delivered samples nobody read in time; samples were lost | Lower the sample rate, decimate earlier, close GUI sinks (section 17) |
| Occasional `aU` after hours | The SDR clock and the sound card clock differ by a few ppm | Normal for a long-running receiver; an adaptive resampler removes it |

**The same receiver as a headless Python program.** Without Qt, it runs over SSH, as a service, or on an embedded board (section 16):

```python
#!/usr/bin/env python3
# fm_rx.py STATION_HZ: headless wideband FM receiver, HackRF, GNU Radio 3.10
import argparse
import signal
import sys

from gnuradio import analog, audio, filter, gr, soapy
from gnuradio.fft import window
from gnuradio.filter import firdes


class FmRx(gr.top_block):
    def __init__(self, station, lna, vga, samp_rate=8e6, offset=1e6):
        gr.top_block.__init__(self, "fm_rx")
        self.offset = offset
        decim = 16
        quad_rate = samp_rate / decim                       # 500 ksps

        self.src = soapy.source("driver=hackrf", "fc32", 1, "", "", [""], [""])
        self.src.set_sample_rate(0, samp_rate)
        self.src.set_bandwidth(0, 0)
        self.src.set_gain(0, "AMP", False)
        self.src.set_gain(0, "LNA", lna)
        self.src.set_gain(0, "VGA", vga)
        self.set_station(station)

        taps = firdes.low_pass(1.0, samp_rate, 100e3, 50e3, window.WIN_HAMMING)
        self.xlate = filter.freq_xlating_fir_filter_ccf(decim, taps, offset, samp_rate)
        self.demod = analog.wfm_rcv(quad_rate=quad_rate, audio_decimation=10)
        self.resamp = filter.rational_resampler_fff(interpolation=24, decimation=25)
        self.sink = audio.sink(48000, "", True)

        self.connect(self.src, self.xlate, self.demod, self.resamp, self.sink)

    def set_station(self, station):
        self.src.set_frequency(0, station - self.offset)


def main():
    p = argparse.ArgumentParser(description="Wideband FM receiver for a HackRF")
    p.add_argument("station", type=float, help="station frequency in Hz")
    p.add_argument("--lna", type=float, default=16, help="LNA gain, 0 to 40 dB in 8 dB steps")
    p.add_argument("--vga", type=float, default=20, help="VGA gain, 0 to 62 dB in 2 dB steps")
    a = p.parse_args()

    tb = FmRx(a.station, a.lna, a.vga)

    def stop(sig=None, frame=None):
        tb.stop()
        tb.wait()
        sys.exit(0)

    signal.signal(signal.SIGINT, stop)
    signal.signal(signal.SIGTERM, stop)
    tb.start()
    tb.wait()


if __name__ == "__main__":
    main()
```

```sh
$ python3 $SDR/fm_rx.py $FM_HZ --lna 16 --vga 20
```

The calls are the ones GRC generates for the same blocks. `tb.start()` starts one thread per block and returns; `tb.wait()` blocks until the flowgraph ends, which for a hardware source is never, so the signal handler stops it.

## 11. Recordings: file formats, SigMF and offline processing

Recording once and processing many times is the normal way to develop a receiver: the same samples, every time, without the device.

**Sample formats:**

| Name | SigMF `core:datatype` | Bytes per complex sample | Values | Written by |
|---|---|---|---|---|
| cs8 | `ci8` | 2 | -128 to 127 | `hackrf_transfer` |
| cu8 | `cu8` | 2 | 0 to 255, zero at 127.5 | `rtl_sdr` (RTL-SDR) |
| cs16 | `ci16_le` | 4 | 16-bit integers; often 12-bit values in 16 bits | Pluto (libiio), USRP (`sc16`), bladeRF (`SC16_Q11`) |
| cf32 | `cf32_le` | 8 | float32, about ±1.0 | GNU Radio *File Sink* of type complex, GQRX raw I/Q recordings |
| IQ WAV | | 4 (2 channels × 16 bits) | | SDR++ and Windows SDR programs; I and Q as left and right |

**Converting** a HackRF recording to cf32 in NumPy:

```python
import numpy as np

raw = np.fromfile("fm.cs8", dtype=np.int8).astype(np.float32) / 128.0
iq = (raw[0::2] + 1j * raw[1::2]).astype(np.complex64)
iq.tofile("fm.cf32")

# other formats:
# cu8  (RTL-SDR): r = (np.fromfile(f, np.uint8).astype(np.float32) - 127.5) / 127.5
# cs16 (12-bit):  r = np.fromfile(f, np.int16).astype(np.float32) / 2048.0
# cf32:           iq = np.fromfile(f, np.complex64)
```

Memory grows eightfold from cs8 to complex64: 10 seconds at 8 Msps is 160 MB on disk and 640 MB as complex64. For long recordings, open the file with `np.memmap` and process it in blocks.

**A spectrum from a recording**, averaged over many FFTs:

```python
import numpy as np
import matplotlib.pyplot as plt

fs, fc, n = 8e6, 97e6, 4096
iq = np.fromfile("fm.cf32", np.complex64)
frames = iq[: (len(iq) // n) * n].reshape(-1, n)[:2000]           # 2000 FFTs of 4096 samples
spec = np.fft.fftshift(np.fft.fft(frames * np.hanning(n), axis=1), axes=1)
psd = 10 * np.log10(np.mean(np.abs(spec) ** 2, axis=0))
freq = fc + np.fft.fftshift(np.fft.fftfreq(n, 1 / fs))
plt.plot(freq / 1e6, psd); plt.xlabel("MHz"); plt.ylabel("dB (relative)"); plt.grid(True)
plt.show()
```

**Replaying a recording through a flowgraph.** Replace the SDR source with a *File Source* (type complex, repeat on) reading the cf32 file, and keep the rest of the flowgraph. Whether a *Throttle* is needed depends on the sinks:

| Sinks in the flowgraph | Throttle |
|---|---|
| An Audio Sink | No: the sound card paces the flowgraph |
| Only GUI sinks | Yes, at the recording's sample rate, right after the File Source |
| Only file sinks or decoders, batch processing | No: let it run as fast as the CPU can |

For a raw cs8 file, *File Source* (type byte) followed by *IChar To Complex* and a *Multiply Const* of `1/128` does the conversion inside the flowgraph. Most GNU Radio examples expect cf32, so converting once is simpler.

**Head** passes N items and then ends the flowgraph: `File Source → Head (N = 8e6) → ...` processes the first second of a recording and stops, which suits tests and batch jobs.

**Recording from a flowgraph:** a *File Sink* of type complex writes cf32. Record after the first decimating filter when you only need one channel: 500 ksps instead of 8 Msps is 16 times less disk.

**SigMF metadata.** A raw file loses its frequency, rate and format as soon as it's renamed. SigMF (the Signal Metadata Format) keeps them in a JSON file next to the data: `NAME.sigmf-data` (the raw samples, unchanged) and `NAME.sigmf-meta`:

```json
{
  "global": {
    "core:datatype": "ci8",
    "core:sample_rate": 8000000,
    "core:version": "1.2.0",
    "core:hw": "HackRF One r9, ANT500, amp off, LNA 16 dB, VGA 20 dB",
    "core:recorder": "hackrf_transfer 2026.01.3",
    "core:description": "FM broadcast band, station at +1 MHz"
  },
  "captures": [
    { "core:sample_start": 0, "core:frequency": 97000000, "core:datetime": "2026-11-12T20:04:31Z" }
  ],
  "annotations": [
    { "core:sample_start": 0, "core:sample_count": 80000000,
      "core:freq_lower_edge": 97900000, "core:freq_upper_edge": 98100000,
      "core:label": "FM station" }
  ]
}
```

`captures` records each retune (a new entry at the sample where the frequency changed); `annotations` marks signals in time and frequency. The `sigmf` Python package reads and validates these files, and viewers such as IQEngine display them with their annotations. Record the gains and the antenna in `core:hw`: without them, a recording's levels can't be compared with another's.

## 12. Digital signals: decoding a 433 MHz sensor

Cheap wireless devices in the 433 MHz band (weather sensors, doorbells, remote sockets) mostly use **OOK** (on-off keying: the carrier is on or off), sometimes 2-FSK (two frequencies). Their frames are short bursts, repeated a few times. This section decodes one of your own devices; the method is the same for any unknown signal.

**1. Find it.** In GQRX or SDR++, tune to 433.92 MHz with a few Msps of bandwidth and watch the waterfall: weather sensors send every 30 to 60 seconds, doorbells and remotes when pressed. Cheap transmitters can be 100 to 200 kHz off their nominal frequency, so note where the bursts really are.

**2. Record a burst.** Tune 1 MHz below and record 3 seconds while you press the button (or around the sensor's next transmission):

```sh
$ CF=$((ISM_HZ - 1000000))
$ hackrf_transfer -r $REC/ism_${CF}_8M.cs8 -f $CF -s 8000000 -n 24000000 -a 0 -l 24 -g 20
```

**3. Look at it** in inspectrum (sample rate 8000000). Zoom into a burst. Right-click it and add an **amplitude plot** derived from the selection: the pulses appear as a square wave. Enable the cursors and set them over a few pulses to read their duration and the symbol rate.

**4. Measure the pulses.** A short script shifts the burst to 0 Hz, takes the envelope, and prints the width of each pulse and gap. The offset argument is where the burst sits relative to the center (about +920 kHz for a sensor exactly on 433.92 MHz):

```python
#!/usr/bin/env python3
# ook-pulses.py FILE.cs8 FS OFFSET_HZ: pulse and gap widths of the OOK bursts in a recording
import sys
import numpy as np

path, fs, off = sys.argv[1], float(sys.argv[2]), float(sys.argv[3])
raw = np.fromfile(path, np.int8).astype(np.float32) / 128.0
iq = (raw[0::2] + 1j * raw[1::2]).astype(np.complex64)
iq *= np.exp(-2j * np.pi * off / fs * np.arange(len(iq))).astype(np.complex64)  # burst to 0 Hz
width = int(fs / 200e3)                                     # moving average: ~200 kHz low-pass
env = np.abs(np.convolve(iq, np.ones(width, np.float32) / width, mode="same"))
threshold = (env.max() + np.median(env)) / 2                 # halfway between noise and pulses
on = env > threshold
edges = np.flatnonzero(np.diff(on.astype(np.int8))) + 1
state = on[edges[0]]
for w in np.diff(edges)[:300] / fs * 1e6:
    print(f"{'pulse' if state else 'gap  '} {w:9.1f} us")
    state = not state
```

```sh
$ python3 $SDR/ook-pulses.py $REC/ism_${CF}_8M.cs8 8e6 920e3
```

**Expected** shape of the output (the widths depend on the device):

```
pulse     500.4 us
gap      1001.2 us
pulse     499.9 us
gap      2003.0 us
pulse     500.1 us
gap      1000.8 us
...
gap      9001.5 us
```

Two or three distinct widths, then a long gap between repetitions of the frame.

**5. Find the line code:**

| Code | What carries the bit | Looks like |
|---|---|---|
| PWM (pulse width) | Short pulse = one value, long pulse = the other; constant period | Two pulse widths, gaps that complement them |
| PPM (pulse position, gap width) | Constant pulse; short or long gap | One pulse width, two gap widths (the output above) |
| Manchester | The direction of the transition in the middle of each bit | Pulses and gaps of one or two half-bit widths |
| NRZ | The level during each bit period | Widths that are multiples of one bit period |

**6. Turn bits into fields.** Write the bits of several frames one above the other and compare them with what changed between transmissions (the temperature, the button pressed): the constant part is a preamble or an identifier, a changing byte or two is the value, and the last bits are often a checksum or a CRC.

**The same steps in GNU Radio**, live or on a recording:

```
Source ─> Frequency Xlating FIR Filter ─> Complex to Mag ─> Moving Average ─> QT GUI Time Sink (triggered)
          center = offset, decimation 32,                    length 10,         ─> Threshold ─> Float To Char ─> File Sink
          low-pass 100 kHz: 250 ksps                         scale 1/10            (hysteresis)    (0 and 1 bytes)
```

Set the time sink's trigger to *Normal* on a level between noise and pulses: it then shows each burst, still on screen. The *Threshold* block's two levels (low and high) give hysteresis, so noise on the edges doesn't create extra pulses.

**Reference tools:**

- **rtl_433** decodes several hundred device types and has a generic pulse analyzer. Despite the name it reads files from any SDR and drives SoapySDR devices:

```sh
$ rtl_433 -r $REC/ism_${CF}_8M.cs8 -s 8M -A               # -A: pulse analysis and a guess at the coding
$ rtl_433 -d driver=hackrf -f 433.92M -s 2M -g "LNA=24,VGA=20"   # live, all known decoders
```

  Its analyzer output proposes a flexible decoder (`-X 'n=name,m=OOK_PPM,s=...,l=...,r=...'`) with the measured widths, which decodes the device from then on.

- **Universal Radio Hacker** (URH, `pipx install urh`) records from a HackRF, detects the modulation and bit length, shows the bits of each frame side by side, and applies decodings (Manchester, inversion). Its generator tab builds frames for transmission (section 14).

**FSK and PSK receivers**, in outline. Each block corrects one impairment of a real signal:

| Impairment | FSK chain | PSK chain |
|---|---|---|
| Wrong channel, too much bandwidth | Frequency Xlating FIR Filter | Same |
| Unknown amplitude | (FM demodulation ignores it) | AGC |
| Modulation to symbols | Quadrature Demod (gain `fs / (2π · deviation)`) | (stays complex) |
| Symbol timing | Symbol Sync (Gardner or Mueller-Müller detector, samples per symbol = `fs / baud`) | Symbol Sync with a root-raised-cosine matched filter |
| Frequency and phase offset | (small offsets only shift the demodulated level) | Costas Loop (order 2 for BPSK, 4 for QPSK) |
| Multipath | | Linear Equalizer |
| Symbols to bits | Binary Slicer | Constellation Decoder, then Differential Decoder if differentially coded |
| Start of the packet | Correlate Access Code - Tag Stream (the sync word) | Same |

The *QT GUI Constellation Sink* after the Costas loop shows whether the receiver has locked: four tight clusters for QPSK, a ring or a cloud when it hasn't. The theory of these loops belongs to the [baseband](../baseband/) notes.

## 13. Writing your own blocks

**An Embedded Python Block** lives inside the `.grc` file: add the block in GRC, open its code editor, and replace the template. This one passes samples through and tags the start and end of each burst above a threshold, which makes bursts visible to every block downstream:

```python
import numpy as np
import pmt
from gnuradio import gr


class blk(gr.sync_block):
    """Burst tagger: tags the first and last sample of each burst above a threshold"""

    def __init__(self, threshold=0.1):
        gr.sync_block.__init__(self, name="Burst tagger",
                               in_sig=[np.complex64], out_sig=[np.complex64])
        self.threshold = threshold
        self.active = False

    def work(self, input_items, output_items):
        x = input_items[0]
        output_items[0][:] = x
        above = np.abs(x) > self.threshold
        before = np.concatenate(([self.active], above[:-1]))
        for i in np.flatnonzero(above != before):
            self.active = bool(above[i])
            key = "burst_start" if self.active else "burst_end"
            self.add_item_tag(0, self.nitems_written(0) + i, pmt.intern(key), pmt.PMT_T)
        return len(x)
```

The constructor arguments become the block's parameters in GRC. `work()` receives NumPy arrays and must return the number of items it produced. Vectorized NumPy code keeps up with a few Msps; a Python loop over every sample is about a thousand times slower, which is why the loop above runs only over the transitions.

**An out-of-tree (OOT) module** is the way to package blocks in C++ or Python, reuse them across flowgraphs and install them:

```sh
$ cd $SDR
$ gr_modtool newmod lab                    # creates gr-lab/
$ cd gr-lab
$ gr_modtool add -t sync -l cpp burst_tagger --argument-list "float threshold" --add-python-qa
```

| Path | What to edit |
|---|---|
| `include/gnuradio/lab/burst_tagger.h` | The public interface: `make()` and setters |
| `lib/burst_tagger_impl.h`, `lib/burst_tagger_impl.cc` | The implementation: the constructor (input and output signatures) and `work()` |
| `python/lab/qa_burst_tagger.py` | The unit test |
| `grc/lab_burst_tagger.block.yml` | The GRC description: parameters, ports, the generated Python call |
| `python/lab/bindings/` | pybind11 bindings, regenerated by `gr_modtool bind` |

The C++ `work()` of the same block:

```cpp
int burst_tagger_impl::work(int noutput_items,
                            gr_vector_const_void_star& input_items,
                            gr_vector_void_star& output_items)
{
    auto in = static_cast<const gr_complex*>(input_items[0]);
    auto out = static_cast<gr_complex*>(output_items[0]);

    for (int i = 0; i < noutput_items; i++) {
        out[i] = in[i];
        const bool above = std::norm(in[i]) > d_threshold * d_threshold;  // |x|^2 > t^2
        if (above != d_active) {
            d_active = above;
            add_item_tag(0, nitems_written(0) + i,
                         pmt::intern(above ? "burst_start" : "burst_end"), pmt::PMT_T);
        }
    }
    return noutput_items;
}
```

`d_threshold` and `d_active` are members declared in `burst_tagger_impl.h`. A sync block only has `work()`; a general block also implements `forecast()` (how many inputs it needs for a given output) and calls `consume_each()`.

The QA test runs a tiny flowgraph from a vector source to a vector sink and checks the tags:

```python
def test_tags(self):
    src = blocks.vector_source_c([0, 0, 1, 1, 0], False)
    dut = lab.burst_tagger(0.5)
    dst = blocks.vector_sink_c()
    self.tb.connect(src, dut, dst)
    self.tb.run()
    tags = [(t.offset, pmt.symbol_to_string(t.key)) for t in dst.tags()]
    self.assertEqual(tags, [(2, "burst_start"), (4, "burst_end")])
```

**Building, testing and installing:**

```sh
$ gr_modtool bind burst_tagger           # after every change to the public header
$ cmake -B build -DCMAKE_INSTALL_PREFIX=/usr/local
$ cmake --build build -j$(nproc)
$ (cd build && ctest --output-on-failure)
$ sudo cmake --install build && sudo ldconfig
```

Installed under `/usr/local`, the module needs two things to be found:

- **Python**: `python3 -c "from gnuradio import lab"` must work. If it fails with `ModuleNotFoundError`, add the directory the install step printed for `lab/__init__.py` to `PYTHONPATH`.
- **GRC**: the block files go to `/usr/local/share/gnuradio/grc/blocks`, which GRC doesn't search by default. Add it in `~/.gnuradio/config.conf`:

```ini
[grc]
local_blocks_path=/usr/local/share/gnuradio/grc/blocks
```

**VOLK** (Vector-Optimized Library of Kernels) holds SIMD versions (SSE, AVX, NEON) of common vector operations, and GNU Radio's own blocks use it. In your blocks, `volk_32fc_magnitude_squared_32f()` and similar calls replace loops over samples. VOLK picks the fastest implementation from a profile of the machine, written once by:

```sh
$ volk_profile                 # writes ~/.volk/volk_config; minutes, run once per machine
```

This matters most on Arm boards (section 16).

**Logging**: in C++, `d_logger->info("burst at {}", nitems_written(0) + i);` (GNU Radio 3.10 uses spdlog-style format strings); in Python blocks, `self.logger.info(...)` where available, or `print`. The log level is set in `~/.gnuradio/config.conf` under `[LOG]`.

## 14. Transmitting safely and legally

**The rules** of section 2 apply in full here: everything in this section goes **into a cable** with attenuators, or into a dummy load. Over the air only with an amateur licence and within its rules, or within the exact rules of a licence-exempt band, with a filter (below).

**The test bench.** The HackRF can't receive while it transmits, so a second receiver watches the output:

```
 HackRF One (TX) ──> 30 dB attenuator ──> 20 dB attenuator ──> RTL-SDR or second SDR (RX)
                     (rated 1 W or more)
```

The arithmetic: at most +15 dBm out, minus 50 dB, gives -35 dBm at the receiver, well under the HackRF's -5 dBm input limit and in the linear range of an RTL-SDR. **Rule of thumb**: keep the receiver input below -30 dBm, and start every test with the TX VGA at 0 dB and the amplifier off. Never connect two SDRs directly, and never point a transmitting antenna at a receiver at close range without attenuation.

**A carrier** for a first check. `-c` sends a constant value to the DAC, which comes out as an unmodulated carrier at the center frequency:

```sh
$ hackrf_transfer -c 64 -f $ISM_HZ -s 8000000 -x 0 -a 0
```

On the receiver, a single line at 433.92 MHz, plus your HackRF's frequency error (section 8).

**Transmitting a file.** `hackrf_transfer -t` sends a cs8 file. This script writes one second of a tone 200 kHz above the center at half amplitude. A whole number of cycles in the file keeps the phase continuous when the file repeats:

```python
#!/usr/bin/env python3
# make-tone.py OUT.cs8: 1 s of a tone at +200 kHz, half amplitude, 8 Msps
import sys
import numpy as np

fs, f, amp = 8e6, 200e3, 0.5
x = amp * np.exp(2j * np.pi * f / fs * np.arange(int(fs)))
out = np.empty(2 * len(x), np.int8)
out[0::2] = np.round(x.real * 127)
out[1::2] = np.round(x.imag * 127)
out.tofile(sys.argv[1])
```

```sh
$ python3 $SDR/make-tone.py $SDR/tone.cs8
$ hackrf_transfer -t $SDR/tone.cs8 -f $((ISM_HZ - 200000)) -s 8000000 -x 10 -a 0 -R
```

`-x` sets the TX VGA (0 to 47 dB), `-a 1` would add the RF amplifier, and `-R` repeats the file until Ctrl-C. The tone lands at 433.92 MHz while the LO, and its leakage, stays 200 kHz away: offset tuning works for transmission too.

**A transmitter in GNU Radio:**

```
Signal Source (complex, cosine, 200 kHz, amplitude 0.5) ──> Soapy HackRF Sink
                                                            sample rate samp_rate, center ISM_HZ - 200e3,
                                                            amplifier off, VGA 0 to start
```

Keep the digital amplitude below about 0.7: values beyond ±1.0 clip in the conversion to 8 bits and spread energy over neighboring frequencies. The decoder of section 12 can be tested the same way: generate the pulses of a known frame (URH's generator tab, or a *Vector Source* of 0 and 1 values repeated to the right pulse widths, multiplied by a carrier), transmit them into the cable, and check that rtl_433 or your own flowgraph decodes the bits you sent.

**Spectral purity.** The HackRF has **no output filtering** for its harmonics and mixer products: a 433 MHz carrier also comes out at 866 MHz, 1.3 GHz and beyond, and other products appear around the wanted signal. Their level varies with frequency and settings (**typical**: a few tens of dB below the carrier). In a cable, that's harmless. Over the air, it would transmit outside the band you're allowed to use: an over-the-air transmitter needs a band-pass or low-pass filter for its band, and a check of the output on a spectrum analyzer (another SDR shows its own harmonics as well, which makes it a poor judge).

**Half duplex.** The HackRF either receives or transmits. Switching means stopping one stream and starting the other, which takes milliseconds and isn't timed precisely. That rules out protocols with a tight receive-to-transmit turnaround: a cellular UE or base station, Wi-Fi acknowledgements, most two-way packet radios. For those, a full-duplex device (Pluto, bladeRF, USRP B210, LimeSDR; section 4).

## 15. Other drivers and APIs: SoapySDR, UHD, libiio and librtlsdr

**The layers**, from the application down:

```
 application        GNU Radio, GQRX, SDR++, rtl_433, srsRAN, your program
      │
 abstraction        SoapySDR  │  gr-osmosdr  │  (or the vendor API directly)
      │
 vendor library     libhackrf │ librtlsdr │ libiio + libad9361 │ UHD │ libbladeRF │ LimeSuite
      │
 transport          libusb (user space)  │  network  │  libiio's USB or network backend
      │
 kernel             USB host controller and usbfs; no SDR-specific driver for most USB SDRs
```

Most USB SDRs have **no kernel driver**: the vendor library talks to the device through libusb and the kernel's generic usbfs. That's why access is a matter of udev rules and groups (section 6), and why a kernel driver that grabs the device first gets in the way (the RTL-SDR's DVB driver, below). The Pluto is the exception: its own Linux kernel drives the radio chip, and the host talks to that kernel's IIO subsystem (below).

**libhackrf directly.** For a small program on an embedded host, the vendor library alone is enough. The callback receives each USB transfer of raw cs8 samples:

```c
/* hackrf-power.c: average power of each USB transfer for 5 s.
 * Build: gcc -O2 -o hackrf-power hackrf-power.c -lhackrf -lm */
#include <libhackrf/hackrf.h>
#include <math.h>
#include <stdio.h>
#include <unistd.h>

static int rx_cb(hackrf_transfer *t)
{
    const int8_t *b = (const int8_t *)t->buffer;
    double p = 0;

    for (int i = 0; i < t->valid_length; i++)
        p += (double)b[i] * b[i];
    p /= t->valid_length / 2;                         /* I² + Q² per complex sample */
    printf("%6.1f dBFS\n", 10 * log10(p / (128.0 * 128.0)));
    return 0;                                         /* non-zero stops the stream */
}

int main(void)
{
    hackrf_device *dev = NULL;

    if (hackrf_init() != HACKRF_SUCCESS || hackrf_open(&dev) != HACKRF_SUCCESS) {
        fprintf(stderr, "no HackRF\n");
        return 1;
    }
    hackrf_set_sample_rate(dev, 8e6);
    hackrf_set_freq(dev, 98000000);
    hackrf_set_amp_enable(dev, 0);
    hackrf_set_lna_gain(dev, 16);
    hackrf_set_vga_gain(dev, 20);

    hackrf_start_rx(dev, rx_cb, NULL);
    sleep(5);
    hackrf_stop_rx(dev);
    hackrf_close(dev);
    hackrf_exit();
    return 0;
}
```

The callback runs in libhackrf's USB thread: it must return quickly, or transfers back up and samples are lost. A real program copies the buffer into a queue and processes it in another thread. For an Arm board, cross-compile it against the target's libhackrf as in the [toolchain note](../../embedded-linux/fundamentals/cross-compilation-toolchain/cross-compilation-toolchain.md) (`pkg-config --cflags --libs libhackrf` with the target sysroot).

**SoapySDR** is one API (C++, C and Python) for many devices, with one plugin module per device type. GNU Radio's *Soapy* blocks, SDR++, rtl_433, SatDump and many others use it, so a device with a Soapy module works in all of them. In Python (`python3-soapysdr`):

```python
import numpy as np
import SoapySDR
from SoapySDR import SOAPY_SDR_RX, SOAPY_SDR_CF32

sdr = SoapySDR.Device(dict(driver="hackrf"))
sdr.setSampleRate(SOAPY_SDR_RX, 0, 8e6)
sdr.setFrequency(SOAPY_SDR_RX, 0, 98e6)
sdr.setGain(SOAPY_SDR_RX, 0, "LNA", 16)
sdr.setGain(SOAPY_SDR_RX, 0, "VGA", 20)

rx = sdr.setupStream(SOAPY_SDR_RX, SOAPY_SDR_CF32)
sdr.activateStream(rx)
buf = np.empty(65536, np.complex64)
for _ in range(100):
    sr = sdr.readStream(rx, [buf], len(buf))
    if sr.ret < 0:                    # negative: an error code, -4 is SOAPY_SDR_OVERFLOW
        print("error", sr.ret)
        continue
    print(sr.ret, "samples, mean power", np.mean(np.abs(buf[: sr.ret]) ** 2))
sdr.deactivateStream(rx)
sdr.closeStream(rx)
```

Changing the dictionary to `driver=rtlsdr` or `driver=plutosdr` (with the module installed) runs the same code on another device; only the gain names and the sample rate limits change (`SoapySDRUtil --probe` lists them). **SoapyRemote** serves a device over the network: `SoapySDRServer --bind` on the machine with the SDR, `driver=remote,remote=<host>` on the client.

**UHD** is Ettus Research's driver for the USRPs. The FPGA images must match the UHD version, and `uhd_images_downloader` fetches them:

```sh
$ sudo apt install uhd-host
$ sudo uhd_images_downloader
$ uhd_find_devices
$ uhd_usrp_probe                            # daughterboards, ranges, clock and time sources
$ /usr/lib/uhd/examples/rx_samples_to_file --freq 98e6 --rate 8e6 --duration 10 --type short $REC/fm.cs16
```

UHD has what the HackRF lacks: device timestamps on every buffer (`rx_time` tags in GNU Radio), commands timed to a sample, external 10 MHz and PPS inputs, and full duplex. Cellular stacks (srsRAN, OpenAirInterface) are developed mainly against it (section 18). The examples' directory can differ between distributions.

**libiio and the Pluto.** The Pluto's AD9363 is driven by an IIO (Industrial I/O) driver in the Pluto's own Linux kernel. Its settings are IIO attributes (frequency, gain, sample rate) and its samples flow through IIO buffers. The `iiod` daemon on the Pluto serves them over USB or the network, and **libiio** on the host is the client: the same API works locally on the device (`local:`), over USB (`usb:`) and over the network (`ip:`):

```sh
$ sudo apt install libiio-utils python3-pip
$ iio_info -s                                          # find contexts: usb:... and ip:192.168.2.1
$ iio_attr -u ip:192.168.2.1 -c ad9361-phy altvoltage0 frequency 433920000    # RX LO
$ iio_attr -u ip:192.168.2.1 -i -c ad9361-phy voltage0 gain_control_mode manual
$ iio_attr -u ip:192.168.2.1 -i -c ad9361-phy voltage0 hardwaregain 40
```

In `ad9361-phy`, `altvoltage0` is the RX LO and `altvoltage1` the TX LO; `voltage0` is the first RX (input) or TX (output) channel. The sample buffers are on the `cf-ad9361-lpc` device (RX) and `cf-ad9361-dds-core-lpc` (TX). The syntax above is that of libiio 0.2x; libiio 1.0 changes the API and some tool options, so check which version your distribution ships. In Python, **pyadi-iio** wraps the same attributes:

```python
import adi

sdr = adi.Pluto("ip:192.168.2.1")
sdr.sample_rate = int(4e6)
sdr.rx_lo = int(433.92e6)
sdr.rx_rf_bandwidth = int(2e6)
sdr.gain_control_mode_chan0 = "manual"
sdr.rx_hardwaregain_chan0 = 40
sdr.rx_buffer_size = 2 ** 16
x = sdr.rx()               # NumPy complex array; 12-bit sample values
```

In GNU Radio 3.10, the *PlutoSDR Source* and *Sink* blocks (gr-iio, part of GNU Radio) take the same URI.

**librtlsdr and the DVB driver.** The RTL2832U is a DVB-T TV receiver chip, and the kernel's `dvb_usb_rtl28xxu` driver claims it as a TV tuner, after which librtlsdr can't open it (`usb_claim_interface error -6`). Debian's `rtl-sdr` package blacklists the module; otherwise:

```sh
$ echo 'blacklist dvb_usb_rtl28xxu' | sudo tee /etc/modprobe.d/blacklist-rtlsdr.conf
$ sudo modprobe -r dvb_usb_rtl28xxu
$ rtl_test                                         # finds the stick, tests for lost samples
$ rtl_sdr -f 98e6 -s 2.4e6 -n 24000000 $REC/fm_98M_2M4.cu8
$ rtl_fm -f 98e6 -M wbfm -s 200000 -r 48000 - | aplay -r 48000 -f S16_LE
```

The RTL-SDR Blog V4 uses an R828D tuner and a built-in HF upconverter that older librtlsdr releases don't drive correctly: use a librtlsdr with V4 support (the RTL-SDR Blog fork, or an upstream release that includes it).

**Streaming over a network.** `rtl_tcp`, SoapyRemote, or GNU Radio's *ZMQ PUB Sink* / *ZMQ SUB Source* and UDP blocks move samples between machines. Count the bandwidth first: 8 Msps as cf32 is 64 MB/s, half of a gigabit link and far beyond Wi-Fi. Send integers instead of floats, or decimate before sending.

**Choosing an API for a product:**

| Need | Choice |
|---|---|
| One device type, full control, smallest footprint | Its vendor library (libhackrf, librtlsdr, libiio, UHD) |
| Several device types, or freedom to change device later | SoapySDR |
| Rapid development, a GUI, an existing block library | GNU Radio; move hot spots to C++ blocks (section 13) |
| Timestamped samples, timed transmission | UHD, or libiio and Soapy drivers that support time; not the HackRF (section 17) |
| Processing next to the radio | The Pluto's local IIO context, or an FPGA (section 16) |

## 16. SDR on embedded Linux: Raspberry Pi 4, BeagleBone Black and the Pluto

An SDR on a small board is a common product shape: a sensor receiver that publishes decoded values, a spectrum monitor, an ADS-B or AIS station. The limits are the USB link, the CPU, and storage.

**Budgets**, **typical**:

| Host | USB | CPU | What fits |
|---|---|---|---|
| Raspberry Pi 4 | Two USB 3.0 and two USB 2.0 ports, all behind one VL805 controller on PCIe | 4 × Cortex-A72, 1.5 to 1.8 GHz, NEON | HackRF capture at 8 to 10 Msps; FM demodulation with early decimation; decoders such as rtl_433, dump1090 and AIS-catcher |
| BeagleBone Black | One USB 2.0 host port (MUSB controller) | 1 × Cortex-A8, 1 GHz, NEON | An RTL-SDR at 1 to 2.4 Msps with a light decoder (rtl_433); a HackRF only at low rates with almost no processing |
| ADALM-Pluto, on the device | | 2 × Cortex-A9, 667 MHz, plus a Zynq-7010 FPGA | Light processing next to the radio; only the results cross USB |

**How to fit:**

- **Pick the lowest sample rate that works.** Bytes per second and CPU load both scale with it.
- **Decimate first and in stages.** At 8 Msps, a cheap filter with a wide transition followed by `/8`, then a sharper filter at 1 Msps, costs much less than one sharp filter at 8 Msps. Every block after the first decimation runs at the lower rate.
- **No GUI sinks**, no Python loops over samples: the headless program of section 10 or a C program (section 15).
- **Run `volk_profile` on the target** once. Without a profile, VOLK may use generic kernels instead of NEON ones.
- **Set the CPU governor to `performance`** while measuring: frequency scaling makes throughput vary and can cause overflows on load changes.
- **Don't record to the SD card at high rates.** SD cards stall for hundreds of milliseconds while they erase; use a tmpfs, a USB SSD, or decimate before writing.
- **Power.** A HackRF draws several hundred mA from USB. The Pi 4 limits the total current of its USB ports, and the BBB's host port supplies 500 mA. A powered hub fixes power but can add USB trouble (section 17); prefer a board supply with enough margin.

**Debian on the Pi**: Raspberry Pi OS (64-bit) has the same packages as section 6 (`hackrf`, `gnuradio`, `rtl-433`, `soapysdr-module-hackrf`), so a prototype is an `apt install` away.

**A headless receiver as a service**: rtl_433 decodes the 433 MHz sensors and publishes them to an MQTT broker on the network (`192.168.0.10`, a placeholder):

```ini
# /etc/systemd/system/sensors433.service
[Unit]
Description=433 MHz sensor receiver
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/rtl_433 -d driver=hackrf -f 433.92M -s 2M -F mqtt://192.168.0.10:1883
Restart=on-failure
RestartSec=5
User=sdr
Group=plugdev

[Install]
WantedBy=multi-user.target
```

```sh
# useradd --system --groups plugdev sdr
# systemctl daemon-reload && systemctl enable --now sensors433
# journalctl -u sensors433 -f
```

On a BusyBox system without systemd, the same command goes into a start script under `/etc/init.d/` as in the [rootfs note](../../embedded-linux/fundamentals/rootfs/rootfs.md#7-init-inittab-and-the-startup-scripts), with a loop that restarts it, and a udev or mdev rule for the device permissions (rootfs note, section 6).

**Build systems.** Buildroot has packages for `hackrf`, `librtlsdr` (`rtl-sdr`), `libiio`, `gnuradio` (with one option per component: analog, filter, audio, Python, ...) and `gr-osmosdr` (with HackRF and RTL-SDR options). For Yocto, the `meta-sdr` layer provides GNU Radio, UHD, HackRF and RTL-SDR recipes; check the layer index for one compatible with your Yocto release. GNU Radio with Python and its dependencies (Boost, FFTW, Volk, NumPy) adds tens of MB to a root filesystem; a C program on libhackrf or librtlsdr adds a few hundred kB. The [Buildroot](../../embedded-linux/linux-build-systems/buildroot/) and [Yocto](../../embedded-linux/linux-build-systems/yocto/) notes cover the build systems themselves.

**The Pluto is an embedded Linux system.** Inside: a Zynq-7010 (two Cortex-A9 cores and an FPGA), the AD9363 transceiver, 512 MB of DDR3L and 32 MB of QSPI flash. Its firmware, built from the `plutosdr-fw` repository, packs the pieces of the embedded Linux notes into one file, `pluto.frm`: U-Boot, an ADI Linux kernel with the IIO drivers, a device tree, the FPGA bitstream (built with Xilinx Vivado), and a Buildroot root filesystem run as an initramfs. Over USB it appears as a composite gadget:

| USB function | What it gives |
|---|---|
| Mass storage | `config.txt` (network and USB settings), `info.html`, and firmware updates: copy `pluto.frm` to it and eject |
| Network (RNDIS / ECM) | The Pluto at `192.168.2.1`: libiio (`ip:192.168.2.1`) and SSH (`root`, default password `analog`; change it on a shared network) |
| Serial (ACM) | A console, `/dev/ttyACM0` on the host |
| libiio over USB | The `usb:` context |

From an SSH session on the Pluto, the radio is a local IIO context. A program cross-compiled for its Cortex-A9 (`arm-linux-gnueabihf`, the same toolchain family as the BBB in the toolchain note) can run there, process samples locally and send only results over USB. Two things the community does with it: extend the tuning range to 70 MHz to 6 GHz and 56 MHz of bandwidth by declaring the chip as an AD9364 (`fw_setenv attr_name compatible`, `fw_setenv attr_val ad9364`, reboot; documented on the ADI wiki, outside the AD9363's specification, works on most units), and alternative firmware that moves more processing into the FPGA (Maia SDR: a waterfall computed in the FPGA, served to a browser).

**FPGA-based SDR.** When the host can't keep up, the first stages move into an FPGA: digital downconversion, filtering, decimation, triggering, sometimes whole demodulators. USRPs have RFNoC (a framework for blocks inside the USRP's FPGA), the Pluto and Zynq boards have the ADI HDL reference designs, and RFSoC boards put the ADCs, DACs and FPGA on one chip. That's HDL work, outside this note.

## 17. Throughput, timing and synchronization

**Bytes per second:**

| Stream | Rate | Over |
|---|---|---|
| HackRF, 20 Msps, cs8 | 40 MB/s | USB 2.0 high speed carries about 35 to 45 MB/s of bulk data in practice: at the limit |
| HackRF, 8 Msps, cs8 | 16 MB/s | Comfortable on a dedicated port |
| RTL-SDR, 2.4 Msps, cu8 | 4.8 MB/s | Easy |
| Pluto, 4 Msps, cs16, one channel | 16 MB/s | Around the Pluto's practical USB limit |
| USRP B210, 56 Msps, sc16, one channel | 224 MB/s | USB 3.0 |
| Any stream at 8 Msps as cf32 inside GNU Radio | 64 MB/s per connection | Memory bandwidth, per buffer |

**Where samples get lost**, and the first thing to check for each:

| Cause | Check |
|---|---|
| The USB link: a shared controller, a hub, a USB-C dock, a virtual machine's USB passthrough, a long or poor cable | `hackrf_transfer -r /dev/null -s 20000000 -n 200000000`: the rate line must match the sample rate × 2 |
| The CPU: the flowgraph is slower than the stream | `top` per thread (`H`): one block's thread at 100% is the bottleneck |
| The disk: writes stall | Write to `/dev/null`, then to a tmpfs, then to the disk |
| Power saving: CPU frequency scaling, USB autosuspend | `performance` governor; `cat /sys/bus/usb/devices/*/power/control` |

Test the layers in that order: USB alone (`/dev/null`), then the disk, then the processing. `hackrf_transfer` prints `Couldn't transfer any bytes for one second.` when the stream stops altogether, which is usually a USB or power problem.

**Buffers and latency.** Each stage holds samples: USB transfers (libhackrf uses transfers of a few hundred kB, so tens of ms each at 8 Msps), GNU Radio's buffers between blocks, the audio card's buffer. Larger buffers absorb jitter and reduce overflows; smaller ones reduce latency. A host-based SDR has a latency of milliseconds to hundreds of milliseconds from antenna to application. Protocols that must answer within microseconds (Wi-Fi ACKs, cellular HARQ timing) need the response generated in the FPGA or a dedicated chip. In GNU Radio, a block's maximum output buffer (in its *Advanced* tab) trades one against the other.

**Timestamps.** HackRF samples carry no time: the time of a sample is its index divided by the sample rate, counted from a start that the host knows only within the USB latency (milliseconds). UHD devices timestamp every buffer with the device's own clock (`rx_time`), which a **PPS** (pulse per second) input sets to absolute time; libiio and some Soapy drivers support time as well. Anything that compares the timing of signals across devices (time difference of arrival, TDD frame alignment) needs that.

**Frequency synchronization.** Devices that share a reference run at the same frequency: chain the HackRFs from CLKOUT to CLKIN, or distribute one 10 MHz signal (from a GPSDO) to all of them. **Start synchronization**: the HackRF's hardware trigger (the trigger pins on its expansion header, with `hackrf_transfer -H` to wait for the trigger) starts several devices on the same edge.

**Phase coherence** is a third, harder requirement: the HackRF's synthesizers and mixers settle at an arbitrary phase after each tune, so two HackRFs on the same reference still have an unknown, changing phase difference. Direction finding and beamforming need receivers designed for it: channels sharing one LO (the two channels of a USRP B210 or bladeRF 2.0), or multi-channel receivers built for coherence (KrakenSDR), with a calibration step.

## 18. Applications: aircraft, sensors, satellites, GNSS and cellular

| Application | Frequency | Tool | What it needs |
|---|---|---|---|
| FM and AM broadcast, airband voice | 87.5 to 108 MHz, 118 to 137 MHz (AM) | GQRX, SDR++ | Any SDR; section 10 |
| ISM sensors and remotes | 433.92, 868, 915 MHz | rtl_433, URH | Any SDR; section 12 |
| ADS-B from aircraft | 1090 MHz | dump1090-fa, readsb, then tar1090 for a map | 2 Msps; a 1090 MHz filter and a good antenna; RTL-SDR is the usual choice, HackRF works with builds that support it |
| AIS from ships | 161.975 and 162.025 MHz | AIS-catcher | Most SDRs, including the HackRF through SoapySDR |
| Weather satellites | 137 MHz (Meteor-M LRPT); 1.7 GHz (HRPT, GOES HRIT, with a dish) | SatDump | A V-dipole or QFH antenna for 137 MHz. The NOAA APT satellites were retired in 2025 |
| Spectrum surveys | 1 MHz to 6 GHz | `hackrf_sweep`, QSpectrumAnalyzer | The HackRF's sweep mode (section 7) |
| GNSS (GPS, Galileo) | 1575.42 MHz (L1/E1) | gnss-sdr | An active antenna powered by the bias tee (`-p 1`, or `bias=1` in an osmosdr device string); a stable clock helps acquisition (section 8); 2 to 4 Msps |
| LTE and NR cell information | The operator bands ([cellular note](../../wireless-protocols-and-connectivity/cellular/cellular.md#7-physical-channels-cell-search-system-information-and-random-access)) | srsRAN 4G's `cell_search`, LTE cell scanners | Receive only; the broadcast part (MIB, SIBs) is unencrypted |
| Running an LTE or 5G network | Test bands | srsRAN 4G, srsRAN Project, OpenAirInterface | Full duplex, timestamps and a good clock: USRP B210 or similar, not a HackRF |

**GNSS signals are below the noise floor**: about -130 dBm, spread over 2 MHz, where the thermal noise is about -111 dBm (section 3). The receiver finds them by correlating with the known spreading code: one millisecond of correlation narrows 2 MHz to about 1 kHz, about 33 dB of processing gain, and longer integration adds more. That's also why 8 bits are enough. Acquisition searches frequency in steps; a ±20 ppm crystal adds ±31 kHz of uncertainty at 1575 MHz, which widens the search and slows the first fix. Never transmit on GNSS frequencies, even into a cable in a building: leakage is enough to disturb receivers nearby, and it is illegal everywhere.

**Cellular, receive side.** The LTE cell search signals (PSS, SSS) and the PBCH occupy the central 1.08 MHz of the carrier, and an NR SSB occupies 240 subcarriers: 3.6 MHz at 15 kHz spacing, 7.2 MHz at 30 kHz. Both fit in the HackRF's bandwidth, so the cell identity, the MIB and the frequency error can be measured with it; the [cellular note](../../wireless-protocols-and-connectivity/cellular/cellular.md#7-physical-channels-cell-search-system-information-and-random-access) describes what the cell search finds. srsRAN 4G's tools reach devices through its RF layer (UHD, bladeRF, SoapySDR). A cell's reported frequency offset is also the most accurate calibration a HackRF owner usually has (section 8).

**Cellular, network side.** An open-source eNB or gNB with a programmable SIM is the way to study attach, registration and the core from the inside, with the test PLMN 001-01 of the cellular note. It needs a full-duplex SDR with timestamps (a USRP B210 is the reference), a host with enough CPU, and either a shielded enclosure with cables and attenuators, or a test licence for the band. Transmitting on an operator's band without one interferes with real users and emergency calls.

**Wi-Fi and Bluetooth.** A 20 MHz Wi-Fi channel needs 20 Msps of clean bandwidth, the HackRF's limit; GNU Radio's gr-ieee802-11 decodes it with a better device. Bluetooth hops over 79 MHz faster than any USB SDR can follow; dedicated hardware (Ubertooth) or a wideband SDR is needed.

## 19. Troubleshooting by symptom

**The device and USB:**

| Symptom | Likely cause | Check, then see |
|---|---|---|
| `hackrf_info`: `No HackRF boards found.` | Not enumerated: a charge-only cable, a bad port, no firmware | `lsusb`: `1d50:6089` normal, `1fc9:000c` DFU mode, nothing at all: the cable (section 6) |
| `hackrf_open() failed: Access denied (insufficient permissions)` | No udev rule, or the user isn't in `plugdev` yet | `id` shows `plugdev`; log out and in after `usermod` (section 6) |
| `hackrf_open() failed: Resource busy` | Another program has the device open | Close GQRX, SDR++, other flowgraphs; `lsof | grep -i usb` |
| Firmware and library versions differ in `hackrf_info`, or odd errors after an update | Host tools and firmware from different releases; two libhackrf copies installed | Flash the matching firmware; `which hackrf_info`, `ldd` (section 6) |
| No device after `hackrf_spiflash` | Interrupted or wrong firmware file | DFU recovery (section 6) |
| `Couldn't transfer any bytes for one second.` | USB stalled: power, cable, controller, hub | Another port directly on the machine; a powered setup (sections 16, 17) |
| The rate in `hackrf_transfer` is below `2 × fs` bytes/s | Samples lost on USB or disk | `-r /dev/null` first, then the disk (section 17) |
| `SoapySDR::Device::make() no match` | The Soapy module for the device isn't installed or doesn't load | `SoapySDRUtil --info` lists modules and load errors (section 6) |

**The signal:**

| Symptom | Likely cause | Check, then see |
|---|---|---|
| Nothing but noise | Wrong antenna for the band, gain too low, bias tee needed for an active antenna | A strong known signal first (FM broadcast); LNA up in 8 dB steps (section 8) |
| A spike at the exact center | DC offset and LO leakage (zero-IF) | Offset tuning (section 8) |
| A weaker copy of a signal on the other side of the center | IQ imbalance image | Retune by 100 kHz: an image moves twice as far (section 8) |
| Signals everywhere, FM stations at odd frequencies, a raised floor | Front-end overload or ADC clipping | The 10 dB test; lower gain; a band-stop filter (section 8) |
| Narrow lines that move with the center frequency | Spurs from the device itself | Note them; offset tuning around them (section 8) |
| Signals a few kHz to tens of kHz off their known frequency | Crystal error | Measure ppm against a reliable signal (section 8) |
| A narrow signal is noisy or distorted at low sample rates | Aliasing below 8 Msps | 8 Msps or more, decimate in software (section 5) |
| Bursts missing from a long recording | Lost samples | Rate line, overflows (section 17) |
| A recording looks like noise in a viewer or NumPy | Wrong format: cs8 read as cu8, cs16 as cf32, wrong sample rate | Format table (section 11); SigMF from then on |

**GNU Radio:**

| Symptom | Likely cause | Check, then see |
|---|---|---|
| `aU` on the console, choppy audio | The rate plan doesn't end at the audio sink's rate, the CPU can't keep up, or a Throttle also paces the graph | Recompute the rate plan; `top -H` (section 10) |
| `O` or overflow messages | The flowgraph is slower than the source | Decimate earlier, remove GUI sinks, lower the rate (sections 10, 16, 17) |
| Audio at the wrong pitch, filters cutting the wrong band | Blocks disagree about the sample rate | Every rate derived from GRC variables (section 9) |
| A file-only flowgraph uses 100% CPU and the GUI races | No clock in the flowgraph | Add a Throttle (section 11) |
| `ModuleNotFoundError: No module named 'osmosdr'` (or `soapy`, `iio`) | The package isn't installed for this Python | `apt install gr-osmosdr`; same Python as GNU Radio |
| An OOT block is missing in GRC, or `AttributeError` on import | GRC doesn't search `/usr/local`; bindings not regenerated; `ldconfig` not run | `local_blocks_path`, `gr_modtool bind`, `PYTHONPATH` (section 13) |
| The constellation is a ring or a cloud | No carrier or timing lock: wrong samples per symbol, too much frequency offset | Symbol Sync parameters, Costas loop order (section 12) |

**Transmission:**

| Symptom | Likely cause | Check, then see |
|---|---|---|
| Nothing on the receiver | TX VGA at 0 and too much attenuation, wrong frequency, the receiver is the same HackRF | Raise `-x` in steps; a second receiver (section 14) |
| A wide, splattered signal | Digital amplitude above 1.0 clips | Keep the amplitude below about 0.7 (section 14) |
| A line at the TX center frequency | LO leakage | Offset the signal from the center (section 14) |
| Signals at 2× and 3× the frequency | Harmonics: the HackRF has no output filter | A filter before any antenna (section 14) |
| The receiving SDR stops working after a test | Too much power at its input | Attenuation: below -30 dBm at the receiver, never above -5 dBm for a HackRF (sections 5, 14) |

**Other devices:**

| Symptom | Likely cause | Check, then see |
|---|---|---|
| RTL-SDR: `usb_claim_interface error -6` | The DVB-T driver holds the stick | Blacklist `dvb_usb_rtl28xxu` (section 15) |
| RTL-SDR Blog V4: wrong frequencies, no HF | librtlsdr without V4 support | A librtlsdr with V4 support (section 15) |
| Pluto not at 192.168.2.1 | The USB network interface isn't up on the host, or `config.txt` changed the address | `ip addr` for a new `usb0` or `enx...` interface; read `config.txt` on the mass storage (section 16) |
| Pluto: overflows above a few Msps | USB 2.0 throughput | 4 Msps or less, or process on the Pluto (section 16) |
| `iio_attr` options rejected | libiio 1.0 tools against 0.2x syntax, or the reverse | `iio_info -V`; the tool's `--help` (section 15) |
| USRP: `FPGA image not found` or version mismatch | Images don't match the UHD version | `uhd_images_downloader` (section 15) |
| GNSS: no fix | No antenna power, no sky view, large frequency error | Bias tee, antenna outdoors, an external reference (section 18) |

## 20. Not covered here

- **Signal processing in depth**: filter design, the FFT, windows, multirate systems: the [signal processing](../signal-processing/) folder.
- **Modulation, synchronization and coding theory**: matched filters, carrier and timing recovery, OFDM, channel coding: the [baseband](../baseband/) folder.
- **RF design**: antennas, LNAs, filters, mixers, link budgets: the [RF](../rf/) folder.
- **The protocols**: what Wi-Fi and cellular transmit and why: the [Wi-Fi](../../wireless-protocols-and-connectivity/wifi/wifi.md) and [cellular](../../wireless-protocols-and-connectivity/cellular/cellular.md) notes.
- **FPGA development for SDR**: RFNoC, the ADI HDL reference designs, RFSoC.
- **GNU Radio 4.0**: its block API and scheduler, once released.
- **The PortaPack and its firmware**, the Opera Cake, and the HackRF Pro.
- **Security research on radio devices** (replay, rolling codes, jamming detection): only on your own devices, within the law (section 2).

## References

HackRF (https://greatscottgadgets.com/hackrf/):

- Documentation: https://hackrf.readthedocs.io: hardware components, the frequency plan, gain stages, sample rates, firmware updates and DFU recovery, CLKIN/CLKOUT and hardware triggering, `hackrf_sweep`.
- Source and releases: https://github.com/greatscottgadgets/hackrf
- `hackrf_transfer -h`, `hackrf_sweep -h`: every option.

GNU Radio (https://www.gnuradio.org):

- Wiki and tutorials: https://wiki.gnuradio.org: the beginner tutorials, *Your First Flowgraph*, *Creating Python OOT with gr-modtool*, *Creating C++ OOT with gr-modtool*, *Stream Tags*, *Message Passing*, *Embedded Python Block*.
- Block documentation: https://wiki.gnuradio.org/index.php/Category:Block_Docs
- VOLK: https://www.libvolk.org
- News, including the GNU Radio 4.0 release candidates: https://www.gnuradio.org/news/

Drivers and APIs:

- SoapySDR: https://github.com/pothosware/SoapySDR/wiki
- UHD and the USRP manual: https://files.ettus.com/manual/
- libiio: https://analogdevicesinc.github.io/libiio/ and pyadi-iio: https://analogdevicesinc.github.io/pyadi-iio/
- ADALM-Pluto: https://wiki.analog.com/university/tools/pluto and the firmware: https://github.com/analogdevicesinc/plutosdr-fw
- RTL-SDR: https://osmocom.org/projects/rtl-sdr/wiki and the RTL-SDR Blog V4 guide: https://www.rtl-sdr.com/V4/
- Linux IIO: https://docs.kernel.org/driver-api/iio/

Tools:

- inspectrum: https://github.com/miek/inspectrum
- Universal Radio Hacker: https://github.com/jopohl/urh
- rtl_433: https://github.com/merbanan/rtl_433 (and its `docs/ANALYZE.md` for unknown devices)
- GQRX: https://gqrx.dk; SDR++: https://github.com/AlexandreRouma/SDRPlusPlus
- SigMF specification: https://github.com/sigmf/SigMF
- SatDump: https://github.com/SatDump/SatDump; AIS-catcher: https://github.com/jvde-github/AIS-catcher
- gnss-sdr: https://gnss-sdr.org
- srsRAN: https://www.srsran.com; OpenAirInterface: https://openairinterface.org

Regulation:

- ERC Recommendation 70-03 (short range devices in CEPT countries): https://docdb.cept.org
- FCC Part 15: 47 CFR Part 15, https://www.ecfr.gov
- Your national regulator, for licences and the exact rules of each band.

Books and courses:

- Marc Lichtman, *PySDR: A Guide to SDR and DSP using Python* (free): https://pysdr.org: IQ sampling, filters, synchronization, with the Pluto and other devices.
- Michael Ossmann, *Software Defined Radio with HackRF* (free video course): https://greatscottgadgets.com/sdr/
- Travis F. Collins, Robin Getz, Di Pu, Alexander M. Wyglinski, *Software-Defined Radio for Engineers* (Artech House, 2018; free PDF from Analog Devices), built around the Pluto.
- Richard G. Lyons, *Understanding Digital Signal Processing*: the DSP behind every block.
