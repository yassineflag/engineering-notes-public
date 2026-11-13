# Hardware

Notes on electronics and radio hardware: analog and digital circuits, signal processing, baseband, RF, and software-defined radio.

## Notes

### Analog

- [Analog](analog/)

### Digital

- [Digital](digital/)

### Signal processing

- [Signal processing](signal-processing/)

### Baseband

- [Baseband](baseband/) (modulation, IQ, coding)

### RF

- [RF](rf/)

### SDR

- [Software-Defined Radio: HackRF One and GNU Radio](sdr/sdr_v1_2026-11-12.md): what an SDR is from the antenna to IQ samples, the minimum theory (complex baseband, aliasing, FFT bins, dB, dBFS and dBm, ADC bits and dynamic range, decimation), SDR hardware compared (RTL-SDR, Airspy, SDRplay, HackRF, Pluto, LimeSDR, bladeRF, USRP), inside the HackRF One (frequency plan, gain stages, limits, firmware and DFU recovery), first recordings with `hackrf_transfer` and `hackrf_sweep`, gain staging, overload, DC spike, IQ images and ppm calibration, GNU Radio concepts and an FM receiver in GRC and in headless Python, recordings with SigMF, decoding a 433 MHz OOK sensor (inspectrum, rtl_433, URH) and FSK/PSK chains, writing Python and C++ blocks with `gr_modtool`, transmitting into cables legally and safely, SoapySDR, libhackrf, UHD, libiio and librtlsdr, SDR on a Raspberry Pi 4, a BeagleBone Black and the Pluto's own embedded Linux, throughput, timestamps and synchronization, applications from ADS-B to GNSS and cellular, and troubleshooting by symptom.
