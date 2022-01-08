## Setup 
```bash
% source setup-ece5745.sh
% mkdir -p $HOME/ece5745
% cd $HOME/ece5745
% git clone git@github.com:cornell-ece5745/ece5745-tut5-asic-tools
% cd ece5745-tut5-asic-tools
% TOPDIR=$PWD
```
## Manual Gather Step

#### Running tests, simulators, dumping VCD's and Testbenches (VTB's)

```bash
% mkdir -p $TOPDIR/sim/build
% cd $TOPDIR/sim/build
% pytest ../tut3_pymtl/sort
NEW% pytest ../tut3_pymtl/sort --test-verilog --dump-vtb

% cd $TOPDIR/sim/build
NEW % ../tut3_pymtl/sort/sort-sim --impl rtl-struct --stats --translate --dump-vtb
```

## 4-state RTL simulation and vcd2saif translation

```bash
# Update directory name ?
% mkdir -p $TOPDIR/sim/vcs_build
% cd $TOPDIR/sim/vcs_build
% mkdir -p $TOPDIR/sim/vcs_build/outputs/vcd
% mkdir -p $TOPDIR/sim/vcs_build/outputs/saif
% vcs ../build/SortUnitStructRTL__nbits_8__pickled.v -full64 -debug_pp -sverilog +incdir+../build +lint=all -xprop=tmerge -top SortUnitStructRTL__nbits_8_tb ../build/SortUnitStructRTL__nbits_8_test_basic_tb.v +vcs+dumpvars+outputs/vcd/SortUnitStructRTL__nbits_8_test_basic_vcs.vcd -override_timescale=1ns/1ns -rad +vcs+saif_libcell -lca
% ./simv
% vcs ../build/SortUnitStructRTL__nbits_8__pickled.v -full64 -debug_pp -sverilog +incdir+../build +lint=all -xprop=tmerge -top SortUnitStructRTL__nbits_8_tb ../build/SortUnitStructRTL__nbits_8_sort-rtl-struct-random_tb.v +vcs+dumpvars+outputs/vcd/SortUnitStructRTL__nbits_8_sort-rtl-struct-random_vcs.vcd -override_timescale=1ns/1ns -rad +vcs+saif_libcell -lca
% ./simv


% cd $TOPDIR/sim/vcs_build/outputs
% vcd2saif -input ./vcd/SortUnitStructRTL__nbits_8_sort-rtl-struct-random_vcs.vcd -output ./saif/SortUnitStructRTL__nbits_8_sort-rtl-struct-random.saif
```

## Synthesis

```bash
 % mkdir -p $TOPDIR/asic-manual/synopsys-dc
 % cd $TOPDIR/asic-manual/synopsys-dc
 % dc_shell-xg-t

 dc_shell> alias "dc_shell>" ""
 # setup_session.tcl
 dc_shell> set_app_var target_library "$env(ECE5745_STDCELLS)/stdcells.db"
 dc_shell> set_app_var link_library   "* $env(ECE5745_STDCELLS)/stdcells.db"
 dc_shell> saif_map -start
 # read_design.tcl
 dc_shell> analyze -format sverilog ../../sim/build/SortUnitStructRTL__nbits_8__pickled.v
 dc_shell> elaborate SortUnitStructRTL__nbits_8
 # constraints.tcl
 dc_shell> create_clock -name ideal_clock1 -period 0.3 [get_ports clk]
 dc_shell> set_input_delay -clock ideal_clock1 [expr 0.3*0.05] [all_inputs]
 dc_shell> set_output_delay -clock ideal_clock1 [expr 0.3*0.05] [all_outputs]
 dc_shell> set_max_fanout 20 SortUnitStructRTL__nbits_8
 dc_shell> set_max_transition [expr 0.25*0.3] SortUnitStructRTL__nbits_8
 # make_path_groups.tcl?
 dc_shell> set ports_clock_root [filter_collection \
                       [get_attribute [get_clocks] sources] \
                       object_class==port]
 dc_shell> group_path -name REGOUT \
                      -to   [all_outputs]
 dc_shell> group_path -name REGIN \
                      -from [remove_from_collection [all_inputs] $ports_clock_root]
 dc_shell> group_path -name FEEDTHROUGH \
                      -from [remove_from_collection [all_inputs] $ports_clock_root] \
                      -to   [all_outputs]
 # compile.tcl
 dc_shell> check_design
 dc_shell> compile
 # generate_results.tcl
 dc_shell> saif_map -create_map \
                    -input "../../sim/vcs_build/outputs/saif/SortUnitStructRTL__nbits_8_sort-rtl-struct-random.saif" \
                    -source_instance "SortUnitStructRTL__nbits_8_tb/DUT"
 dc_shell> saif_map -type ptpx -write_map "post-synth.namemap"
 dc_shell> write -format verilog -hierarchy -output post-synth.v
 dc_shell> write -format ddc     -hierarchy -output post-synth.ddc
 dc_shell> write_sdc -nosplit post-synth.sdc; # DIDNT INCLUDE IN MANUAL TUT BC POWER AND INNOVUS MAKE THEIR OWN, NEED TO TEST THIS SINCE WE ADDED INPUT DELAY AND STUFF. My guess is that we'll need to have this command, and modify the power and innovus manual steps to read in this sdc. We should also make sure the sort unit meets timing. 
 # reporting.tcl
 dc_shell> report_timing -nosplit -transition_time -nets -attributes
 dc_shell> report_area -nosplit -hierarchy
 dc_shell> report_power -nosplit -hierarchy
 dc_shell> exit

 % cd $TOPDIR/asic-manual/synopsys-dc
 % more post-synth.v

 % cd $TOPDIR/asic-manual/synopsys-dc
 % design_vision-xg
 design_vision> set_app_var target_library "$env(ECE5745_STDCELLS)/stdcells.db"
 design_vision> set_app_var link_library   "* $env(ECE5745_STDCELLS)/stdcells.db"
```
 (Follow tutorial instructions from here).
 To do on your own, sweep clock period until it meets timing

 % cd $TOPDIR/asic-manual/synopsys-dc
 % dc_shell-xg-t -f init.tcl

 ## Cadence Innovus
