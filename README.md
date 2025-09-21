# Microwatt-Accel: High-Speed Arithmetic Slice for the OpenPOWER Microwatt Core (SKY130)

## Abstract

We accelerate the Microwatt POWER core by replacing its baseline integer/floating-point arithmetic with a custom, timing-driven arithmetic slice: hybrid parallel-prefix adders, Booth-Dadda multipliers, a fast SRT-style divider, and optimized popcount/leading-zero logic. The slice is ISA-compatible, drop-in at the execute stage, and verified with RTL testbenches plus post-layout STA/SDF. All RTL, OpenLane configs, and (if we use AI for code) prompt logs are open-sourced per the challenge rules. Target: 1.5–3.0× cycle-level speedups on arithmetic-heavy kernels within the OpenFrame user area on SKY130.

## Motivation

Microwatt already includes integer multiply/divide and an FPU, but they are written for clarity and portability, not maximum PPA. By swapping in timing-aware arithmetic macros (prefix adders tuned for wire delay at 130 nm, pipelined Booth-Dadda multiplier, faster divider, better bit-ops), we can lift core IPC on arithmetic-heavy workloads while keeping the pipeline and ISA intact. We’ll integrate within Microwatt’s execution path (execute1.vhdl, multiply*.vhdl, divider.vhdl, fpu.vhdl) using the same interfaces and updated latency parameters.

## What we’re building
1) Arithmetic Slice (AS) IP (SKY130-ready)

   - Hybrid Prefix Adders (64-bit): Han-Carlson/Brent-Kung hybrids to balance fanout and wirelength at 130 nm; reused in integer ALU and FPU mantissa paths.

   - Booth-Dadda Multiplier (32/64-bit): Radix-4 Booth encoding + Dadda tree + final prefix adder; 1–2 pipeline stages (configurable) to hit timing.

Fast Divider: Iterative SRT-4 (2 bits/iter) with early-exit and speculative correction; 1–2 cycle/quotient-chunk micro-pipeline.

Bit-ops: Optimized popcount/leading-zero (tree structures), wiring mindful; replaces countbits.vhdl critical cones.

Clean SKY130 mapping: Only standard-cell logic; no analog or SRAM macros; STA via OpenSTA; sign-off in OpenLane (chipIgnite flow). Challenge requires SKY130 + standard cells + passing precheck—this fits. 
ChipFoundry

2) Integration into Microwatt

Drop-in replacement of multiply*.vhdl, divider.vhdl, FPU mantissa adder/multiplier, and ALU adder cone; minor changes in execute1.vhdl for updated latencies/bypass.

Interfaces preserved (operand/valid/ready/exception) to avoid architectural changes.

Config flags to select baseline vs. accelerated units at synthesis.

Repo evidence of these blocks existing today (for us to replace/enhance): multiply.vhdl, multiply-32s.vhdl, divider.vhdl, fpu.vhdl, countbits.vhdl.




# OpenFrame Overview

The OpenFrame Project provides an empty harness chip that differs significantly from the Caravel and Caravan designs. Unlike Caravel and Caravan, which include integrated SoCs and additional features, OpenFrame offers only the essential padframe, providing users with a clean slate for their custom designs.

<img width="256" alt="Screenshot 2024-06-24 at 12 53 39 PM" src="https://github.com/efabless/openframe_timer_example/assets/67271180/ff58b58b-b9c8-4d5e-b9bc-bf344355fa80">

## Key Characteristics of OpenFrame

1. **Minimalist Design:** 
   - No integrated SoC or additional circuitry.
   - Only includes the padframe, a power-on-reset circuit, and a digital ROM containing the 32-bit project ID.

2. **Padframe Compatibility:**
   - The padframe design and pin placements match those of the Caravel and Caravan chips, ensuring compatibility and ease of transition between designs.
   - Pin types are identical, with power and ground pins positioned similarly and the same power domains available.

3. **Flexibility:**
   - Provides full access to all GPIO controls.
   - Maximizes the user project area, allowing for greater customization and integration of alternative SoCs or user-specific projects at the same hierarchy level.

4. **Simplified I/O:**
   - Pins that previously connected to CPU functions (e.g., flash controller interface, SPI interface, UART) are now repurposed as general-purpose I/O, offering flexibility for various applications.

The OpenFrame harness is ideal for those looking to implement custom SoCs or integrate user projects without the constraints of an existing SoC.

## Features

1. 44 configurable GPIOs.
2. User area of approximately 15mm².
3. Supports digital, analog, or mixed-signal designs.

# openframe_timer_example

This example implements a simple timer and connects it to the GPIOs.

## Installation and Setup

First, clone the repository:

```bash
git clone https://github.com/efabless/openframe_timer_example.git
cd openframe_timer_example
```

Then, download all dependencies:

```bash
make setup
```

## Hardening the Design

In this example, we will harden the timer. You will need to harden your own design similarly.

```bash
make user_proj_timer
```

Once you have hardened your design, integrate it into the OpenFrame wrapper:

```bash
make openframe_project_wrapper
```

## Important Notes

1. **Connecting to Power:**
   - Ensure your design is connected to power using the power pins on the wrapper.
   - Use the `vccd1_connection` and `vssd1_connection` macros, which contain the necessary vias and nets for power connections.

2. **Flattening the Design:**
   - If you plan to flatten your design within the `openframe_project_wrapper`, do not buffer the analog pins using standard cells.

3. **Running Custom Steps:**
   - Execute the custom step in OpenLane that copies the power pins from the template DEF. If this step is skipped, the precheck will fail, and your design will not be powered.
