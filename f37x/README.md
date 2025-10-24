# F37X Platform Support

This directory provides the complete F37X board port of the Sextans SpMM
accelerator.  It contains both the Vivado HLS 2019.1 entry point used to build
an IP core and a self-contained host application that prepares sparse/dense
matrices and communicates with the accelerator over XRT.  The accelerator keeps
the computation micro-architecture identical to the original Alveo design:
matrix tiles are streamed from HBM into on-chip BRAM/URAM buffers and processed
by eight processing elements.

## Directory layout

```
f37x/
├── host/              # Host-side executable sources
├── src/               # Vivado HLS kernel wrapper and compute pipeline
├── hls_sextans_f37x.tcl
└── README.md (this file)
```

- `src/sextans.cpp` – F37X specific top level that maps kernel memories to HBM
  bundles and exposes an AXI4-Lite control interface.
- `src/sextans_kernel.hpp` – Local copy of the Sextans compute pipeline used by
  the F37X wrapper.
- `host/host.cpp` – Host utility that loads matrices, programs the FPGA and
  verifies the accelerator output.
- `host/sparse_helper.h` & `host/mmio.h` – Matrix parsing utilities copied
  locally for ease of distribution.
- `host/xcl2/` – Lightweight helper used to interact with XRT.
- `hls_sextans_f37x.tcl` – Helper script for Vivado HLS 2019.1 that
  synthesises the kernel and exports an IP catalog component.

## Building with Vivado HLS 2019.1

1. Launch the tool from this directory:

   ```bash
   vivado_hls -f hls_sextans_f37x.tcl -tclargs <part_name> [clock_period_ns]
   ```

   - `part_name` should match the FPGA device used on the F37X board.
     A default of `xcvu37p-fsvh2892-2L-e` is used when no argument is
     provided.
   - `clock_period_ns` optionally overrides the default 3.3ns constraint.

2. After synthesis, the generated IP core is placed under the `ip/` directory.

## Customisation options

The F37X wrapper exposes several compile-time knobs:

- Override `F37X_AXI_BUNDLE(n)` to match the HBM AXI port naming used in your
  platform project.
- Override `F37X_CONTROL_BUNDLE` to change the AXI4-Lite control bundle name.
- `src/sextans_kernel.hpp` honours the following optional macros before
  inclusion:
  `SEXTANS_WINDOW_SIZE`, `SEXTANS_DEP_DIST_LOAD_STORE`,
  `SEXTANS_B_PARTITION_FACTOR` and `SEXTANS_URAM_DEPTH`.

These options make it straightforward to retarget the design without modifying
the kernel implementation.

## Building the host application

The host-side executable lives entirely under `host/`.  A minimal build script
could look like the following:

```bash
cd f37x/host
g++ -std=c++11 -I./xcl2 -O2 host.cpp xcl2/xcl2.cpp -lOpenCL -o sextans_host
```

At runtime the program expects the Vivado generated `.xclbin` and a sparse
matrix file in Matrix Market format:

```bash
./sextans_host <sextans_f37x.xclbin> <matrix_A.mtx> <N> [ALPHA] [BETA] [device_name]
```

- `N` must be a multiple of 8.
- `ALPHA` and `BETA` fall back to 0.85 and -2.06 when omitted.
- The optional `device_name` filters the OpenCL devices returned by XRT.  It is
  useful when the host machine exposes multiple Xilinx cards.  You can also set
  the `XCL_TARGET_DEVICE` environment variable instead of passing the argument.

