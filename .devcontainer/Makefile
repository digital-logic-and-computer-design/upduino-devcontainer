# Based on https://tech.davis-hansson.com/p/make/
SHELL := bash
.ONESHELL:
.SHELLFLAGS := -eu -o pipefail -c
.DELETE_ON_ERROR:
MAKEFLAGS += --warn-undefined-variables
MAKEFLAGS += --no-builtin-rules
# Assume a recent version of GNU Make
.RECIPEPREFIX = >

MAKEFILE_PATH = $(dir $(realpath $(firstword $(MAKEFILE_LIST))))
IMAGES_DIR := $(MAKEFILE_PATH)/../images

# Default to nothing
default:

# Include the dependencies file
# If there's not a dependencies.mk file, stop with an error
ifeq ($(wildcard dependencies.mk),)
  $(error No dependencies.mk file found)
else
  include dependencies.mk
endif

#### HDL Build Rules (Essentialy a shell script ####


%.rtl.json:
>     @echo "Building $(filter %.top.sv, $^)"
>     yosys -p "read_verilog -sv $^ ; hierarchy -auto-top; proc; opt; fsm -expand; memory -nomap; wreduce -memx; opt_clean;" -o $@


## Parsing out "top" module (.top.sv)
# %.rtl.json:
# >     @echo "Building $(filter %.top.sv, $^)"
# ifeq ($(filter %.top.sv, $^),"")
# >     @echo No top module found, using auto-top
# >     yosys -p "read_verilog -sv $^ ; hierarchy -auto-top; proc; opt; fsm -expand; memory -nomap; wreduce -memx; opt_clean;" -o $@
# else
# >     @echo Top: $(filter %.top.sv, $^)
# >     @echo Module: $(subst .top.sv, , $(filter %.top.sv, $^))
# >     yosys -p "read_verilog -sv $^ ; hierarchy -auto-top; proc; opt; fsm -expand; memory -nomap; wreduce -memx; opt_clean;" -o $@
# >   yosys -p "read_verilog -sv $^ ; hierarchy -top $(subst .top.sv, , $(filter %.top.sv, $^)) ; proc; opt; fsm -expand; memory -nomap; wreduce -memx; opt_clean;" -o $@
# endif


%.ice40.jpg:
>  yosys -p "read_verilog -sv $^ ; hierarchy -auto-top; synth_ice40; show -format jpg -prefix $(basename $@)"

%.ice40.dot:
>  yosys -p "read_verilog -sv $^ ; hierarchy -auto-top; synth_ice40; show -prefix $(basename $@)"


%.aig.jpg:
>  yosys -p "read_verilog -sv $^ ; hierarchy -auto-top; proc; opt; fsm ; opt; memory; aigmap; show -format jpg -prefix $(basename $@)"

%.ice40.rtl.json:
>  yosys -p "read_verilog -sv $^ ; hierarchy -auto-top; proc; opt_clean; fsm -expand; memory -nomap; wreduce -memx; opt_clean; synth_ice40; " -o $@

# Always re-run test benches
%.vcd:
>    rm -f $(@:.vcd=.sim)
>    iverilog -g 2012 $(filter %.sv, $^) -o $(@:.vcd=.sim)
>    (cd $(dir $@) && vvp $(notdir $(@:.vcd=.sim)) )



### BIN / Work in progress
# TODO: Decouple this into three steps
# # Check if there's a manta / debug JSON file.  If so, build it first (to manta.sv)
# Print an error if no PCF file
%.bin:
>     @echo "Building $(@:.bin=) using $(filter %.sv, $^)"
>     @if [ -z "$(filter %.pcf, $^)" ]; then echo "No PCF file found"; exit 1; fi
>     yosys -p "read_verilog -sv  $(filter %.sv, $^) ; hierarchy -auto-top; synth_ice40 -spram -json $(@:.bin=.json); tee -q -o $(@:.bin=.stats.json) stat -json;"
>     nextpnr-ice40 --up5k --package sg48 --pcf $(filter %.pcf, $^) --json $(@:.bin=.json) --asc $(@:.bin=.asc)
>     icepack $(@:.bin=.asc) $@


%.placed.svg:
>     @echo "Building Placement SVG $@ using $(filter %.sv, $^)"
>     @if [ -z "$(filter %.pcf, $^)" ]; then echo "No PCF file found"; exit 1; fi
>     yosys -p "read_verilog -sv  $(filter %.sv, $^) ; hierarchy -auto-top; synth_ice40 -spram -json $(@:.placed.svg=.json); tee -q -o $(@:.bin=.stats.json) stat -json;"
>     nextpnr-ice40 --up5k --package sg48 --pcf $(filter %.pcf, $^) --json $(@:.placed.svg=.json) --placed-svg $@ --asc $(@:.placed.svg=.asc) --report $(@:.placed.svg=.report.json)

%.routed.svg:
>     @echo "Building Placement SVG $@ using $(filter %.sv, $^)"
>     @if [ -z "$(filter %.pcf, $^)" ]; then echo "No PCF file found"; exit 1; fi
>     yosys -p "read_verilog -sv  $(filter %.sv, $^) ; hierarchy -auto-top; synth_ice40 -spram -json $(@:.routed.svg=.json); tee -q -o $(@:.bin=.stats.json) stat -json;"
>     nextpnr-ice40 --up5k --package sg48 --pcf $(filter %.pcf, $^) --json $(@:.routed.svg=.json) --routed-svg $@ --asc $(@:.routed.svg=.asc) --report $(@:.routed.svg=.report.json)


