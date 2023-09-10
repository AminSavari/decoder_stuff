First we have to have our RTL of the system ready. It is better to have a single verilog file that has all the blocks of the design. For this, we have a Makefile that merges all verilog/systemverilog into a single file and exclude the ones that are only useful for simulation.


