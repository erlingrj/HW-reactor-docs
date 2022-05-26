# HW/SW Reactors
In a HW/SW Reactor I think we generally want the HW Reactors to run under the logical timeline of the SW Reactor runtime. I.e. we dont want HW Reactors to progress in lockstep with physical time in parallel with the SW Reactors. To achieve this we must limit expressiveness of the HW Reactors. By removing Actions and Timers from HW Reactors we no longer need to stay in lock-step with Physical time. Now the HW Reactor subsystem can be "idle" until it is triggered by the SW scheduler through a write to the control and status (CSR) reguster of the top-level HW Reactor.

Currently I can think of 3 targets for HW/SW Reactors
1. MPSoC FPGAs (E.g. Xilinx Zynq) ARM cores + FPGA on a single chip
2. Datacenter FPGAs (E.g. Xilinx Alveo) FPGA board with PCIe to be connected with x86 system
3. FlexPRET+HW Reactors

An open question which I have not considered that much yet is wether we can, or whether it makes sense to, wrap accelerators running on GPUs++ as HW Reactors.  