```bash
 % mkdir -p $TOPDIR/asic-manual/cadence-innovus
 % cd $TOPDIR/asic-manual/cadence-innovus
```

Create a file called constraints.sdc with:
```
create_clock clk -name ideal_clock -period 0.3
```
Create a file called setup-timing.tcl with: 
```
 create_rc_corner -name typical \
    -cap_table "$env(ECE5745_STDCELLS)/rtk-typical.captable" \
    -T 25

 create_library_set -name libs_typical \
    -timing [list "$env(ECE5745_STDCELLS)/stdcells.lib"]

 create_delay_corner -name delay_default \
    -early_library_set libs_typical \
    -late_library_set libs_typical \
    -rc_corner typical

 create_constraint_mode -name constraints_default \
    -sdc_files [list constraints.sdc]

 create_analysis_view -name analysis_default \
    -constraint_mode constraints_default \
    -delay_corner delay_default

 set_analysis_view \
    -setup [list analysis_default] \
    -hold [list analysis_default]
```
```bash
 % cd $TOPDIR/asic-manual/cadence-innovus
 % innovus -64

 innovus> set init_mmmc_file "setup-timing.tcl"
 innovus> set init_verilog   "../synopsys-dc/post-synth.v"
 innovus> set init_top_cell  "SortUnitStructRTL__nbits_8"
 innovus> set init_lef_file  "$env(ECE5745_STDCELLS)/rtk-tech.lef $env(ECE5745_STDCELLS)/stdcells.lef"
 innovus> set init_gnd_net   "VSS"
 innovus> set init_pwr_net   "VDD"

 innovus> init_design
 innovus> floorPlan -r 1.0 0.70 4.0 4.0 4.0 4.0

 innovus> globalNetConnect VDD -type pgpin -pin VDD -inst * -verbose
 innovus> globalNetConnect VSS -type pgpin -pin VSS -inst * -verbose

 innovus> sroute -nets {VDD VSS}

 innovus> addRing -nets {VDD VSS} -width 0.6 -spacing 0.5 \
            -layer [list top 7 bottom 7 left 6 right 6]

 innovus> addStripe -nets {VSS VDD} -layer 6 -direction vertical \
            -width 0.4 -spacing 0.5 -set_to_set_distance 5 -start 0.5

 innovus> addStripe -nets {VSS VDD} -layer 7 -direction horizontal \
            -width 0.4 -spacing 0.5 -set_to_set_distance 5 -start 0.5

 innovus> place_design

 innovus> ccopt_design

 innovus> routeDesign

 innovus> setFillerMode -corePrefix FILL -core "FILLCELL_X4 FILLCELL_X2 FILLCELL_X1"
 innovus> addFiller

 innovus> verifyConnectivity
 innovus> verify_drc

 innovus> saveNetlist post-par.v
 innovus> extractRC
 innovus> rcOut -rc_corner typical -spef post-par.spef
 innovus> streamOut post-par.gds \
            -merge "$env(ECE5745_STDCELLS)/stdcells.gds" \
            -mapFile "$env(ECE5745_STDCELLS)/rtk-stream-out.map"
 innovus> report_timing
 innovus> report_area
 innovus> report_power -hierarchy all
 innovus> exit

 ## Viewing GDS 

 % cd $TOPDIR/asic-manual/cadence-innovus
 % klayout -l $ECE5745_STDCELLS/klayout.lyp post-par.gds

```

