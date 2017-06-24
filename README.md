# UCSB ArchLab OpenTPU Project

The OpenTPU project provides a programmable framework for creating machine learning acceleration hardware.

This project is a an attempt at an open-source version of Google's Tensor Processing Unit (TPU). The TPU is Google's custom ASIC for accelerating the inference phase of neural network computations.

We used details from Google's paper titled "In-Datacentre Performance Analysis of a Tensor Processing Unit" (https://arxiv.org/abs/1704.04760) which is to appear at ISCA2017.

#### The OpenTPU is powered by PyRTL (http://ucsbarchlab.github.io/PyRTL/).

## Requirements

- Python 3
- PyRTL version >= 0.8.5
- numpy

Both PyRTL and numpy can be installed with pip; e.g., `pip install pyrtl`.

## How to Run

To run the simple matrix multiply test in both the hardware and functional simulators:
Make sure MATSIZE is set to 8 in config.py.
`python3 assembler.py simplemult.a`
`python3 runtpu.py simplemult.out simplemult_hostmem.npy simplemult_weights.npy`
`python3 sim.py simplemult.out simplemult_hostmem.npy simplemult_weights.npy`

To run the Boston housing data regression test in both the hardware and functional simulators:
Make sure MATSIZE is set to 16 in config.py.
`python3 assembler.py boston.a`
`python3 runtpu.py boston.out boston_inputs.npy boston_weights.npy`
`python3 sim.py boston.out boston_inputs.npy boston_weights.npy`


### Hardware Simulation
The executable hardware spec can be run using PyRTL's simulation features by running `runtpu.py`. The simulation expects as inputs a binary program and numpy array files containing the initial host memory and the weights.

Be aware that the size of the hardware Matrix Multiply unit is parametrizable --- double check `config.py` to make sure MATSIZE is what you expect.

### Functional Simulation
sim.py implements the functional simulator of OpenTPU. It reads in three cmd args: the assembly program, the host memory file, and the weights file. Due to the different quantization mechnisms between high-level applications (written in tensorflow) and OpenTPU, the simulator runs in two modes: 32b float mode and 8b int mode. The downsampling/quantization mechanism is consistent with the HW implementation of OpenTPU. It generates two sets of outputs, one set being 32b-float typed, the other 8b-int typed.

Example usage:

    python sim.py boston.out boston_input.npy boston_weights.npy

For the .npy data files, please refer to __Applications__ for how to generate them.

checker.py implementes a simple checking function to verify the results from HW, simulator and applications. It checkes the 32b-float application results against 32b-float simulator results and then checks the 8b-int simulator results against 8b-int HW results.

Example usage:

    python checker.py


## FAQs:
### What are the parts of the OpenTPU programmable hardware?
#### The OpenTPU hardware consists of:
- A weight FIFO
- A matrix multiply unit
- An activation unit (with ReLU and sigmoid activation functions)
- An accumulator
- A unified buffer

### What does the OpenTPU do?
#### This hardware is able to support:
- Inference phase of neural network computations

### What CAN'T the OpenTPU do?
#### The OpenTPU does not support:
- Convolution operations
- Pooling operations
- Programming normalization

### Does the OpenTPU implement all the instructions from the paper?
#### No, the OpenTPU currently supports the following instructions only:
- RHM (read host memory)
- WHM (write host memory)
- RW (read weights)
- MMC (matrix multiply)
- ACT (activate)
- NOP (no op)
- HLT (halt)

### I'm a Distinguished Hardware Engineer at Google and the Lead Architect of the TPU. I see many inefficiencies in your implementation.
Hi Norm! Tim welcomes you to Santa Barbara to talk about all things TPU :)


## Software details

### Writing a Program
OpenTPU uses no dynamic scheduling; all execution is fully determinstic* and the hardware relies on the compiler to correctly schedule operations and pad NOPs to handle delays. This OpenTPU release does \
not support "repeat" flags on instructions, so many NOPs are required to ensure correct execution.

*DRAM is a source of non-deterministic latency, discussed in the Memory Controller section of Microarchitecture.

### Latencies
The following gives the hardware execution latency for each instruction on OpenTPU:

RHM - _M_ cycles for reading _M_ vectors
WHM - _M_ cycles for writing _M_ vectors
RW - _N*N_/64 cycles for _N_x_N_ MM Array for DRAM transfer, and up to 3 additional cycles to propagate through the FIFO
MMC - _L+2N_ cycles, for _N_x_N_ MM Array and _L_ vectors multiplied in the instruction
ACT - _L+1_ cycles, for _L_ vectors activated in the instruction


## Microarchitecture

### Matrix Multiply (MM) Unit
The core of the compute of the OpenTPU is the parametrizable array of 8-bit Multiply-Accumulate Units (MACs), each consisting of an 8-bit integer multiplier and an integer adder of between 16 and 32 bits\
*. Each MAC has two buffers storing 8-bit weights (the second buffer allows weight programming to happen in parallel). Input vectors enter the array from the left, with values advancing one unit to the r\
ight each cycle. Each unit multiplies the input value by the active weight, adds it to the value from the unit above, and passes the result to the unit below. Input vectors are fed diagonally so that val\
ues align correctly as partial sums flow down the array.

*The multipliers produce 16-bit outputs; as values move down the columns of the array, each add produces 1 extra bit. Width is capped at 32, creating the potential for uncaught overflow.


### Accumulator Buffers
Result vectors from the MM Array are written to a software-specified address in a set of accumulator buffers. Instructions indicate whether values should be added into the value already at the address or\
 overwrite it. MM instructions read from the Unified Buffer (UB) and write to the accumulator buffers; activate instructions read from the accumulator buffers and write to the UB.


### Weight FIFO
At scale (256x256 MACs), a full matrix of weights (a "tile") is 64KB; to avoid stalls while weights are moved from off-chip weight DRAM, a 4-entry FIFO is used to buffer tiles. It is assumed the connecti\
on to the weight DRAM is a standard DDR interface moving data in 64-byte chunks (memory controllers are currently emulated with no simulated delay, so one chunk arrives each cycle). When an MM instructio\
n carries the "switch" flag, each MAC switches the active weight buffer as first vector of the instruction propagates through the array. Once it reaches the end of the first row, the FIFO begins feeding \
new weight values into the free buffers of the array. New weight values are passed down through the array each cycle until each row reaches its destination.


### Systolic Setup
Vectors are read all at once from the Unified Buffer, but must be fed diagonally into the MM Array. This is accomplished with a set of sequential buffers in a lower triangular configuration. The top valu\
e reaches the matrix immediately, the second after one cycle, the third after two, etc., so that each value reaches a MAC at the same time as the corresponding partial sum from the same source vector.


### Memory Controllers
Currently, memory controllers are emulated and have no delay. The connection to Host Memory is currently the size of one vector. The connection to the Weight DRAM uses a standard width of 64 bytes.

Because the emulated controllers can return a new value each cycle, the OpenTPU hardware simulation currently has no non-detministic delay. With a more accurate DRAM interface that may encounter dynamic \
delays, programs would need to either take care to schedule for the worst-case memory delay, or make use of another instruction to ensure memory operations complete before the values are required*.

*We note that the TPU "SYNC" instruction may fulfill this purpose, but is currently unimplemented on OpenTPU.


### Configuration
Unified Buffer size, Accumulator Buffer size, and the size of the MM Array can all be specified in config.py. However, the MM Array must always be square, and vectors/weights are always composed of 8-bit integers.
 