## tinygrad with PYNQ
### Goal of this project
[tinygrad](https://github.com/geohot/tinygrad) is all about making it easy to write, contribute to, and accelerate deep learning. To make this possible there are only 11 (by my count) first-class operations that must be implemented to accelerate tinygrad on a new architecture. To showcase how easy this can be I'm going to write hardware accelerators for each of these ops in VHDL (and HLS when necessary) and program a [PYNQ](http://www.pynq.io/) board with those overlays (PYNQ boards are just ZYNQ MPSoC's that can be reconfigured and interacted with at runtime with Python which is fitting for tinygrad). Then, simply plug these PYNQ ops in to tinygrad and you have an accelerated deep learning framework!

My specific PYNQ board has [these specs](https://www.tul.com.tw/productspynq-z2.html).


### What to expect
This is purely a learning experiment that a decent blog post might come out of. I don't expect this PYNQ accelerator to catch on or be fast, but I do expect to learn how to write VHDL, compile HLS, create a bitstream for an MPSoC, create PYNQ overlays, DMA into accelerators, and learn a bit about autograd. I think that the principles behind the [Cherry computer](https://geohot.github.io/blog/jekyll/update/2021/06/13/a-breakdown-of-ai-chip-companies.html) are exciting so why not prepare to be part of the battle against NVIDiA?

There are a few reasons why this PYNQ accelerator will most likely not be high performance:
- The MP of the MPSoC is dual ARM A9 cores. These are not the newest cores, being released in 2007. Better cores to run the rest of tinygrad (everything but the first class ops) would improve performance greatly. 
- Every first class op (FCO) will require data transfer from Processing System (PS) memory/cache to Programmable Logic (PL) over AXI. This will go both ways since I don't know how to keep the data in PL.
- There isn't much intelligence going into this. No pipelining, no out-of-order execution, no routing between FCO accelerators. Basically, whatever numpy functions that implement FCOs will happen in PL instead of PS.

** actually I've been reading up on DMA with the PYNQ board and you may be able to allocate tensors in DRAM that the PL IP can directly access. That means all tensors could belong to "PL memory" so you don't have to allocate and deallocate every FCO call.

Anyways, I think I can get most of this done over the weekend. Weekend is my time, not job time. We'll see what roadblocks we come up against (probably MacOS and M1 core issues) but this should get done by Sunday with a blog post released by Friday. What the heck, I may even stream this.


Steps
1. Get tinygrad working on A9 just CPU ops
2. Implement a FCO in HLS or VHDL and validate
3. Implement overlay with FCO and hook into it from tinygrad
4. See performance difference
5. Repeat 2-4 until all FCOs are implemented

### Longer term craziness:
PYNQ lets you use asyncio to send off commands to PL and do other stuff while waiting for computation to finish. This could be very powerful if you have a bunch of functional units implementing FCOs and you know how to exploit parallelism in the architecture!

### Open questions
* Partial reconfiguration - can you optimally allocate FCO accelerators depending on the model you train?
* What size array should the units operate on? 
* How do you stream big matrix through small FCO accel? Overlay implements that?

I can only do HLS and synthesis on my Ubuntu desktop. Doing remote desktop from Mac is so frustrating I'd rather go in person. From Mac I can write and verify Verilog for ops. Ops are combinational (I think) so logic to fetch from AXI stream is generic and outside of ops.

### Vector Unit
Should vector unit input size be same as memory bus width? That might be 128 bits.
Can I implement matrix unit as a series of calls to vector unit?

Implement vector unit in Python first (like Cherry emulation).

Learn Verilog from [here](https://verilogguide.readthedocs.io/en/latest/verilog/overview.html) in free time, keeping in mind goal of creating very wide unary and binary ALUs -- should these be the same hardware? Where should I get my matmul accel from? How do you read data from AXI slave, pass it to ALU, send it back to AXI master in HDL? People do it in HLS but what is the construct in HLS?

Soon implement some important PyTorch examples in tinygrad.



# HLS Notes
### AXI Streams
My accel will have to read arrays from an AXI stream and write outputs to an AXI stream. To accelerate a vector process using this one could first load the entire array into memory, operate on each element (or pair of elements for binary ops), and then write entire result array to output AXI stream. Alternatively, for many element-wise vector ops you could have enough binary/unary operator units for the max burst size of the AXI stream, computing the results as inputs stream in and writing immediately to the output stream.

I have no idea how you implement matmul given that A and B are contiguous in memory. Perhaps you read B rows in typical contiguous format and place them in B's BRAM columns to transpose it on read.

My PYNQ board has 4.9MB of BRAM (630KB of fast BRAM?) which can handle 1.2M elements. That's 76 matrices which means we can have multiple matmul accelerators or a <b>cube</b> accelerator.

Some guy has a great [DMA tutorial](https://discuss.pynq.io/t/tutorial-pynq-dma-part-1-hardware-design/3133) on the PYNQ forums.

And here is a [Xilinx guide](https://www.xilinx.com/support/documentation/application_notes/xapp1209-designing-protocol-processing-systems-hls.pdf) on implementing 'Protocol Processing'


I can just write and test the HLS in software at any time!! Plus implement it in Tinygrad.


### Little Q/A
Q: Where do HLS arrays live (when you do HLS ARRAY PARTITION)?

A: [Examples with HLS streaming](https://github.com/Xilinx/Vitis-HLS-Introductory-Examples/tree/master/Interface/Streaming) and [lecture notes](https://www.csl.cornell.edu/courses/ece5775/pdf/lecture02.pdf). Arrays generally map to ROMs or RAMs. They can be partitioned across multiple RAMs with certain #pragmas. Partition with element-by-element interleaving or block chunking. 

Q: Is 'transpose on copy' free? (AKA read rows in DRAM into cols in BRAM and vice versa for writing) Does a BRAM know it's a row BRAM?

A:

Q: Does PYNQ have a place in research? Do people actually use it or is it too underpowered to mean much? 

A: 

Q: Has anybody implemented Systolic array matmul on PYNQ?

A: [somebody did](https://github.com/neasotho/SystolicArray_FPGA)

Q: How do you factor big matmul into many small (128x128) matmul?

A: [Tiled matmul](https://penny-xu.github.io/blog/tiled-matrix-multiplication)... do it in software or hardware.

Q: How bad is it if we implement matmul via N^2 dot products? You could partially optimize the inner loop by keeping A[row,:] or B[:, col] around for one row or col computation of C.

A: {benchmarking results}