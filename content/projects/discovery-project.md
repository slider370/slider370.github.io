---
title: "Discovery Project: FPGA FFT Spectrum Analyzer"
date: 2026-03-26
draft: false
---

This page documents my **ECE 1100 Discovery Project**, which is the same technical effort as my FPGA FFT spectrum analyzer project.

## Discovery Project Summary

My Discovery Project explores how digital hardware, signal processing, and system integration come together in a practical engineering workflow. I implemented an FPGA-based FFT analyzer that captures audio, transforms it to the frequency domain, and sends spectral data toward a browser-based visualization path. The core reason I chose this project is that it reflects the type of work I want to pursue long term: rigorous, silicon-adjacent engineering where architecture, correctness, and verification all matter.

At a high level, the project pipeline is:

1. Acquire audio samples through an I2S input path.
2. Apply signal conditioning and windowing.
3. Compute FFT and magnitude data in hardware.
4. Package and stream results over Ethernet.
5. Display the resulting spectrum in a web interface.

This flow gave me a full-stack engineering challenge while still keeping the center of gravity in hardware design and verification-oriented thinking.

![Current FFT analyzer spectrum view screenshot from my project workflow](/images/fft_analyzer_showcase.png)

## Context, Contributions, and Learning

My specific contribution has been developing and integrating the system in a way that is testable and incrementally debuggable. That includes working across module boundaries, validating signal flow with known stimuli, and checking implementation reports to confirm design health. A major practical lesson from this process is the importance of known-good test conditions. I used deterministic sources during bring-up so I could isolate integration issues quickly before trusting live input paths.

I also used implementation outputs such as timing and utilization reports to evaluate whether the design is in a stable state for continued iteration. Seeing constraints met and resource usage remain in a manageable range gave me confidence that the architecture is viable for additional improvements.

Beyond the technical outputs, this project has helped me strengthen core engineering habits:

- break large systems into verifiable blocks,
- document assumptions and current limitations,
- iterate based on evidence rather than guesswork,
- and communicate progress clearly.

Those habits connect directly to the type of role I want after graduation, especially in design verification.

## Results and Current Status

The project is in active development and already demonstrates end-to-end system behavior across the major stages of the pipeline. Timing closure and resource usage indicate a workable baseline implementation, and the architecture is positioned for continued refinement. Current priorities include final integration polish, deeper validation coverage, and clearer presentation of measured behavior.

To support this progress, I also keep visual snapshots of both the software-facing spectrum display and the physical hardware setup so I can communicate system state clearly during reviews and showcase presentations.

![FFT analyzer web interface capture from my earlier build iteration](/images/fft-analyzer-ui.png)

![FPGA hardware setup photo used while validating the analyzer pipeline](/images/fft-hardware-setup.jpg)

For the Discovery Project Showcase, I will continue expanding this page with additional screenshots, refined results discussion, and final reflections on design tradeoffs and future improvements.
