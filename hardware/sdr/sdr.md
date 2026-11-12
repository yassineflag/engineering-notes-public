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

*Planned: What is hardware and what is software in an SDR, the receive and transmit chains, zero-IF, superheterodyne and direct sampling, what the host actually receives, why the samples are complex.*

## 2. The example setup: hardware, software, signals and rules

*Planned: The HackRF One, the host, the comparison devices, antennas and lab accessories, the example signals, directories and variables, and the legal rules followed in this note.*

## 3. IQ samples: the minimum theory

*Planned: Complex baseband and negative frequencies, sample rate and bandwidth, aliasing, the FFT and its bins, dB, dBFS and dBm, ADC bits and dynamic range, decimation and processing gain.*

## 4. SDR hardware compared: RTL-SDR, HackRF, Pluto, USRP and others

*Planned: The families of SDR hardware compared by frequency range, bandwidth, resolution, duplex, interface, driver and price class; how to choose.*

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
