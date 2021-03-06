export

VERILOG_DIRS	:= $(realpath src)
VERILOG_INCS	= $(addprefix -I,$(VERILOG_DIRS))

SCRIPT_DIR	= scripts
YOSYS_SCRIPT	= $(SCRIPT_DIR)/yosys
VERILOG_SRCS	:= $(shell find $(VERILOG_DIRS) -maxdepth 1 -name '*.v')

TEST_DIR	:= $(realpath test)

TOP_MODULE	= top
TOP_MODULE_SRC	= $(realpath src/$(TOP_MODULE).v)

.PHONY: prog
prog:
	openocd -s scripts/openocd -f openocd.cfg

# dont regenerate bitstream automatically, prevents users from needed
# to generate bitstream.
# .PHONY: prog
# prog: top.svf scripts/openocd/*
# 	openocd -s scripts/openocd -f openocd.cfg

.PHONY: bitstream
bitstream: top.svf

top.svf: top.bit scripts/vivado/svf.tcl
	vivado -nolog -nojournal -mode batch -source scripts/vivado/svf.tcl

flash.bin: top.bit
	vivado -nolog -nojournal -mode batch -source scripts/vivado/flash_bitstream.tcl

top.edif: src/*
	$(MAKE) -C $(YOSYS_SCRIPT)

top.bit: src/* env.sh
	vivado -nolog -nojournal -mode batch -source scripts/vivado/synth.tcl

env.sh: $(SCRIPT_DIR)/roms.py $(SCRIPT_DIR)/fir.py $(SCRIPT_DIR)/fft.py $(SCRIPT_DIR)/window.py
	python $(SCRIPT_DIR)/roms.py

.PHONY: lint
lint:
	$(eval TOP_MODULE = fir)
	verilator $(VERILOG_INCS) \
		--language "1364-2005" \
		--lint-only \
		--bbox-sys \
		--bbox-unsup \
		--Wall \
		--top-module $(TOP_MODULE) \
		$(realpath src/$(TOP_MODULE).v)

# cocotb simulations
.PHONY: test
test: test_top test_fft test_fir

.PHONY: test_top
test_top:
	$(eval TOP_MODULE = top)
	$(MAKE) -C $(TEST_DIR) cocotb

.PHONY: test_fft
test_fft:
	$(eval TOP_MODULE = fft)
	$(MAKE) -C $(TEST_DIR) cocotb

.PHONY: test_fir
test_fir:
	$(eval TOP_MODULE = fir)
	$(MAKE) -C $(TEST_DIR) cocotb

# traditional testbenches
.PHONY: sim
sim:
	iverilog -DTOP_SIMULATE=1 -s top_tb src/*.v -Isrc && ./a.out

.PHONY: sim_adf4158
sim_adf4158:
	iverilog -DADF4158_SIMULATE=1 -s adf4158_tb src/adf4158.v && ./a.out

.PHONY: sim_ff_sync
sim_ff_sync:
	iverilog -DFF_SYNC_SIMULATE=1 -s ff_sync_tb src/ff_sync.v && ./a.out

.PHONY: sim_async_fifo
sim_async_fifo:
	iverilog -DFIFO_SIMULATE=1 -s async_fifo_tb src/async_fifo.v -Isrc && ./a.out

# formal verification
.PHONY: formal
formal: formal_dual_ff formal_pll_sync_ctr

.PHONY: formal_dual_ff
formal_dual_ff:
	$(eval TOP_MODULE = dual_ff)
	$(MAKE) -C $(YOSYS_SCRIPT) formal

.PHONY: formal_pll_sync_ctr
formal_pll_sync_ctr:
	$(eval TOP_MODULE = pll_sync_ctr)
	$(MAKE) -C $(YOSYS_SCRIPT) formal

.PHONY: clean
clean:
	rm -f a.out
	rm -f flash.prm
	rm -f usage_statistics_webtalk*
	rm -rf .Xil