# >     nextpnr-ice40 --up5k --package sg48 --pcf $(filter %.pcf, $^) --json $(@:.bin=.json) --asc $(@:.bin=.asc)

# >     nextpnr-ice40 --up5k --package sg48 --pcf $(filter %.pcf, $^) --json $(@:.bin=.json) --routed-svg $(@:.bin=.routed.svg) --placed-svg $(@:.bin=.placed.svg) --asc $(@:.bin=.asc)
# >     nextpnr-ice40 --up5k --package sg48 --pcf $(filter %.pcf, $^) --json $(@:.bin=.json) --write $(@:.bin=.nextpnr.json) --asc $(@:.bin=.asc)

# ifneq ($(filter %.debug.json, $^),"")
# # put the .sv in this directory and the resulting debug.json in the output directory
# >    cp $(filter %.debug.json, $^) $(@D)/$*.debug.json
# endif
# # include the manta .sv if one.  TODO

# Will remove any lines between lines that contain <RM> and </RM>
%.rom.txt:
>     sed '/<RM>/,/<\/RM>/d' $^ > $(@:.rom.txt=.temp)
>     riscv64-unknown-elf-as -mlittle-endian $(@:.rom.txt=.temp) -o $(@:.rom.txt=.o)
>     riscv64-unknown-elf-objdump -d $(@:.rom.txt=.o) > $(@:.rom.txt=.dump)
>     sed -n -E  -E "s/^[[:space:]]*[0-9a-fA-F]+:[[:space:]]*([0-9a-fA-F]{8})[[:space:]]+(.*)$$/\1 \/\/ \2/p" $(@:.rom.txt=.dump) > $@
>     rm -f $(@:.rom.txt=.o) $(@:.rom.txt=.dump) $(@:.rom.txt=.temp)
>     $(MAKEFILE_PATH)/SimpleRISC-VInstCheck.sh $@

# >     rm -f $(@:.rom.txt=.o) $(@:.rom.txt=.dump) $(@:.rom.txt=.temp)


# >    @build=$(call check_build,$@,$^)
# >    if [ $$build -eq 1 ]; then
# >       echo "Building $@ using $^"
# >       pcf=$(filter %.pcf,$^);
# >       echo "PCF: $${pcf}"
# >       verilog=$(filter %.sv,$^);
# >       echo "Verilog: $${verilog}"
# >       manta=$(filter %.debug.json,$^);   ## TODO: Manta processing
# >       yosys -p "read_verilog -sv $${verilog} ; hierarchy -auto-top; proc; opt_clean; fsm -expand; memory -nomap; wreduce -memx; opt_clean; synth_ice40 -json $(PRODUCTS_DIR)/$*.json; tee -q -o $(PRODUCTS_DIR)/$*.stats.json stat;"
# >       nextpnr-ice40 --up5k --package sg48 --pcf $${pcf} --json $(PRODUCTS_DIR)/$*.json --asc $(PRODUCTS_DIR)/$*.asc
# >       icepack $(PRODUCTS_DIR)/$*.asc $(IMAGES_DIR)/$@
# >    else
# >       echo "$@ is up to date."
# >    fi;
# >    ## Open programmer



# %.bin: ${IMAGES_DIR}
# >    @build=$(call check_build,$@,$^)
# >    if [ $$build -eq 1 ]; then
# >       echo "Building $@ using $^"
# >       pcf=$(filter %.pcf,$^);
# >       echo "PCF: $${pcf}"
# >       verilog=$(filter %.sv,$^);
# >       echo "Verilog: $${verilog}"
# >       manta=$(filter %.debug.json,$^);   ## TODO: Manta processing
# >       yosys -p "read_verilog -sv $${verilog} ; hierarchy -auto-top; proc; opt_clean; fsm -expand; memory -nomap; wreduce -memx; opt_clean; synth_ice40 -json $(PRODUCTS_DIR)/$*.json; tee -q -o $(PRODUCTS_DIR)/$*.stats.json stat;"
# >       nextpnr-ice40 --up5k --package sg48 --pcf $${pcf} --json $(PRODUCTS_DIR)/$*.json --asc $(PRODUCTS_DIR)/$*.asc
# >       icepack $(PRODUCTS_DIR)/$*.asc $(IMAGES_DIR)/$@
# >    else
# >       echo "$@ is up to date."
# >    fi;
# >    ## Open programmer


# $(PRODUCTS_DIR):
# >	mkdir -p $(PRODUCTS_DIR)

# $(IMAGES_DIR):
# >	mkdir -p $(IMAGES_DIR)

# .PHONY: clean
# clean:
# >	 rm -f $(PRODUCTS_DIR)/*.rtl.json
# >	 rm -f $(PRODUCTS_DIR)/*.vcd
# >	 rm -f $(PRODUCTS_DIR)/*.bin
# >	 rm -f $(PRODUCTS_DIR)/*.json
# >	 rm -f $(PRODUCTS_DIR)/*.asc
