---
title: "Discovery Project: FPGA FFT Spectrum Analyzer"
date: 2026-03-26
draft: false
---

This page documents my **ECE 1100 Discovery Project** — an FPGA-based FFT spectrum analyzer that captures audio, transforms it to the frequency domain in hardware, and visualizes the results in real time through a browser-based interface.

## The Idea

I wanted a project that would push me to work across the full stack of digital hardware engineering — not just writing HDL, but also thinking about system architecture, signal integrity, and verification. The idea I pitched was an **FPGA FFT spectrum analyzer**: a system that takes in audio, computes a Fast Fourier Transform on the FPGA fabric, and streams the resulting frequency-domain data to a web-based visualizer over Ethernet.

I chose this because it sits right at the intersection of signal processing and digital design, which are central to the kind of silicon engineering work I want to do after graduation. It also forced me to learn skills across several domains — HDL design, DSP theory, protocol interfacing, and front-end visualization — rather than staying in one comfort zone.

At a high level, the project pipeline is:

1. **Audio Input** — Acquire 16-bit audio samples at ~48.8 kHz through a Pmod I2S2 digital microphone interface on the Nexys A7 FPGA board.
2. **Signal Conditioning** — Apply a 4096-point Hann window (ROM lookup) and an IIR biquad filter to reduce spectral leakage and shape the input.
3. **FFT Computation** — Compute a 4096-point radix-2 FFT using the Xilinx FFT IP core, then extract magnitudes with an alpha-max-beta-min approximation.
4. **Data Streaming** — Buffer results in an async FIFO with gray-code CDC, then transmit 8 raw Ethernet packets per frame (broadcast, EtherType 0x88B5) via an RMII TX-only MAC.
5. **Web Visualization** — A FastAPI backend receives raw Ethernet frames and broadcasts them over WebSocket to a React frontend that renders a live spectrum graph.

![FPGA device floorplan showing placed logic on the Xilinx XC7A100T die](/images/vivado_device.png)

## The Process

My progress through the semester followed an incremental, bottom-up strategy. Here is what each two-week period looked like:

**Feb 17 – Mar 2: Research and Project Setup.**
I spent this period researching FFT architectures, comparing radix-2 vs. radix-4 tradeoffs, and deciding on a 4096-point FFT that would give ~12 Hz frequency resolution while fitting comfortably on the Nexys A7 (XC7A100T). I set up the Vivado 2025.2 project, created initial module skeletons for the top-level (`fft_analyzer_top.sv`) and the I2S receiver (`i2s_receiver.sv`), and wrote the pin constraint file (`nexys4ddr.xdc`) to map the Pmod I2S2 signals and Ethernet PHY pins.

**Mar 3 – Mar 16: Core Pipeline Development.**
This was the most productive period. I wrote and individually tested the major RTL modules: the Hann window ROM (`hann_window.sv` with a 4K coefficient hex file), the IIR biquad filter (`iir_filter.sv`), the FFT controller wrapping the Xilinx xfft IP (`fft_controller.sv`), and the magnitude calculator (`magnitude_calc.sv`). I also built testbenches for each module (`tb_i2s_receiver.sv`, `tb_iir_filter.sv`, `tb_fft_analyzer_top.sv`) and simulated them with Icarus Verilog to verify correct behavior with deterministic test inputs (known sine waves and DC offsets). By March 12, I had my first successful Vivado implementation run with all timing constraints met.

**Mar 17 – Mar 30: Ethernet and System Integration.**
I designed and tested the Ethernet MAC (`ethernet_mac.sv`) with CRC-32 generation (`crc32.sv`), the frame buffer async FIFO (`frame_buffer.sv`) with CDC utilities (`cdc_utils.sv`), and the system control FSM (`system_control.sv`) that manages capture and packet transmission. The Ethernet MAC testbench (`tb_ethernet_mac.sv`) was critical — it caught several packet framing bugs before I attempted full-system integration. I also built the internal debug tone generator (`debug_tone_generator.sv`) so I could verify the entire pipeline end-to-end without needing real audio input.

**Mar 31 – Apr 13: Software Stack and End-to-End Demo.**
With the hardware pipeline working, I built the software side: a FastAPI backend (`server/main.py`) that opens a raw Ethernet socket and broadcasts received FFT frames over WebSocket, and a React + TypeScript frontend (`client/`) with a canvas-based spectrum visualizer. I also added a mock data mode so the web pipeline can be tested without FPGA hardware. By early April, I had a working end-to-end demo — the FPGA generates FFT data (from the internal debug tone), sends it over Ethernet, and the browser displays a live spectrum at 60 FPS. The File Analyzer tab (offline audio FFT analysis in the browser) was a bonus feature I added during this period.

![The web-based spectrum visualizer showing a live FFT analysis of an audio file, with frequency on the x-axis and magnitude in dB on the y-axis](/images/fft_analyzer_showcase.png)

## Challenges and Roadblocks

This project had its share of setbacks, and working through them ended up being some of the most valuable learning:

- **I2S Bit-Alignment Issues:** Early on, the audio samples coming out of the I2S receiver were shifted by several bits, producing garbage data downstream. I spent a significant amount of time with simulation waveforms before realizing my clock-edge and word-select logic had an off-by-one alignment error. Fixing this taught me how critical it is to simulate protocol-level interfaces thoroughly before trusting them in the larger system.

- **Timing Closure on the IIR Filter Path:** When I first synthesized the full pipeline, the tightest path had only 0.49 ns of slack — the critical path ran through the IIR filter's DSP48E1 multiplier, through 12 levels of logic (6 carry chains, LUTs), to a register in the IIR output. I learned to read Vivado timing reports in detail to understand where the bottleneck was, and I resolved it by restructuring the filter's accumulation logic to add pipeline stages. Getting all paths to meet timing at 100 MHz without any failing endpoints was a major milestone.

