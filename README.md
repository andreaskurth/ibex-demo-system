# Ibex Demo System

This an example RISC-V SoC targeting the Arty-A7 FPGA board. It comprises the
[lowRISC Ibex core](https://www.github.com/lowrisc/ibex) along with the
following features:

* RISC-V debug support (using the [PULP RISC-V Debug Module](https://github.com/pulp-platform/riscv-dbg))
* A UART (transmit only for now)
* GPIO (output only for now)
* Timer
* SPI
* A basic peripheral to write ASCII output to a file and halt simulation from software

Debug can be used via a USB connection to the Arty-A7 board. No external JTAG
probe is required.

## Software Requirements

* Xilinx Vivado - https://www.xilinx.com/support/download.html
* rv32imc GCC toolchain - lowRISC provide one:
  https://github.com/lowRISC/lowrisc-toolchains/releases
* cmake
* python3 - Additional python dependencies in python-requirements.txt installed
  with pip
* openocd

To install python dependencies use pip, you may wish to do this inside a virtual
environment to avoid disturbing you current python setup (note it uses a lowRISC
fork of edalize and FuseSoC so if you already use these a virtual environment is
recommended)

```bash
# Setup python venv
python3 -m venv .
source ./bin/activate

# Install python requirements
pip3 install -r python-requirements.txt
```

You may need to run the last command twice if you get the following error:
`ERROR: Failed building wheel for fusesoc`

## Building Software

First the software must be built. This is provide an initial binary for the FPGA
build.

```
cd sw
mkdir build
cd build
cmake ../
make
```

Note the FPGA build relies on a fixed path to the initial binary (blank.vmem) so
if you want to create your build directory elsewhere you need to adjust the path
in `ibex_demo_system.core`

## Building FPGA
FuseSoC handles the FPGA build. Vivado tools must be setup beforehand. From the
repository root:

```
source /path/to/vivado/settings64.sh
fusesoc --cores-root=. run --target=synth --setup --build lowrisc:ibex:demo_system
```
## Programming FPGA
To program the FPGA, either use FuseSoC again

```
fusesoc --cores-root=. run --target=synth --run lowrisc:ibex:demo_system
```

Or use the Vivado GUI

```
make -C ./build/lowrisc_ibex_demo_system_0/synth-vivado/ build-gui
```

Inside Vivado you do not have to run the synthesis, the implementation or generate the bitstream.
Simply click on "Open Hardware Manager", then on "Auto Connect" and finally on "Program Device".

## Loading an application to the programmed FPGA

The util/load_demo_system.sh script can be used to load and run an application. You
can choose to immediately run it or begin halted, allowing you to attach a
debugger.

```bash
# Run demo
./util/load_demo_system.sh run ./sw/build/demo/demo

# Load demo and start halted awaiting a debugger
./util/load_demo_system.sh halt ./sw/build/demo/demo
```

To view terminal output use screen:

```bash
# Look in /dev to see available ttyUSB devices
screen /dev/ttyUSB1 115200
```

If you see an immediate `[screen is terminating]`, it may mean that you need super user rights.
In this case, you may try using `sudo`.

## Debugging an application

Either load an application and halt (see above) or start a new OpenOCD instance

```
openocd -f util/arty-a7-openocd-cfg.tcl
```

Then run GDB against the running binary and connect to localhost:3333 as a
remote target

```
riscv32-unknown-elf-gdb ./sw/build/demo/demo

(gdb) target extended-remote localhost:3333
```
## Building Simulation

The Demo System simulator binary can be built via FuseSoC. From the Ibex
repository root run:

```
fusesoc --cores-root=. run --target=sim --tool=verilator --setup --build lowrisc:ibex:demo_system
```
## Running the Simulator

Having built the simulator and software, from the Ibex repository root run:

```
./build/lowrisc_ibex_demo_system_0/sim-verilator/Vibex_demo_system [-t] --meminit=ram,<sw_elf_file>
```

`<sw_elf_file>` should be a path to an ELF file  (or alternatively a vmem file)
built as described above. Use `./sw/build/demo/demo` to run the `demo`
binary.

Pass `-t` to get an FST trace of execution that can be viewed with
[GTKWave](http://gtkwave.sourceforge.net/).

```
Simulation statistics
=====================
Executed cycles:  5899491
Wallclock time:   1.934 s
Simulation speed: 3.05041e+06 cycles/s (3050.41 kHz)

Performance Counters
====================
Cycles:                     457
NONE:                       0
Instructions Retired:       296
LSU Busy:                   108
Fetch Wait:                 20
Loads:                      53
Stores:                     55
Jumps:                      21
Conditional Branches:       12
Taken Conditional Branches: 7
Compressed Instructions:    164
Multiply Wait:              0
Divide Wait:                0
```
