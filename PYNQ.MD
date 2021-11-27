## Tinygrad with PYNQ
### Goal of this project
[Tinygrad](https://github.com/geohot/tinygrad) is all about making it easy to write, contribute to, and accelerate deep learning. To make this possible there are only 11 (by my count) first-class operations that must be implemented to accelerate Tinygrad on a new architecture. To showcase how easy this can be I'm going to write hardware accelerators for each of these ops in VHDL (and HLS when necessary) and program a [PYNQ](http://www.pynq.io/) board with those overlays (PYNQ boards are just ZYNQ MPSoC's that can be reconfigured and interacted with at runtime with Python which is fitting for Tinygrad). Then, simply plug these PYNQ ops in to Tinygrad and you have an accelerated deep learning framework!


### What to expect
This is purely a learning experiment that a decent blog post might come out of. I don't expect this PYNQ accelerator to catch on or be fast, but I do expect to learn how to write VHDL, compile HLS, create a bitstream for an MPSoC, create PYNQ overlays, DMA into accelerators, and learn a bit about autograd. I think that the principles behind the [Cherry computer](https://geohot.github.io/blog/jekyll/update/2021/06/13/a-breakdown-of-ai-chip-companies.html) are exciting so why not prepare to be part of the battle against NVIDiA?

There are a few reasons why this PYNQ accelerator will most likely not be high performance:
- The MP of the MPSoC is dual ARM A9 cores. These are not the newest cores, being released in 2007. Better cores to run the rest of Tinygrad (everything but the first class ops) would improve performance greatly. 
- Every first class op (FCO) will require data transfer from Processing System (PS) memory/cache to Programmable Logic (PL) over AXI. This will go both ways since I don't know how to keep the data in PL.
- There isn't much intelligence going into this. No pipelining, no out-of-order execution, no routing between FCO accelerators. Basically, whatever numpy functions that implement FCOs will happen in PL instead of PS.

** actually I've been reading up on DMA with the PYNQ board and you may be able to allocate tensors in DRAM that the PL IP can directly access. That means all tensors could belong to "PL memory" so you don't have to allocate and deallocate every FCO call.

Anyways, I think I can get most of this done over the weekend. Weekend is my time, not job time. We'll see what roadblocks we come up against (probably MacOS and M1 core issues) but this should get done by Sunday with a blog post released by Friday. What the heck, I may even stream this.


Steps
1. Get Tinygrad working on A9 just CPU ops
2. Implement a FCO in HLS or VHDL and validate
3. Implement overlay with FCO and hook into it from Tinygrad
4. See performance difference
5. Repeat 2-4 until all FCOs are implemented

### Longer term craziness:
PYNQ lets you use asyncio to send off commands to PL and do other stuff while waiting for computation to finish. This could be very powerful if you have a bunch of functional units implementing FCOs and you know how to exploit parallelism in the architecture!

### Open questions
* Partial reconfiguration - can you optimally allocate FCO accelerators depending on the model you train?
* What size array should the units operate on? 
* How do you stream big matrix through small FCO accel? Overlay implements that?