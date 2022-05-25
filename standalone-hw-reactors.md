# Working document on the standalone HW Reactor framework

## Specs
- Support Both HDL(Chisel) and HLS(Vitis HLS)
- Aim for being cross-platform (not only AMD Xilinx FPGAs)
- ...


## Implementation
Both HLS and HDL target will be built ontop of a base Chisel Reactor framework which provides the basics.
THe proposed implementation is inspired by Esterel/SCCharts where there is really no distinction between Physical and Logical time.
Each Reactor/Reaction has to be associated with a WCET given in a number of Clock Cycles. Based on this the Critical Path of the Reactor graph can be established which sets the upper bound on the frequency of the global clock signal. The frequency will be a number of FPGA clock cycles which in turn will have a physical clock frequency. E.g. 100MHz.
During each Global Clock Cycle (GCC) the Reactors are enabled (their guards are enabled) in a pre-defined order (which is decided by the dependencies)


### Clocks
1. FPGA clock. defined by the critical path through the design. Typically 100MHz-300MHz. Give like e.g. 30CC
2. Global clock. Defined by critical path through the Reactor graph. Is a number of CCs, e.g. 100CC. Is denoted like 1GCC (Global Clock Cycle)


### Ports
The ports/interfaces are very crucial component. In SW Reactors the ports are in generally "call-by-reference", e.g. pointers to memory. 
All ports (both Input and Output) buffer the values being sent. The connection between Input and Output ports are realized through a Ready-Valid interface
We should support the following ports, this is largely inspired by how Vitis HLS (from AMD Xilinx) synthesizes interfaces from C++ functions.

#### Call-by-value ports
- A call-by-value port carries the actual data travels directly between the Reactors on wires.
- This is the lowest hanging fruit IMO. 
- An important limitation is that the data must be transferred in a single transaction (e.g. no array-like structure)
- See HelloWorld and Gcd examples for such a port

#### Call-by-reference ports
- In call-by-value ports, data is written and read from a memory shared by the Reactors.
- This is either a off-chip DRAM or an on-chip BRAM. 
- The ports will now consist of 1) A read/write interface to a RAM 2) A trigger interface to signal that there is data. This trigger interface should maybe also include the address into the RAM where the data is stored.
- This would be used with arrays that needs random access. E.g. an image which we want to apply a kernel on
- See [ImageProcHLS example](./examples/ImageProcHLS/ImageProcHLS.lf) for such an example

#### FIFO ports
- This is an special case of call-by-reference ports. Only we limit ourselves to accessing the array sequentially. If you e.g. want to add a constant weight to all pixel values of an image.
- This is also realized by a BRAM memory decoupling the two Reactors. Only the BRAM is implementing a FIFO.
- Here we only need ready-valid interface from the port to the FIFO.
- The receiving Reactor will not start executing before the previous is completely done. There is really room for pipelining when you have a decoupling FIFO between 2 Reactors, but it is not clear to me ATM how Reactor semantics could be preserved while pipelining.  
- See [ImageProcFIFOHLS example](./examples/ImageProcFIFOHLS/ImageProcFIFOHLS.lf) for an example


### Actions
- An Action is a Count-down timer which enables a Reaction input when it reaches zero.
- The count-down timer is decremented each time the Reactor is enabled by the Top Scheduler. This ensures a consistent view of the surroundings for the Reactor
- We have to put an upper bound on the number of outstanding Actions that can be scheduled.
- A timer is a simple Action that is re-armed automatically and which carries no value

### State
- Each Reactor can contain state which is either stored in a BRAM (with corresponding 1CC access latency) or a Register/Flip Flop with no access latency.

### Reactions
- Reactions are user-specified computational blocks.
- In HDL the Reaction is a HW block which will keep executing each FPGA Clock Cycle (CC) until the user makes a call to an API function (lf_done())
- In HDL the Reaction will probably be implemented as a state machine. See GcdChisel example for how a state machine is implemented in a Reaction
- In HLS the reaction is a C/C++ program which is executed until completion. The user must be careful not to include things like infinte loops, recursion and other non-synthesizable constructs.
- In HLS the user must provide suitable #pragmas to the HLS compiler for pipelining etc.

### Connections
- The user will specify the connection between Reactions, Actions, Ports etc in a config file which is passed to the Top Level Reactor
- At Chisel compile-time this config file will be read and the appropriate connections will be made between the components.



#### Logical execution time
- Each Reaction must be associated with a WCLET (in FPGA Clock Cycles). This is to calculate the critical path through the Reactor/Reaction Graph.
