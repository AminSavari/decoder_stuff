# Chipyard Verilog Integration

This guide provides instructions for integrating your Verilog design with the Chipyard framework.

## Prerequisites

Make sure you have the following prerequisites installed on your machine:

- [Conda](https://docs.conda.io/en/latest/)
- [Git](https://git-scm.com/)

## Installation

### Step 1: Install conda-lock

Assuming you have conda installed on your machine, install conda-lock by running the following commands in a bash shell:

```bash
conda install -n base conda-lock=1.4
conda activate base
```

### Step 2: Clone Chipyard Repository

Clone the Chipyard repository (version 1.9.1) using Git:

```bash
git clone https://github.com/ucb-bar/chipyard.git
cd chipyard
git checkout 1.9.1
```
### Step 3: Build Necessary Tools

To build the necessary tools, run:

```bash

./build-setup.sh riscv-tools
```
If you are in the conda base environment, source the env.sh file in the top directory of Chipyard to set up the necessary variables and paths:

```bash

source env.sh
```
Now, Chipyard is ready to simulate your design.

### Integrating Your Verilog Design

    Prepare your RTL (Register-Transfer Level) and place your Verilog code in generators/rocket-chip/src/main/resources/vsrc/.

    Use your "top" module and instantiate it in RoccBlackBox.v.

    Connect the ports as necessary.

### Configuration Changes

In the Chipyard framework, some Scala configuration files need to be modified.

    In `./generators/rocket-chip/src/main/scala/tile/LazyRoCC.scala`, you can see the implementation of the BlackBoxExample. Ensure that you have the necessary RoCC accelerator defined.

    In `./generators/rocket-chip/src/main/scala/subsystem/Configs.scala`, modify the configuration class as follows:

```scala

class WithRoccExample extends Config((site, here, up) => {
  case BuildRoCC => List(
    (p: Parameters) => {
      val blackbox = LazyModule(new BlackBoxExample(OpcodeSet.custom0, "RoccBlackBox")(p))
      blackbox
    })
})
```
This configuration defines the RoCC interface for your top design.

    Finally, `in ./generators/chipyard/src/main/scala/config/RocketConfigs.scala`, add the following class:

```scala

class RocketBlackBoxConfig extends Config(                         
  new freechips.rocketchip.subsystem.WithRoccExample() ++
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++
  new chipyard.config.AbstractConfig)
```
This class adds the RoCC wrapper for your top design into the Chipyard framework.
Simulation

Navigate to `./sims/vcs` (or `./sims/verilator` if you don't have access to Synopsys tools) and run the following commands in order:

```bash

make verilog CONFIG=RocketBlackBoxConfig
```
This generates the Verilog for your design.

```bash

make run-binary-debug CONFIG=RocketBlackBoxConfig BINARY=../../generators/rocket-chip/software/design.riscv USE_VPD=1
```
This command runs the simulation with signal dump file generation.
NOTE: each time you made changes to your design, you have to run the `make verilog CONFIG=...` command before running the command above. 

NOTES: 
* if we only use the `run-binary` option no signal dump file will be generated (i.e. VCD or VPD files).
* use the same CONFIG name as the class name in `RocketConfigs.scala`

### Generating RISC-V Binary

To generate the binary for the RISC-V core, create a C program with ROCC instructions (format and usage defined in Rocc.h) and compile it for the RISC-V target. Two MakeFiles are provided for both OS-based and baremetal RISC-V development.



### Makefile for OS-Based RISC-V
```make
SHELL = bash
CC = riscv64-unknown-linux-gnu-gcc
CFLAGS = -std=c99 -fno-common -fno-builtin-printf -static -Wall

SOURCE = program_name.c
TARGET = target_name # the name of .riscv file

all: $(TARGET)

$(TARGET): $(SOURCE)
	$(CC) $(CFLAGS) -o $(TARGET) $(SOURCE) -lpthread

.PHONY: clean
clean:
	rm -f $(TARGET)
```

### Makefile for Baremetal RISC-V


```make
RISCV_PREFIX ?= riscv64-unknown-elf-
RISCV_GCC ?= $(RISCV_PREFIX)gcc
RISCV_GCC_OPTS ?= -DPREALLOCATE=1 -mcmodel=medany -static -std=gnu99 -O2 -ffast-math -fno-common -fno-builtin-printf -fno-tree-loop-distribute-patterns
RISCV_LINK ?= $(RISCV_GCC) -T $(src_dir)/common/test.ld $(incs)
RISCV_LINK_OPTS ?= -static -nostdlib -nostartfiles -lm -lgcc -T link.ld
RISCV_OBJDUMP ?= $(RISCV_PREFIX)objdump --disassemble-all --disassemble-zeroes --section=.text --section=.text.startup --section=.text.init --section=.data
RISCV_SIM ?= spike --isa=rv$(XLEN)gc

target_name.riscv : program_name.c syscalls.c crt.S
	$(RISCV_GCC) $(RISCV_GCC_OPTS) -o $@ $^ $(RISCV_LINK_OPTS)
```

Make sure to replace program_name.c and target_name with your C program and desired binary name.

With these steps, you should be able to integrate your Verilog design into the Chipyard framework and generate the necessary binaries for RISC-V.
