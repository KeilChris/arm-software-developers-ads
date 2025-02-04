---
# User change
title: "From Arm NEON to SVE"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Arm NEON

Modern CPUs have vector units that operate in a SIMD fashion. This greatly improves application performance, depending on the vector width.

The Arm v7 ISA introduced Advanced SIMD or Arm NEON instructions. These instructions are supported on the latest Arm v8 and Arm v9 ISA. NEON registers are composed of 32 128-bit registers V0-V31 and support multiple data types: integer, single-precision (SP) floating-point and double-precision (DP) floating-point.

## Arm SVE

In order to alleviate restrictions regarding fixed-length vector sizes, Arm has designed the Scalable Vector Extension (SVE).

Arm SVE is vector-length agnostic, allowing vector width from 128 up to 2048 bits. This enables software to scale dynamically to any SVE capable Arm hardware. 

SVE is not an extension of NEON but a separate, optional extension of Arm v8-A with a new set of instruction encodings.
Initial focus is HPC and general-purpose server. SVE2 extends capabilities to enable more data-processing domains.

### SVE registers

SVE is a predicate-centric architecture with:

- Scalable vector registers
    - Z0-Z31 extends NEON's 128-bit V0-V31
    - Supported data types
        - packed DP, SP, half-precision (HP) floating-point elements
        - packed 64, 32, 16 and 8-bit integer elements
- Scalable predicate registers
    - P0-P15 predicated for loop/arithmetic control
    - 1/8th the size of SVE registers (1 bit/byte)
    - FFR: first fault register for software speculation

### Simple addition example

Here is an example code compiled for SVE (left) and for NEON (right):

{{< godbolt width="100%" height="700px" mode="diff" lopt="-O3 -march=armv8-a" ropt="-O3 -march=armv8-a+sve" src="int fun(double * restrict a, double * restrict b, int size)\n{\n  for (int i=0; i < size; ++i)\n  {\n    b[i] += a[i];\n  }\n}" >}}

Note how small the SVE assembly is in comparison to NEON. This is due to the predicate behaviour which avoids generating assembly for remainder loops (scalar operations performed when the iteration domain is not a multiple of the vector length). The following describes the behaviour of the SVE assembly. 

The first 4 lines initialize the registers R2, R3, R4. R2 corresponds to the array size, R3 to the loop index, and R4 to the vector length.

The instruction _whilelo_ initializes the predicate register as follows:

- Increment a counter for each predicate lane in P0, starting from 0 (the loop index),
- Set the corresponding lane as “active” (or true) if the index is less than R2.

The _ld1d_ instructions perform memory loads: these are predicated instructions. Starting from the index stored in R3 and if the corresponding predicate lanes in P0 are active, load elements from the the array A (R0) in Z0 and B (R1) in Z1. 

The next instruction _fadd_ is not predicated: values in Z0 and Z1 are added and the result is stored in Z0. However, only the predicated values are stored in array B (R1) in memory with _st1w_.

Finally, the value of the loop index R3 is incremented by the vector length R4 and the value of the loop index is updated:

- Increment a counter for each predicate lane in P0, starting from R3 (the loop index),
- Set the corresponding lane as “active” (or true) if the index is less than R2.

If all lanes are inactive, we break the loop.

This example illustrates the logic behind SVE: it keeps the code simple and increases vectorization.