To do on your own:

Change to:
```
 dc_shell> compile -ungroup_all 
 (OR)
 dc_shell> compile_ultra -no_autoungroup
```
Then run cadence innovus commands:
```
 % cd $TOPDIR/asic-manual/cadence-innovus
 % innovus -64 -no_gui -files init.tcl
```

## Synopsys pt power
```bash
 % mkdir -p $TOPDIR/asic-manual/synopsys-pt
 % cd $TOPDIR/asic-manual/synopsys-pt
 % pt_shell

 pt_shell> alias "pt_shell>" ""
 # setup_session.tcl
 pt_shell> set_app_var target_library "$env(ECE5745_STDCELLS)/stdcells.db"
 pt_shell> set_app_var link_library   "* $env(ECE5745_STDCELLS)/stdcells.db"
 pt_shell> set_app_var power_enable_analysis true
 #read_design.tcl
 pt_shell> read_verilog   "../cadence-innovus/post-par.v"
 pt_shell> current_design SortUnitStructRTL__nbits_8
 pt_shell> link_design
 pt_shell> create_clock clk -name ideal_clock1 -period 0.6

 pt_shell> source ../synopsys-dc/post-synth.namemap
 pt_shell> read_saif "../../sim/vcs_build/outputs/saif/SortUnitStructRTL__nbits_8_sort-rtl-struct-random.saif" -strip_path "SortUnitStructRTL__nbits_8_tb/DUT"
 pt_shell> read_parasitics -format spef "../cadence-innovus/post-par.spef" 
 #report_power.tcl
 pt_shell> update_power
 pt_shell> report_power -nosplit
 pt_shell> report_power -nosplit -hierarchy
 pt_shell> exit
```

 To do on your own:

```
 % cd $TOPDIR/sim/build
 % ../tut3_pymtl/sort/sort-sim --impl rtl-struct --input zeros --stats --translate --dump-vtb
 num_cycles          = 105
 num_cycles_per_sort = 1.05
% cd $TOPDIR/sim/vcs_build
% mkdir -p $TOPDIR/sim/vcs_build/outputs/vcd
% mkdir -p $TOPDIR/sim/vcs_build/outputs/saif
% vcs ../build/SortUnitStructRTL__nbits_8__pickled.v -full64 -debug_pp -sverilog +incdir+../build +lint=all -xprop=tmerge -top SortUnitStructRTL__nbits_8_tb ../build/SortUnitStructRTL__nbits_8_sort-rtl-struct-zeros_tb.v +vcs+dumpvars+outputs/vcd/SortUnitStructRTL__nbits_8_sort-rtl-struct-zeros_vcs.vcd -override_timescale=1ns/1ns -rad +vcs+saif_libcell -lca
% ./simv
% cd $TOPDIR/sim/vcs_build/outputs
% vcd2saif -input ./vcd/SortUnitStructRTL__nbits_8_sort-rtl-struct-zeros_vcs.vcd -output ./saif/SortUnitStructRTL__nbits_8_sort-rtl-struct-zeros.saif
``` 

```
 pt_shell> read_saif "../../sim/vcs_build/outputs/saif/SortUnitStructRTL__nbits_8_sort-rtl-struct-zeros.saif" -strip_path "SortUnitStructRTL__nbits_8_tb/DUT"
```

Using Verilog RTL Models:

```bash
 % cd $TOPDIR/sim/build
 % rm -rf *
 % pytest ../tut4_verilog/sort --dump-vtb; #Do we need --test-verilog?
```

```
 % cd $TOPDIR/sim/build
 % ../tut4_verilog/sort/sort-sim --impl rtl-struct --stats --translate --dump-vtb
 ```