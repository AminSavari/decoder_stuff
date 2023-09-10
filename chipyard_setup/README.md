Assuming you have conda installed on your machine, you have to install conda-lock by using the following commands (use bash shell): 

```
 conda install -n base conda-lock=1.4
 conda activate base
```

when done, clone the chipyard repo (version 1.9.1):

```
 git clone https://github.com/ucb-bar/chipyard.git
 cd chipyard
 git checkout 1.9.1

```

Finally to build the necessary tools:

`./build-setup.sh riscv-tools`

If you are in the conda base environment, source the env.sh in the top directory of chipyard to set up the necessary varibles and paths.

`source env.sh` 

Now chipyard is ready to simulate your design.

Prepare your RTL and put your verilog code in `generators/rocket-chip/src/main/resources/vsrc/`. 

Use your "top" module and instanciate it in `RoccBlackBox.v`

Connect the ports as necessary.

Next, some scala config files need to be changed. The Config files that has to see or change are as follows:


1. in `./generators/rocket-chip/src/main/scala/tile/LazyRoCC.scala` you can see the implementation of the BlackBoxExample

2. in `./generators/rocket-chip/src/main/scala/subsystem/Configs.scala` change the follwing class:

<pre>
```scala
class WithRoccExample extends Config((site, here, up) => {
  case BuildRoCC => List(
    (p: Parameters) => {
        val accumulator = LazyModule(new AccumulatorExample(OpcodeSet.custom0, n = 4)(p))
        accumulator
    },
    (p: Parameters) => {
        val translator = LazyModule(new TranslatorExample(OpcodeSet.custom1)(p))
        translator
    },
    (p: Parameters) => {
        val counter = LazyModule(new CharacterCountExample(OpcodeSet.custom2)(p))
        counter
    },
    (p: Parameters) => {
      val blackbox = LazyModule(new BlackBoxExample(OpcodeSet.custom3, "RoccBlackBox")(p))
      blackbox
    })
})
```
</pre>

to:

<pre>
```scala
class WithRoccExample extends Config((site, here, up) => {
  case BuildRoCC => List(
    (p: Parameters) => {
      val blackbox = LazyModule(new BlackBoxExample(OpcodeSet.custom0, "RoccBlackBox")(p))
      blackbox
    })
})
```
</pre>

As you can notice the changes, we only need one accelerator to be connected to the rocket core which is the BlackBoxExample module defined in [1.]. So here we are building a RoCC interface which is capable of connecting 4 accelerators to each rocket core.


3. Finally, in `./generators/chipyard/src/main/scala/config/RocketConfigs.scala` add the following class:

<pre>
```scala
class RocketBlackBoxConfig extends Config(                         
  new freechips.rocketchip.subsystem.WithRoccExample() ++        // RoccExample is instanciated here 
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++         // single rocket-core
  new chipyard.config.AbstractConfig)
```
</pre>

This class is to add the RoCC wrapper for our top design into the chipyard framework.


---
Then go to `./sims/vcs` (or `./sims/verilator` if you don't have access to Synopsys tools), and run the following commands in order:

`make verilog CONFIG=RocketBlackBoxConfig`

This will generate the verilog for our design and all the other modules needed for the simulation (notice the CONFIG name is the same as class name define in [3.]
`make run-binary-debug CONFIG=RocketBlackBoxConfig BINARY=../../generators/rocket-chip/software/design.riscv USE_VPD=1`

NOTE: each time you made changes to your design, you have to run the `make verilog CONFIG=...` command before running the command above. 

NOTES: 
*if we only use the `run-binary` option no signal dump file will be generated (i.e. VCD or VPD files).
*again use the same CONFIG name as the class name
*to generate the binary for the RISCV core we have to write a C program filled with some ROCC_INSTRUCTIONS (The format and usage is defined in Rocc.h) and compiled for the RISCV target. To do this, I took two MakeFiles from Abel to compile the C program and generate the riscv binary file (i.e., .riscv file) for baremetal and OS-based development on RISCV architecture.

Just run make in the same directory as your .c code. 

### the Makefile for OS-Based RISCV 
<pre>
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
</pre>

### the Makefile for baremetal RISCV

<pre>
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
</pre>

all the files to binary generation are included in ./software directory of this repo.