- **Spectral Leakage Without Windowing:** My initial FFT output looked noisy and smeared even with clean test tones. After researching, I realized I needed to apply a window function before the FFT to reduce spectral leakage. I wrote a Python script (`generate_hann_window.py`) to produce Q15 Hann coefficients, stored them in a `.hex` ROM file, and built the `hann_window.sv` module. Seeing the spectrum clean up dramatically was one of the most satisfying moments of the project.

- **Cross-Clock-Domain Integration:** The FPGA design uses two clock domains — 100 MHz for the DSP pipeline and 50 MHz for the Ethernet MAC (RMII). Getting data across that boundary reliably required building a gray-code async FIFO and proper CDC synchronizers (`cdc_utils.sv`). I initially hit intermittent data corruption that only appeared under certain timing conditions, which taught me why CDC is one of the hardest problems in digital design.

## ECE Skills Gained

This project gave me hands-on experience with several core ECE skills:

- **HDL Design (SystemVerilog):** I wrote synthesizable SystemVerilog for 13 RTL modules — from the I2S receiver to the FFT controller to the Ethernet MAC with CRC-32. This gave me practical experience with RTL coding, state machines, fixed-point arithmetic, and IP core integration that goes well beyond what I've done in class so far.

- **FPGA Synthesis and Implementation Flow:** I learned to use the Vivado toolchain end-to-end: synthesis, implementation, bitstream generation, and reading timing/utilization reports to assess design health. My final implementation uses 9.79% of LUTs (6,209 / 63,400), 4.78% of registers (6,060 / 126,800), 21 DSP48E1 slices, and 9.5 Block RAM tiles on the XC7A100T — all with 0 timing violations and a worst negative slack of +0.49 ns at 100 MHz.

- **Digital Signal Processing:** I implemented the radix-2 FFT algorithm (via the Xilinx FFT IP), Hann windowing with a 4096-point coefficient ROM, IIR biquad filtering, and alpha-max-beta-min magnitude approximation — all in fixed-point hardware. This required understanding DSP concepts like spectral leakage, frequency resolution (~12 Hz/bin), and the relationship between sample rate (48.8 kHz) and FFT size.

- **Protocol Interfacing (I2S and Ethernet):** I designed and debugged both an I2S receiver for the Pmod I2S2 audio ADC and a TX-only RMII Ethernet MAC with CRC-32 generation. This gave me experience with serial communication protocols, clock domain crossings, and bit-level debugging using simulation waveforms.

- **Verification-Oriented Thinking:** I built testbenches for each major module and used Icarus Verilog for simulation. Throughout the project, I used known-good test inputs (internal debug tone at ~1.5 kHz), checked outputs against expected values, and relied on implementation reports to validate design correctness. This approach — testing methodically rather than guessing — is directly aligned with the design verification work I want to pursue professionally.

![Vivado timing summary showing all constraints met with 0 failing endpoints](/images/vivado_timing.png)

## Results and Current Status

The project demonstrates full end-to-end system behavior: audio flows through the I2S input, conditioning, FFT, and magnitude stages on the FPGA, then streams over Ethernet to a browser-based visualizer.

**FPGA Implementation Summary (Vivado 2025.2, XC7A100T-1, post-route):**

| Resource | Used | Available | Utilization |
|---|---|---|---|
| Slice LUTs | 6,209 | 63,400 | 9.79% |
| Slice Registers | 6,060 | 126,800 | 4.78% |
| Block RAM (RAMB18) | 19 (9.5 tiles) | 270 (135 tiles) | 7.04% |
| DSP48E1 | 21 | 240 | 8.75% |
| Bonded IOB | 28 | 210 | 13.33% |

All timing constraints are met with 0 failing endpoints. The worst negative slack is +0.490 ns on the 100 MHz domain (critical path through the IIR filter's DSP multiplication and accumulation chain). The design runs two clock domains: 100 MHz for the DSP pipeline and 50 MHz for the Ethernet RMII interface, both generated by a PLL from the 100 MHz board oscillator.

The web visualizer runs at 60 FPS, providing a smooth, responsive display of the spectral data. The project also includes a File Analyzer mode that performs offline FFT analysis of uploaded audio files directly in the browser.

Current priorities include expanding validation coverage with additional test signals, investigating live microphone input on actual hardware, and exploring higher-point FFT sizes for finer frequency resolution.

## Reflections and Next Steps

This project has been the most technically demanding thing I've worked on in my time at Georgia Tech so far, and it's also been the most rewarding. It confirmed that hardware engineering and digital design are where I want to focus my career — specifically in **design verification**, where the kind of methodical, evidence-based debugging I practiced throughout this project is a core part of the job.

One thing that surprised me was how much of the work is *not* writing HDL. A huge portion of my time went into reading timing reports, debugging protocol interfaces in simulation, and designing test strategies. That experience shifted my perspective: I used to think of hardware engineering as mostly about "designing the circuit," but now I understand that verification, integration, and toolchain fluency are equally important.

If I could go back, I would have started with even simpler integration tests earlier in the semester rather than perfecting individual modules in isolation first. Some of my hardest bugs only appeared when modules were connected, and catching those earlier would have saved time.

I plan to continue developing this project beyond ECE 1100. My next goals are to add support for live microphone input on physical hardware, implement a higher-point FFT for better frequency resolution, and explore porting the design to a different FPGA platform. This project has also strengthened my interest in joining hardware-focused teams and internships — the experience of building and verifying a real digital system from scratch is something I want to keep doing.

